---
title: Buffer in Vectortiles
subtitle: Why invisible features are important.
tags: [gis, maps, vectortiles]

---

## Introduction

I was recently asked by [PetersonGIS](https://petersongis.com/), why I recommend to dynamically adjusting the buffer size around vectortiles with the zoom level. I will try to explain that in this post.

## Rendering Vectortiles

First we have to understand how exactly vectortiles are rendered and displayed.

Similar to raster tiles, vector tiles are stitched into one big image within your browser from smaller images (tiling). The only difference is that the actual rastering (converting data to an image) of each vector tile is done in your browser (client-side) instead by a server.

First, each vector tile is rendered individually onto a canvas that is larger that the actual tile. This canvas is then clipped to the actual boundingbox of the tile and stiched together with all other canvasses. 

![](/img/blog/Selection_150.png)

Usually, vector tiles are created with a buffer. That means the vector tile will contain data that is outside the visible area. So what will happen I we remove that buffer?

## Clipping

Lets have a look at a screenshot of a vector tile map:

![](/img/blog/Selection_145.png)

Looks pretty normal but if you look closely, you will notice several spots, where the roads look somehow clipped/distorted:

![](/img/blog/Selection_146.png)

These pictures were taken from a map rendered from vector tiles that were created with NO buffer at all around the actual tile. The clipping occurs exactly on the edges of the tiles:

![](/img/blog/Selection_147.png)

The effects becomes even more noticeable as we zoom into another area and increase the with of the lines (zoom level 17):

![](/img/blog/Selection_149.png)

The reason for these clipping artefacts is, that 1-dimensional features of a vector tile (like roads) are actually rendered with a `line-width` thus being drawn as "2D shapes". This is an x-ray-view of the lower tile being rendered before clipping:

![](/img/blog/Selection_151.png)

The motorway features are rendered as thick lines but they end right at the edge of the tile leading to a gap in the line after assembly with it's neighbor tile. 

This can be observed with feature that are rendered with very thick lines **and** that have a sharp angle between them an the tile edge.

But there is a simple solution!

## Buffer

## Vector tiles provider



## References

- [Vector tile specification](https://docs.mapbox.com/vector-tiles/specification/)
- [Mapbox Vector Tile (MVT) Comparison and Dissector](https://observablehq.com/@henrythasler/mapbox-vector-tile-dissector)