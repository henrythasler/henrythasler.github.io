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

## Remove identifier 

Let's grab the [Vector Tile Dissector](https://observablehq.com/@henrythasler/mapbox-vector-tile-dissector) and have a look what the content of the tiles is right now:

![](/img/blog/Selection_175.png)

We see lots of IDs that are stored as numbers. The amount is several thousand for the z10-tile. The current style is NOT using these IDs so we can remove **all** references to osm_id/id from the layer schema:

![](/img/blog/Selection_177.png)

Let's check the result:

![](/img/blog/Selection_178.png)

Almost all `Number` properties are gone and the size has been reduced significantly.

Tile | `14/8717/5683.mvt` | `10/544/355.mvt`
---|---|---
Orignal | `64 KiB (64984 Bytes)` | `352 KiB (359820 Bytes)`
IDs removed | `51 KiB (51966 Bytes)` | `278 KiB (283677 Bytes)`
**Saved Overall** | **`-20%`** | **`-21%`**

Great, just by removing unused properties we have saved 20%! Let's see if we can squeeze the tiles some more.

## Tile Buffer

I a recent post, I wrote about [tile buffers](https://blog.cyclemap.link/2020-01-25-tilebuffer/). Currently, the vector tiles have a buffer of 256 coordinate units. Let's try to reduce the buffer to 64 units for low zoom levels:

Tile | `14/8717/5683.mvt` | `10/544/355.mvt`
---|---|---
Raw | ![](/img/blog/Selection_179.png) | ![](/img/blog/Selection_180.png)
Rendered | ![](/img/blog/Selection_182.png) | ![](/img/blog/Selection_181.png)

There are no visual artefacts but the reduction is marginal. That was expected.

Tile | `14/8717/5683.mvt` | `10/544/355.mvt`
---|---|---
Orignal | `64 KiB (64984 Bytes)` | `352 KiB (359820 Bytes)`
IDs removed | `51 KiB (51966 Bytes)` | `278 KiB (283677 Bytes)`
64 coordinate unit buffer | `50 KiB (51114 Bytes)` | `276 KiB (282250 Bytes)`
**Saved Overall** | **`-21%`** | **`-22%`**

## GZIP

## References

- [Vector tile specification](https://docs.mapbox.com/vector-tiles/specification/)
- [Mapbox Vector Tile (MVT) Comparison and Dissector](https://observablehq.com/@henrythasler/mapbox-vector-tile-dissector)
