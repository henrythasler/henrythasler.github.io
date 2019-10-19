
As [vector maps](https://en.wikipedia.org/wiki/Vector_tiles) have been really successful (see [maptiler example](https://www.maptiler.com/maps/#topo//vector/13.55/10.99274/47.45656)) in the last years I want to upgrade my own map and [toolchain](https://github.com/henrythasler/TileGenerator) to fancy new vector tiles technology. This series of blog posts serves mainly to sort out my thoughts and hopefully make life easier for the next guy to try this.

This is what the map looks like at the moment:

![rastermap](/img/cyclemap-raster-current.png)

As the process of presenting a vector tile map to an end-user requires a whole set of tools (aka Toolchain), I will try to cover all parts of the design- and implementation-process.

All project-files will be stored on github.com/henrythasler/vectortiles. Create an issue there, if you have any questions, hints or whatevers.

## Goals

The following goals MUST be reached by the new toolchain/setup:

- A free source for spatial data (e.g. openstreetmap)
- Means of pre-processing and storing the spatial data (e.g. in a database)
- Content and quality of the resulting map is roughly the same as it is with the current toolchain.
- Use open source software whenever possible
- Dockerize the whole toolchain for portability
- support for high-DPI screens

## Stretch Goals

The following items would be nice-to-have

- Existing style definitions (CartoCSS, Mapnik XML) can be reused/converted
- Map design process is supported by some GUI/Editor
- Toolchain can also generate raster tiles for offline use with low-end devices (smartphone).

## Sifting the web for useful information

During my initial research I found these pages on various related topics helpful (random order):

- How to make mvt with PostGIS by Parlons Tech
- Vector tiles, PostGIS and OpenLayers by Giovanni Allegri
- awesome-vector-tiles by mapbox
- OpenMapTiles.org
- Using the new MVT function in PostGIS by Chris Whong
- Vector Tiles - Introduction & Usage with QGIS by Pirmin and Kalberer
- MVT generation: Mapnik vs PostGIS by Rafa de la Torre

## Conclusion

We have set some preliminary goals for this whole project. Some my be revised as the project proceeds. The next post will present some results of the initial research regarding tools and methods.
