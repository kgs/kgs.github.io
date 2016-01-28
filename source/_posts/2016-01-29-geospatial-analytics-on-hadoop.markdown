---
layout: post
title: "Geospatial analytics on Hadoop"
date: 2016-01-29 12:32:01 +0100
comments: true
categories: spark hadoop
---

Few months ago I was working on a project with a lot of geospatial data. Data was stored in HDFS, easily accesible through Hive. One of the tasks was to analyze this data and first step was to join two datasets on columns which were geographical coordinates. I wanted some easy and efficient solution. But here is the problem - there is very little support for this kind of operations in Hadoop world.

<!-- more -->

### Problem

Ok, so what's the problem actually? Let's say we have two datasets (represented as Hive tables). First one is very large set of geo-tagged tweets. Second one is city/place geographic boundaries. We want to match them - for every tweet we want to know it's location name.

Here are the tables (coordinates are given in simple [WKT format][wkt]):

```
+-----------+------------------+---------------------------------------------+
| tweets.id |  tweets.content  |  tweets.location_wkt                        |
+-----------+------------------+---------------------------------------------+
| 11        | Hi there!        | POINT(21.08448028564453 52.245122234020435) |
| 42        | Wow, great trip! | POINT(22.928466796875 54.12185996058409)    |
| 128       | Happy :)         | POINT(13.833160400390625 46.38046653471246) |
...
```

```
+-----------+-----------------+-----------------------------------------------------+
| places.id |  places.name    |  places.boundaries_wkt                              |
+-----------+-----------------+-----------------------------------------------------+
| 65        | Warsaw          | POLYGON((20.76965332 52.356842,21.25305 52.3567 ... |
| 88        | Suwałki         | POLYGON((22.890014 54.12829,22.96142 54.12829 ...   |
| 89        | Triglav         | POLYGON((13.820114 46.383597,13.846206 46.38359 ... |
...
```

So how to do it in Hive or Spark? Without any additional libraries or tricks we can simply do **cross join**, which means: compare every element from first dataset with element from the second one and then decide (using some user defined function) if there is a match. 

But this solution has two major drawbacks:

 * it is super slow
 * we need to write some code (UDFs) which will operate on coordinates (checks if point is in polygon, etc.)

For sure there must be a better way!

### What are the options?

There are few libraries which could help us with this task, but some of them give us only nice API (GIS Tools, Magellan) where other can do spatial joins effectively (SpatialSpark). Let's look at them one by one!

### Esri GIS Tools for Hadoop

People from Esri (international company which provides Geographic Information System software) developed and open sourced [GIS Tools for Hadoop][gistools]. This toolkit contains few elements, but two most important ones are:

 * [Esri Geometry API for JAVA][geomapi] - it includes geometry objects, spatial operations and indexing. It can be used in standalone programs or MapReduce/Spark jobs.
 * [Spatial Framework for Hadoop][spatialhive] - this library includes user defined functions (UDF) that extend Hive to make spatial operations more user-friendly, internally it uses Esri Geometry API.

To install this toolkit you have to simply add jars to Hive classpath and then register needed UDFs. You can find more detailed tutorial [here][earthquake].

Finally you will be able to run Hive query like this:

```sql
SELECT * FROM places, tweets
    WHERE ST_Intersects(
               ST_GeomFromText(places.boundaries_wkt),
               ST_GeomFromText(tweets.location_wkt)
          );
```

If you know [Postgis][postgis] (GIS extension for PostgreSQL) this will look very familiar to you, because syntax is similar. Unofortunately these kind of queries are very inefficient in Hive. Hive will do cross join and it means that for big datasets computations will last for unacceptable amount of time.

#### Spatial binning

There is small trick which can help a bit with efficiency problem when doing spatial joins. It's called spatial binning. The idea is to divide our space with points and polygons to numbered rectangular blocks. Then, for every object (like point or polygon) we assign corresponding block number to it.

Here is (hopefully) helpful image:

{% img center /images/binning.png %}

In the above example, space was divided into 8 blocks, there are some empty blocks and some with many points. For example there are 5 points which will get number 4 as their **BIN ID**.

Going back to our example with tweets (represented as points) and places (represented as polygons) we can assign BIN IDs to both of them and then join them block by block, calling UDFs only for objects with the same BIN ID. It will be more efficient because we will only do cross joins for significantly smaller sets (one block), but many of them (as many as total number of blocks).

Of course, there are some corner cases (like borders of blocks), but general idea is as explained. If you want to read more about this technique, please visit [Esri Wiki][binning].

### Magellan

Second solution I'd like to show you is based on Apache Spark - more powerful (but also a bit more complicated) tool than Apache Hive.

Magellan is open source library for geospatial analytics that uses Spark as underlying engine. Hortonworks published blog post about it [here][magellanblog] and as far as I understand this library was created by one of the company's engineers.

