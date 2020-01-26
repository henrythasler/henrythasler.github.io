---
title: Optimizing vector tile size
subtitle: A comprehensive guide.
tags: [gis, maps, vectortiles]

---

In my [Observable Vector Tile Dissector](https://observablehq.com/@henrythasler/mapbox-vector-tile-dissector) I evaluate different vector tile provider regarding the tilesize they deliver. The limits I chose are ~~purely random~~ based on long experience... But what would one do to actually reduce the tile size. I will try to optimize [my own vector tiles](https://cyclemap.link) and explain the steps involved.

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
**Saved Overall** | **`-20%`** | **`-21%`**

Great, just by removing this unused property we have saved 20%! 

In some cases it might make sense to attach an ID to a feature if you need that reference to load additional information later on (e.g. `onClick()` event). Azure Maps is doing that for example:  

![](/img/blog/Selection_183.png)

Anyway, let's see if we can squeeze the tiles some more.

## Tile Buffer

I a recent post, I wrote about [tile buffers](https://blog.cyclemap.link/2020-01-25-tilebuffer/). Currently, the vector tiles have a buffer of 256 coordinate units. Let's try to reduce the buffer to 64 units for low zoom levels:

Tile | `14/8717/5683.mvt` | `10/544/355.mvt`
---|---|---
Raw | ![](/img/blog/Selection_179.png) | ![](/img/blog/Selection_180.png)
Rendered | ![](/img/blog/Selection_182.png) | ![](/img/blog/Selection_181.png)

There are no visual artefacts but the size reduction is marginal. That was expected.

Tile | `14/8717/5683.mvt` | `10/544/355.mvt`
---|---|---
Orignal | `64 KiB (64984 Bytes)` | `352 KiB (359820 Bytes)`
IDs removed | `51 KiB (51966 Bytes)` | `278 KiB (283677 Bytes)`
64 coordinate unit buffer | `50 KiB (51114 Bytes)` | `276 KiB (282250 Bytes)`
**Saved Overall** | **`-21%`** | **`-22%`**

## Merge

Let's look again at some other statistics of our tile:

![](/img/blog/Selection_184.png)

Two things can be noticed:

1. The `roads` and `railway` layer contain 87% of all features.
2. More than 95% of these features contain just 2 points.

Let's pick a random road and check the database what it looks like:

```SQL
select ST_NPoints(geometry), geometry from import.roads_gen10 where ref = 'St 2345';
```

![](/img/blog/Selection_185.png)

A single continuous road in this tile is made from 112 individual LineStrings where most (104) are made from only 2 points.

This road encoded as vector tile:

```SQL
select
	length(st_asmvt(q, 'roads', 4096, 'geom')) as l
from
	(
	select
		st_asmvtgeom(geometry, st_transform(st_makeenvelope(11.2500, 47.9899, 11.6016, 48.2247, 4326), 3857), 64) as geom
	from
		import.roads_gen10
	where
		(geometry && st_transform(st_makeenvelope(11.2500, 47.9899, 11.6016, 48.2247, 4326), 3857)) and ref = 'St 2345'
    ) as q;
```

is 447 Bytes.



## GZIP

## References

- [Vector tile specification](https://docs.mapbox.com/vector-tiles/specification/)
- [Mapbox Vector Tile (MVT) Comparison and Dissector](https://observablehq.com/@henrythasler/mapbox-vector-tile-dissector)
