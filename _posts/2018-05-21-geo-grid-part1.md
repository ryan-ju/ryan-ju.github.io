---
layout: post
title:  "A Fast Geogrid System Part 1: The Design"
date:   2018-05-21 18:42:21
categories: jekyll update
mathjax: true
key: geo_spatial
comment: true
---
## What is a geo-grid system?

You must be familiar with the words "longitude" and "latitude".  They are used to identify a point on the Eath's surface.  They represent the angle (in degrees) between the point and the Earth's center.

Usually they are shown as float point numbers.  To give an rough idea, 1 degree of latitude is roughly 110km (longitude is more complex, more about this later).

A geo-grid system is a way to group the points on the Earth surface into "cells".  The cells can come in different shapes, but usually in squares and hexgons.

## Why we need a geo-grid system?

A geo-grid is usually used to speed up calculation and reduce problems' dimension.

The most important application is to be used as a geo index.  Imagine you have millions of lat/lon pairs, and you want to get all points in a defined area (like all shops within London's zone 1).  This requires huge computation, potentially scanning through all points in the DB.

Another example is if you want to compute and cache distance data between two points.  If you have 1 million ($$10^6$$) points, then you'll potentially have to cache $$10^{12}$$ records.  Even then, when you query with new points, they won't be in the cache.

Finally, if you want to build a search tree (like a b-tree) out of lat/lon, you'll find it difficult to make them geo aware (i.e., points close together physically are also close in the b-tree hierarchy).

## What solutions are out there?

One popular solution is to use a geo DB that has spatial arithmetic support.  An example is [PostGIS](https://postgis.net/), which follows the [OGC](https://en.wikipedia.org/wiki/Open_Geospatial_Consortium)'s spatial data standards.

Generally, spartial index comes in three types: [geohash](https://en.wikipedia.org/wiki/Geohash), [R-tree](https://en.wikipedia.org/wiki/R-tree) and [space-filling curves](https://en.wikipedia.org/wiki/Space-filling_curve).

Here's a comparison of some of the popular geo DB and their index types:

| DB | Geohash | R-tree | Space-filling Curve |
|--|----|----|----------|
| [PostGIS](https://postgis.net/) | No | Yes | No |
| [MySQL/Maria](https://dev.mysql.com/doc/refman/8.0/en/spatial-types.html) | No | Yes | No |
| [AWS Aurora MySQL](https://aws.amazon.com/blogs/database/amazon-aurora-under-the-hood-indexing-geospatial-data-using-z-order-curves/) | No | Yes | [Z-curve](https://en.wikipedia.org/wiki/Z-order_curve) |
| [Geomesa](http://www.geomesa.org/documentation/index.html) | No | No | Z-curve |
| [Geowave](https://locationtech.github.io/geowave/devguide.html) | No | No | [Hilbert-curve](https://en.wikipedia.org/wiki/Hilbert_curve)|
| [Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/geo-queries.html) | Yes (with [quadtree](https://en.wikipedia.org/wiki/Quadtree) option) | No | No |
| Neo4j+[spatial plugin](http://neo4j-contrib.github.io/spatial/0.24-neo4j-3.1/index.html) | Yes | Yes | Yes |

Geo-DBs deal with the heavy lifting of geo indexes, so the user doesn't need to revent all the wheels.

However, setting up a geo-DB is not a light task, and you might even need a team managing it if used in production.  You will also need the DB's binding library to use it.  Another concern is that your system will be confined to the computing power of the DB.  If you didn't choose a performant DB in the beginning, scaling might be a problem (not to mention migration across DBs).

Also, geohash, R-tree and Z-curve suffer from a serious problem when computing distances, which is discussed below.

## The Earth is round!

This is a fact (please don't dispute).  But this causes a serious issue when mapping the Earth's surface to 2-dimension.  You simply cannot preserve the shapes!

You can't simply use Pythagoras of the lat/lon to calculate distances, because the distance is different depending on where you're (each circle is the same size in reality):
![](https://upload.wikimedia.org/wikipedia/commons/8/87/Tissot_mercator.png)

The most popular way to project the Earth surface accurately is the [UTM](https://en.wikipedia.org/wiki/Universal_Transverse_Mercator_coordinate_system):
![](http://www.dmap.co.uk/utmworld.gif)

The Earth is divided into different regions, each one has different parameters for projecting a curved surface into a flat one.  

For example, one degree longitude at the equater is about 111km, but only 70km at the UK's latitude:

![](http://www.longitudestore.com/images/size-of-latitude-degrees.jpg)

## Polygon calculation is slow.
Even using a Z-curve, the cost of geometry arithmetics is high, especially when using polygons with many edges.

Also the time to communicate with a DB can add up significantly.

## Solution 
A geo-grid library with UTM transformations.

<img src="{{ site.url }}/assets/geo-grid/grid001.png" width="500" />

### Features
* Uniform cells with no distortion
* Accurate distance calculation
* Cell IDs with geo-hash support (neighbouring cells have common prefix).  This allows a search tree to be built using Neo4j etc.
* Shape file to cell IDs translation
* Defining polygons by boundary cells.  The cells can be stored in any DB and read by the library to reconstruct the polygon in memory.
* Fast search and membership tests
* No external dependencies.  Just a pure library.

## Next
In the next post I'll show some of the technical details of implementing the library.