It is in very early stage of development and as of this date it gives us only nice API and unfortunately not so efficient algorithms for spatial joins. 

Here is sample code in Spark (using Scala) to do spatial join using *intersects* predicate:

```scala
// points and polygons are DataFrames of types magellan.{Point, Polygon}
points.join(polygons).where($"point" intersects $"polygon").show()
```

It is definetely library to watch, but as for now it's not so useful in my opinion, mainly because it's lacking features. If you want to know more, please visit Magellan [github page][magellan].

### SpatialSpark

Third solution and also my favourite one (maybe because I contributed to it a bit ;)) is [SpatialSpark][spatialspark]. It's another library that is using Apache Spark as underlying engine. For low-level spatial functions and data structures (like indexes) it is using great and well tested [JTS][jts] library.

It's selling feature is that it can do spatial joins efficiently. It supports two kind of joins:

 * **broadcast spatial join** - it's designed for joining big dataset with smaller one efficiently. Smaller data set is converted to index (R-tree) and kept in memory. Algorithm simply iterates (in distributed way) over big dataset and queries index from the other set efficiently.
 * **partitioned spatial join** - it's designed for joining two big datasets and uses similiar idea to binning, but it's more complicated and more efficient. Sets are divided into small pieces (you can choose what algorithm could be responsible for this operation - there are few implemented to make splits as equal as possible depending on data characteristics) and then each small piece is processed individually (using R-trees).

Here is sample Spark code snippet to do broadcast spatial join for our case with tweets and places:

```scala
// create RDD with pairs (id, location_geometry) for tweets
val leftGeometryById : RDD[(Long, Geometry)] =
	tweets.map(r => (r.id.toLong, new WKTReader().read(r.location_wkt)))

// right geometry (places) has to be relatively small for broadcast join
val rightGeometryById : RDD[(Long, Geometry)] =
	places.map(r => (r.id.toLong, new WKTReader().read(r.boundaries_wkt)))

// we get matching ids from tweets and places
val matchedIdPairs : RDD[(Long, Long)] =
	BroadcastSpatialJoin(sparkContext, leftGeometryById, rightGeometryById,
	                     SpatialOperator.Intersects, 0.0)
```

Unfortunately there are also drawbacks. API is not so clean and easy to use. You have to use classes as shown in example above or use command line tools that expect data in exactly one format (more details on [github page][spatialsparkgithub]). Even bigger problem is that development of SpatialSpark is not so active. Hopefully it will change in future.

### Other options

If you can and want to keep data in some other systems than Hadoop there are few possibilities to do spatial joins. Of course not all of them have the same set of features, but all of them implement some kind of geospatial search that could be useful when dealing with geographic data. 

Here are the links:

 * [Cassandra with Lucene index][stratio] - you can keep data in Cassandra and use secondary index that integrates Lucene features (geospatial search is one of many)
 * [Elasticsearch (with Geohashes)][elastic] - geohashes are a way of encoding latitue and longitude to string, you can keep and query them with Elasticsearch 
 * [GeoMesa][geomesa] - it's whole geospatial distributed database built on top of Apache Accumulo
 * [GeoWave][geowave] - very similar to GeoMesa, but a bit newer

### Summary

As you can probably see now, there is no big choice in terms of spatial joins when we have our data in Hadoop. If you want to do things efficiently then [SpatialSpark][spatialspark] is the only option IMHO. If you want something easier to use then [Esri GIS Tools for Hadoop][gistools] is the way to go, but unfortunately this only makes sense for really small datasets.

That's all! Hopefully you've enjoyed this post. Feel free to comment below if you have any questions or suggestions.

[gistools]: https://esri.github.io/gis-tools-for-hadoop/
[geomapi]: https://github.com/Esri/geometry-api-java
[spatialhive]: https://github.com/Esri/spatial-framework-for-hadoop
[binning]: https://github.com/Esri/gis-tools-for-hadoop/wiki/Aggregating-CSV-Data-%28Spatial-Binning%29
[wkt]: https://en.wikipedia.org/wiki/Well-known_text
[earthquake]: https://github.com/Esri/gis-tools-for-hadoop/tree/master/samples/point-in-polygon-aggregation-hive
[postgis]: http://postgis.net/
[magellanblog]: http://hortonworks.com/blog/magellan-geospatial-analytics-in-spark/
[magellan]: https://github.com/harsha2010/magellan
[spatialspark]: http://simin.me/projects/spatialspark/
[jts]: http://tsusiatsoftware.net/jts/main.html
[spatialsparkgithub]: https://github.com/syoummer/SpatialSpark
[stratio]: https://github.com/Stratio/cassandra-lucene-index
[elastic]: https://www.elastic.co/guide/en/elasticsearch/guide/current/geohashes.html
[geomesa]: http://www.geomesa.org/
[geowave]: https://ngageoint.github.io/geowave/