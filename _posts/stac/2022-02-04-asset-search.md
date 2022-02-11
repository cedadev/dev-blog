---
layout: post
title:  "Asset Specification and Asset Search"
author: Richard Smith
date:   2022-02-04 17:00:00
tags: [stac, search]
---

The STAC specification nests `Assets` (downloadable objects) within an `Item` (group of related Assets).
In the CEDA archive, an Item could have 1000s of assets. How do we represent and retrieve Items with large assets counts?
Enabling paginated Asset lists, Item sub-setting and cross-Item Asset search.

## Index

* [Background](#background)
* [Proposal](#proposal)
* [Outcome](#outcome)

## Background

Assets are nested objects as part of the [item-spec](https://github.com/radiantearth/stac-spec/blob/master/item-spec/item-spec.md). This provides easy access to the related 
assets in the context of their item. We interpret an item to be a collection of assets with 
the same set of properties which are intended to be consumed together. For example:
- A satellite scene
- An aerial image
- Research flight
- A single variable generated a single realisation of a model run
- A time-series product (could be processed satellite data, climate model run, etc.)

Assets for an Item consist of metadata, (thumbnails, ISO19115 records) and data files.

The current approach, nesting these Assets in the Item response works well when:
- The number of Assets is low
- The time-range specified is small

Conversely, when the Asset count is high (time-series) and users might want to select a subset, 
the cost of returning the Item object is high. Extreme examples include:

ECMWF-ERA Datasets 2.5 Degree gridded product is comprised of 133,000 files:
- decadal `(23,000)` /yearly `(2,920)`/ monthly `(248)`/ daily `(8)`

CCI Sea Level L4:
- Full Dataset `(553)`/ Decadal `(120)`/ Yearly `(12)`

An argument could be made to split this into multiple Items using perhaps, decadal/yearly/monthly/daily, 
whichever results in a reasonable number of Assets, but this optimises to fit the standard and make it 
easier for the catalog builder, not the user. 
If the homogeneous time-series is split to reduce the Asset count, a search will yield many duplicate Items, 
where all properties are equal, except a time stamp.

As a user searching, it makes sense to return a homogeneous chunk (Item) and have the Assets attached to this object.

Doing this creates two issues:
- Dealing with many assets >>10
- Sub-setting along time

The jump from collections to Items is done via a link item, using relation `item`.
The `/collections/<collectionId>/items` path provides a paginated list of Items. 
It follows that Assets could be represented in the same way. i.e. `/collections/<collectionId>/items/<itemId>/assets`

The other option is to summarise these large Asset lists via an aggregate object e.g. 
Zarr, [kerchunk](https://pypi.org/project/kerchunk/). 
Zarr is [not a suitable archive format](https://ntrs.nasa.gov/api/citations/20200001178/downloads/20200001178.pdf) due to 
its distributed metadata and would lead to a duplication of the data. 
This incurs a significant cost in time, compute and storage. Lightweight layers, like [Kerchunck](https://pypi.org/project/kerchunk/), could prove useful in this space.

## Proposal

1. `SPEC EXTENSION` Add a link relation "asset" to allow linkage to a separate Asset object.
2. `SPEC EXTENSION` Add an [asset-spec](https://github.com/cedadev/stac-asset-spec) which defines the Asset representation. This follows closely the [item-spec](https://github.com/radiantearth/stac-spec/blob/master/item-spec/item-spec.md), although properties are preferred in the Item. Properties are allowed as per the current asset-spec that can be used for subsetting (e.g. datetime)
3. `API EXTENSION` Add [asset-list](https://github.com/cedadev/stac-asset-list/blob/main/README.md) to provide `/collections/<collectionId>/items/<itemId>/assets` which is the list of Assets for a given itemId. This will provide a paginated list of all Assets and is linked to using a link object as part of an Item with relation "asset" .
4. `API EXTENSION` Add [asset-search](https://github.com/cedadev/stac-asset-search) to allow cross-item searching of individual data files
   - `API EXTENSION` Asset search enables API based sub-setting, using the `/collections/<collectionId>/items/<itemId>/assets` URL. e.g.`/collections/<collectionId>/items/<itemId>/assets?datatime=start/end`

## Schemas

- [Asset Spec](https://github.com/cedadev/stac-asset-spec)
- [Asset Search](https://github.com/cedadev/stac-asset-search)


## Outcome
Moving the potentially long list of Assets to a paginated endpoint allows for clients to selectively 
retrieve assets for Items which have been specifically requested, 
rather than for all items irrespective of their need. This can be represented as a paginated table
in UIs. Presenting the metadata assets with the item is still useful as it allows the user to dig 
deeper and understand whether this is the item they are interested in. 

Asset search allows for filtering, either within an item (e.g datetime or role to retrieve only data assets) or cross-item selection, 
where a subset of related assets from different items might be desired. In the first case, all assets 
point back to a single item record. In the latter case, assets may point back to a number of different items 
(potentially in different Collections).

The power of asset-level search is that that user can search dimensions of the “data hypercube” that 
yields a true intersect of relevant content, rather than being constrained by the limitations of how items 
are defined in a particular STAC catalog. A single search gets the results that are otherwise gained from 
repeated search-and-filter operations, thereby streamlining the process significantly. 
Whilst many catalog managers, and scientific domains, may not need asset-level search - it meets 
multiple requirements when mapped on earth system model data.

