---
title: Optimizing vector tile size
subtitle: A guided tour
tags: [gis, maps, vectortiles]

---

In my [Observable Vector Tile Dissector](https://observablehq.com/@henrythasler/mapbox-vector-tile-dissector) I evaluate different vector tile provider regarding the tilesize they deliver. The limits I chose are ~~purely random~~ based on long experience... But what would one do to actually reduce the tile size. I will try to optimize [my own vector tiles](https://cyclemap.link) and explain the steps involved.

I assume that all spatial data is based on [OpenStreetMap](https://www.openstreetmap.org) and available via a [postgis](https://postgis.net/) database in web mercator ([EPSG:3857](https://epsg.io/3857)) projection.

## TL;DR

By applying different methods, we can reduce the size of a given vector tile to just **14% of the original size**. 

![](/img/blog/sizechart.png)

The lossless methods described below are:

- [Remove unused data](#identifier)
- [Reduce tile buffer](#tile-buffer)
- [Merge features](#merge)
- [Compression](#gzip)

## Status Quo

Let's pick two tiles at different zoom levels and see how big they are:

Tile | `14/8717/5683.mvt` | `10/544/355.mvt`
---|---|---
Size | `64 KiB (64984 Bytes)` | `352 KiB (359820 Bytes)`
Raw | ![](/img/blog/Selection_171.png) | ![](/img/blog/Selection_173.png)
Rendered | ![](/img/blog/Selection_172.png) | ![](/img/blog/Selection_174.png)

I already use zoom-dependent tables to create the vector tiles and also generalize (reduce spatial complexity) the features in lower zoom levels. So what can we do to reduce the size?

## Identifier 

Let's grab the [Vector Tile Dissector](https://observablehq.com/@henrythasler/mapbox-vector-tile-dissector) and have a look what the contents of the tiles are right now:

![](/img/blog/Selection_175.png)

We see lots of IDs that are stored as numbers. The amount is several thousand for the z10-tile. The current style is NOT using these IDs so we can remove **all** references to osm_id/id from the layer schema:

![](/img/blog/Selection_177.png)

Let's check the result:

![](/img/blog/Selection_178.png)

Almost all `Number` properties are gone and the size has been reduced significantly:

Tile | `14/8717/5683.mvt` | `10/544/355.mvt`
---|---|---
Orignal | `64 KiB (64984 Bytes)` | `352 KiB (359820 Bytes)`
IDs removed | `51 KiB (51966 Bytes)` | `278 KiB (283677 Bytes)`
relative size | **80%** | **79%**

Great, just by removing this unused property we have saved around 20%! 

In some cases it might make sense to attach an ID to a feature if you need that reference to load additional information later on (e.g. `onClick()` event). Azure Maps is doing that for example:  

![](/img/blog/Selection_183.png)

Anyway, let's see if we can squeeze the tiles some more.

## Tile Buffer

I a recent post, I wrote about [tile buffers](https://blog.cyclemap.link/2020-01-25-tilebuffer/). Currently, the vector tiles have a buffer of 256 coordinate units. Let's try to reduce the buffer to 64 units:

Tile | `14/8717/5683.mvt` | `10/544/355.mvt`
---|---|---
Raw | ![](/img/blog/Selection_179.png) | ![](/img/blog/Selection_180.png)
Rendered | ![](/img/blog/Selection_182.png) | ![](/img/blog/Selection_181.png)

There are no visual artefacts but the size reduction is marginal. That was expected.

Tile | `14/8717/5683.mvt` | `10/544/355.mvt`
---|---|---
previous step | `51 KiB (51966 Bytes)` | `278 KiB (283677 Bytes)`
64 coordinate unit buffer | `50 KiB (51114 Bytes)` | `276 KiB (282250 Bytes)`
relative size | **98%** | **99%**

## Merge

Let's look again at some other statistics of our z10-tile:

![](/img/blog/Selection_184.png)

Two things can be noticed:

1. The `roads` and `railway` layer contain 87% of all features.
2. More than 95% of these features contain just 2 points.

Let's pick a random road and check the database what it looks like:

```SQL
SELECT 
	ST_NPoints(geometry) as points, 
	geometry 
FROM import.roads_gen10 
where (geometry && ST_Transform(ST_MakeEnvelope(11.2500, 47.9899, 11.6016, 48.2247, 4326), 3857)) AND ref = 'St 2345' order by points DESC;
```

![](/img/blog/Selection_185.png)

This road encoded as vector tile in postgis:

```SQL
SELECT
	length(ST_AsMVT(q, 'roads', 4096, 'geom')) as length
FROM
	(
	SELECT
		ST_AsMVTGeom(
			geometry,
			ST_Transform(ST_MakeEnvelope(11.2500, 47.9899, 11.6016, 48.2247, 4326), 3857), 
			4096, 
			0) AS geom
	FROM
		import.roads_gen10
	WHERE
		(geometry && ST_Transform(ST_MakeEnvelope(11.2500, 47.9899, 11.6016, 48.2247, 4326), 3857))
		AND ref = 'St 2345' ) AS q;
```
results in:
```
Name  |Value|
------|-----|
length|1639 |
```

So a single (continuous) road in this tile is made from 112 individual LineStrings where most (104) are made from only 2 points. Encoded as a vector tile, this road uses 1639 bytes. 

What happens if we `ST_LineMerge()` the individual 2-point `LINESTRING`s into one big `MULTILINESTRING`?

```SQL
SELECT
	length(ST_AsMVT(q, 'roads', 4096, 'geom')) AS length
FROM
	(
	SELECT
		ST_AsMVTGeom(
			ST_LineMerge( ST_Collect(geometry) ),
			ST_Transform(ST_MakeEnvelope(11.2500, 47.9899, 11.6016, 48.2247, 4326), 3857), 
			4096, 
			0) AS geom
	FROM
		import.roads_gen10
	WHERE
		(geometry && ST_Transform(ST_MakeEnvelope(11.2500, 47.9899, 11.6016, 48.2247, 4326), 3857))
		AND ref = 'St 2345' ) as q;
```
results in:
```
Name  |Value|
------|-----|
length|398  |
```

So simply by merging the `LINESTRING`s we can reduce the size of this road by 75%. The main reason behind this is, that in a vector tile for each new `LINESTRING` the cursor has to be moved to the start of the line first  before the line can be drawn. This can be omitted in a `LINESTRING` if the start point of the new line is the same as the end of the previous line. Maybe I'll write another post to explain that in detail. Until then, please refer to the [Vector Tile Specification](https://github.com/mapbox/vector-tile-spec/tree/master/2.1).

Anyway, let's modify our vector tile provider ([cloud-tileserver](https://github.com/henrythasler/cloud-tileserver)) to see what that is worth in a real-world scenario:

![](/img/blog/Selection_190.png)

Yes, it works! We now have fewer but larger Features. Exactly what we want. We can merge some other line features (e.g. `waterways`) as well.

But this method is not limited to lines. We can also `ST_Union()` all the building-polygons within a tile into one big multipolygon. The resulting size reduction is outstanding, especially with low zoom levels:

Tile | `14/8717/5683.mvt` | `10/544/355.mvt`
---|---|---
previous step | `50 KiB (51114 Bytes)` | `276 KiB (282250 Bytes)`
Merge Features | `31 KiB (31518 Bytes)` | `73 KiB (74081 Bytes)`
relative size | **62%** | **26%**

To use this feature, you need to specify a new layer-property (`geom_query`) and also a `GROUP BY` statement for all keys:

![](/img/blog/Selection_191.png)

Please keep in mind, that these merging-operations (especially `ST_Union()`) may be very expensive (computational-wise) for the database-server and may slow down un-cached access.

## GZIP

After reducing the encoded size of the vector tiles, we can further reduce the size by applying a compression algorithm. GZIP is [supported by all available browsers](https://caniuse.com/#search=gzip), so no need to worry about compatibility.

![](/img/blog/Selection_192.png)

Tile | `14/8717/5683.mvt` | `10/544/355.mvt`
---|---|---
previous step | `31 KiB (31518 Bytes)` | `73 KiB (74081 Bytes)`
gzipped | `22 KiB (22382 Bytes)` | `49 KiB (50061 Bytes)`
relative size | **71%** | **68%**

This will not only help to save storage costs but also reduce the loading time for the end-user.

## Summary

Tile | `14/8717/5683.mvt` | `10/544/355.mvt`
---|---|---
Orignal | `64 KiB (64984 Bytes)` | `352 KiB (359820 Bytes)`
IDs removed | `51 KiB (51966 Bytes)` | `278 KiB (283677 Bytes)`
64 coordinate unit buffer | `50 KiB (51114 Bytes)` | `276 KiB (282250 Bytes)`
Merge Features | `31 KiB (31518 Bytes)` | `73 KiB (74081 Bytes)`
relative size | **48%** | **21%**
gzipped | `22 KiB (22382 Bytes)` | `49 KiB (50061 Bytes)`
relative size | **34%** | **14%**

The rendered tiles show some minor differences, mostly due to some features being ordered differently:

Tile | `14/8717/5683.mvt` | `10/544/355.mvt`
---|---|---
Original | ![](/img/blog/Selection_172.png) 64 KiB | ![](/img/blog/Selection_174.png) 352 KiB
Optimized | ![](/img/blog/Selection_195.png) 31 KiB | ![](/img/blog/Selection_196.png) 73 KiB

By analyzing the existing vector tiles and spatial data in the database, we were able to reduce the size of a vector tile to just **21%** (14% incl. compression) of the original size without reducing the visual experience. The methods shown should be applicable to almost any vector tile server available.

I'm looking forward for more ideas on how to reduce the size even more.

## References

- [Vector tile specification](https://docs.mapbox.com/vector-tiles/specification/)
- [Mapbox Vector Tile (MVT) Comparison and Dissector](https://observablehq.com/@henrythasler/mapbox-vector-tile-dissector)
- [cloud-tileserver](https://github.com/henrythasler/cloud-tileserver)
