# Ocean Data Exchange API

This API specification defines a protocol for how a client can consume data from a server and update a local database or store; or push data to a remote endpoint and modify a collection of resources.

A compliant server exposes a number of datasets. A dataset is comprised of a collection of items. Each item is a geospatial item. The item is defined in JSON and may optionally reference a binary component. The dataset exposes a feed of changes. These changes relate to the items in the collection. Each change is represented as the latest representation the geoItem that has been created, modified or deleted. Each item has unique identity across all resources in all datasets exposed by a single server.

## Motivation

The main motivation of this API is to enable the synchronisation of geospatial data resources between compliant services in a generalised manner and supporting a wide variety of data representations. 

As the API supports both PULL and PUSH semantics it can be used in a composable way to meet many use case, e.g. from edge to aggregator (push) and between aggregators (pull).

The API is designed to be super simple and bring a low bar of entry to any software to enable it to participate in a data sharing eco-system. There are just 4 endpoints defined in this API yet that is enough to enable the exchange of most forms of geospatial data.

## Concepts

# Protocol Definition

The HTTP(s) API has a single entry point and is consistently structured to allow introspection and dynamic navigation. A compliant server exposes the following endpoints.

## Datasets Endpoint

The datasets endpoint returns a list of datasets that it is exposing. 

`/datasets` 

The following Schema is used to represent a single dataset in the list:

```json
{
    "description": "Dataset Resource.",
    
    "type": "object",
    
    "required": [
        "name", "url", "changes"
    ],

    "properties": {
        "name": {
            "type": "string",
            "description": "Unique dataset name exposes by this service endpoint."
        },
        "url": {
            "type": "string",
            "description": "Dataset url"
        },
        "changes": {
            "type": "string",
            "description": "Url that exposes the changes for this dataset"
        },
        "containedTypes": {
            "type": "array",
            "description": "An array of strings. Each value is the name of a resource type exposed in this dataset.",
            "items" : {
                "type" : "string"
            }
        }
    }
}
```

An example response from a compliant endpoint could look like the following:

```json
[
    {
        "name" : "jellyfish",
        "url" : "/datasets/jellyfish",
        "changes" : "/datasets/jellyfish/changes",
        "contentType" : [ "application/json+geo"],
        "logicalType" : [ "jellyfish-observation"]
    },
    {
        "name" : "gridded-bathymetry-norway",
        "url" : "/datasets/gridded-bathymetry",
        "changes" : "/datasets/gridded-bathymetry/changes",
        "contentType" : [ "application/json+geo+netcdf"],
        "logicalType" : [ "gridded-bathymetry"]
    }
]
```

## Dataset Endpoint

The dataset endpoint allows a client to retrieve a single dataset. 

`/datasets/{datasetid}`

The schema of the dataset resource is the same as the dataset resource when returned in the list. An example response is shown below.

```json
{
    "name" : "jellyfish",
    "url" : "/datasets/jellyfish",
    "changes" : "/datasets/jellyfish/changes",
    "contentType" : [ "application/json+geo"],
    "logicalType" : [ "jellyfish-observation"]
}
```

## Changes Endpoint

The changes endpoint named in the dataset resource exposes a changes feed of items. The response from this request is an array of JSON objects. The first object is a context that is reserved for future use. The last object is a continuation object that allows clients to consume subsequent changes. 

`/datasets/{datasetid}/changes?since={sinceToken}`

The changes endpoint returns the following JSON response:

```json
[
  {
     "id" : "@context",
     "description" : "application specific / expansion point for rdf, JSON-LD and Entity Graph Data Model" 
  },

  {
    "id" : "jelly-obs-1",
    "isDeleted" : false,
    "geometry": {
        "type": "Point",
        "coordinates": [
        [
            [
            172.91173669923782,
            1.3438851951615003
            ]
        ]
        ]
    },
    "properties": {
        "title": "An observation",
        "description": "a really big one",
        "start_datetime": "2020-12-11T22:38:32.125Z",
        "end_datetime": "2020-12-11T22:38:32.327Z",
        "created": "2020-12-12T01:48:13.725Z",
        "updated": "2020-12-12T01:48:13.725Z"
    },
    "assets" : [
        {
            "type" : "jelly-image",
            "content-type" : "image/png",
            "href" : "somewhere.org/images/jelly-obs-1.png" 
        }
    ]
  },

  {
    ... more items ...
  },

  {
      "id" : "@continuation",
      "token" : "token-issued-by-server-to-get-more-data"
  }
]
```

The first object in the response is for future use and will contain context information. Likely uses are for JSON-LD, or the Entity Graph Data Model.

The main set of objects in the response are geospatial resources. 

When a client receives the geospatial resources it must process them in order. For each item it MUST do the following: 

1. check if the `isDeleted` flag is set to `true`. If so the local resource with the corresponding sourceId MUST be deleted or marked as deleted depending on the local system semantics.

2. If there is no local version of the resource it MUST be created and the id must be associated with the local object in some way. 

3. If there is a local object with the correlating id then this representation must be deleted and replaced with the one received. Implementations are free to perform deltas, but the result must be the same. In the case that the geospatial item consists of both data and binary (as described below) it is assumed that the client replaces both the binary and data elements of the locally stored representation.

The last object in the array is a continuation object and contains a property `token`. The token is an opaque string encoded with base64. This token is issued by the server and clients should NOT manipulate the contents. Manipulating the contents of a continuation token gives undefined behaviour when used as part of a subsequent request. 

A client is responsible for storing the continuation token and then using it as the since token in subsequent calls. e.g.

`/datasets/{datasetid}/changes?since=token-issued-by-server-to-get-more-data`

Note that the continuation token can be used by the server to provide paging. A client typically calls to the changes endpoint until no resources are returned. It then stores the token and then waits for some period before asking again. 

If a server has reloaded all data in a dataset and needs the client to resync from the beginning, then the server can return an HTTP Header:

`oodp-full-sync: true`

If this is returned the client should delete all local data related to this dataset, discard any continuation tokens, then make a call to `/changes` and continue as above.

The client should only store the contiuation token if it has succesfully processed all resources returned in a response.

### NetCDF & Other Binary Files

The above item type is an inline geospatial data entity. However, sometimes the data backing a resource is stored in a binary form. To represent this a geospatial item is defined as above but it uses a special asset type to link to the underlying data.

```json
{
    "id" : "bath-1",
    "isDeleted" : false,
    "bbox": [
        172.91173669923782,
        1.3438851951615003,
        172.95469614953714,
        1.3690476620161975
    ],
    "geometry": {
        "type": "Polygon",
        "coordinates": [
        [
            [
                172.91173669923782,
                1.3438851951615003
            ],
            [
                172.95469614953714,
                1.3438851951615003
            ],
            [
                172.95469614953714,
                1.3690476620161975
            ],
            [
                172.91173669923782,
                1.3690476620161975
            ],
            [
                172.91173669923782,
                1.3438851951615003
            ]
        ]
        ]
    },
    "properties": {
        "title": "An observation",
        "description": "a really big one",
        "start_datetime": "2020-12-11T22:38:32.125Z",
        "end_datetime": "2020-12-11T22:38:32.327Z",
        "created": "2020-12-12T01:48:13.725Z",
        "updated": "2020-12-12T01:48:13.725Z"
    },
    "assets" : [
        {
            "type" : "gridded-bathymetry",
            "content-type" : "application/netcdf",
            "href" : "somewhere.org/data/bath-1.ncdf" 
        }
    ]
  }
```

Clients should consider the contents of the asset as well as the properties on the geospatial item as a composite unit for transactional and update purposes.

## Dataset Push 

The datasets concept can also be used in a push based scenario. In this model a client POSTs resources to the dataset's `resources` endpoint. 

POST `/dataset/{datasetid}/resources`

```json
[
  {
     "id" : "@context",
     "description" : "application specific / expansion point for rdf, JSON-LD and Entity Graph Data Model" 
  },

  {
    "id" : "bath-1",
    "isDeleted" : false,
    "bbox": [
        172.91173669923782,
        1.3438851951615003,
        172.95469614953714,
        1.3690476620161975
    ],
    "geometry": {
        "type": "Polygon",
        "coordinates": [
        [
            [
                172.91173669923782,
                1.3438851951615003
            ],
            [
                172.95469614953714,
                1.3438851951615003
            ],
            [
                172.95469614953714,
                1.3690476620161975
            ],
            [
                172.91173669923782,
                1.3690476620161975
            ],
            [
                172.91173669923782,
                1.3438851951615003
            ]
        ]
        ]
    },
    "properties": {
        "title": "An observation",
        "description": "a really big one",
        "start_datetime": "2020-12-11T22:38:32.125Z",
        "end_datetime": "2020-12-11T22:38:32.327Z",
        "created": "2020-12-12T01:48:13.725Z",
        "updated": "2020-12-12T01:48:13.725Z"
    },
    "assets" : [
        {
            "type" : "gridded-bathymetry",
            "content-type" : "application/netcdf application/base64",
            "data" : "base64 encoded version of file" 
        }
    ]
  }

]
```

The semantics for PUSH are the same as for the pull semantics described above with the following exceptions:

1. Full sync is not accepted as header in the push scenario. 
2. The payload for push MUST include only a context followed by resources. i.e no continuation token json object.

The server returns a 200 OK if the new resource representations are correctly processed. 

Any other response code and the client MUST assume that the server has not received and stored the latest state. 

Use cases for push is typically small on site devices that are pushing data up to a cloud location which then collects resources from many clients before sharing them further or processing them. It is best practice to have different dataset endpoints for Push and Pull. 

# Security

All communication SHOULD be over HTTPs.

It is Recommended to use JWT tokens to secure API endpoints. 

The server is responsible for mapping a client to a set of claims which give access to certain locations. The set of resources exposed MUST only be for the locations that a given client can access. 

The list of datasets exposed to a client should also be limited by client ACL or claims. 

# References

1. SDShare.org - The W3C note on sharing and sync semantic descriptions
2. Universal Data API - MIMIRO open data sharing specification
3. RSS - We syndication protocol
