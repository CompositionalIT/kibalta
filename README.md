# Kibalta
A simple F# wrapper over the .NET Azure Search SDK.

## Warning
This is a very, very early proof of concept. Things might not work. Things will be missing.
However, if you're interested in testing this out, we'd really be interested in your feedback!

## Installation
Currently we aren't bothering with nuget packages. Simply add the Kibalta github to your paket dependencies file instead:

```
github CompositionalIT/kibalta src/Kibalta.fs
```

In your `paket.references`, add the following lines:

```
TaskBuilder.fs
Microsoft.Azure.Search
File: Kibalta.fs
```

## Basic Queries

1. First make a simple typed search function.

```fsharp
let searchMyIndex =
    let client = makeClient "<searchAccountName>" "<searchIndexKey>"
    doSearch<MyType> "<indexName>" client
```

2. Now create a query.

```fsharp
open Microsoft.Azure.Search
open Microsoft.Azure.Search.Filters

let myQuery =
    azureSearch { 
        fulltext "my phrase"
        skip 10
        top 10
        includeTotalResults
    }
```

3. Now execute the search. `documents` will be deserialized as an array of `MyType`.

```fsharp
let results = searchMyIndex myQuery
let documents, facets, count = results.Result
```

## Query Syntax
Query syntax in Kibalta uses a custom F# computation expression to provide a domain-specific language (DSL) over Azure Search.

> Note: This is a experimental project. Please don't start raising issues about the pros/cons of CEs versus fluent APIs, list-based comprehensions etc. :)

### Basic query keywords
The basic operators for the CE are:

```fsharp
let myQuery =
    azureSearch {
        fulltext "my phrase" // fulltext search against all indexed fields. Optional.
        skip 10 // skip the first n records from results
        top 10 // return maximum 10 results
        includeTotalResults // include total count in response (otherwise None is returned)
        facets [ "Category"; "Region" ] // include facets in response (otherwise an empty list is returned)
    }
```

## Filters
You can specify filters using a relatively simple set of keywords / operators. Use filters to reduce the result set either in conjunction with or instead of `fulltext` searches.

```fsharp
let myQuery =
    azureSearch {
        filter (where "Age" Eq 21)
    }
```

Filters can be composed together using either `+` (AND) or `*` (OR).

```fsharp
let myQuery =
    azureSearch {
        filter ((where "Age" Eq 21 + where "FirstName" Eq "Isaac") * (where "Town" Eq "London"))
    }
```

You can create the AST yourself if you prefer rather than using `where`, `+`, and `*`, although it is more verbose:

```fsharp
let f =
    BinaryFilter(
        BinaryFilter(FieldFilter("Age", Eq, 21),
                     And, FieldFilter("FirstName", Eq, "Isaac")),
         Or, FieldFilter("Town", Eq, "London"))
```

There are also some simple helpers to make create common filters easier:

```fsharp
let filter = whereEq("Age", 21)
```

You can filter based on whether values exist for a field by comparing the field to null:

```fsharp
let myQuery =
    azureSearch {
        filter (where "Age" Ne null)
    }
```

This works in both directions, so if you want to find all documents that do not have a value for a certain field, you can do this:

```fsharp
let filter = whereEq("Age", null)
``` 

### Geo distance filtering
Azure Search supports geo distance filtering, and this is exposed in Kibalta:

```fsharp
let withinTenKmOfLondon =
    azureSearch {
        filter (whereGeoDistance "Geo" (-0.127758, 51.507351) Lt 10.)
    }
```

Again, you can manually construct the filter expression if you wish:

```fsharp
let asFilterExpr = GeoDistanceFilter ("Geo", -0.127758, 51.507351, Lt, 10.)
```

### Composing filters programmatically
If you need to *programmatically* combine filters together, you can reduce or fold over them. The `DefaultFilter` value in the Filters namespace acts as the "null" filter for folding:

```fsharp
let filters =
    [ where "Age" Gt 21
      where "Name" Eq "Isaac" ]

// AND the list of filters together.
let filter = filters |> List.fold (+) DefaultFilter
```

As this is a common task, there's a helper function to do exactly this for you:

```fsharp
let filter = combine filters
```

In conjunction with the `whereEq` function, this allows you to quickly build up composed filters, especially if you can a set of key/values:

```fsharp
let filters =
    [ "Name", "Isaac";
      "Town", "London" ]
    |> List.map whereEq
    |> combine
```

## Sorting
Lastly, you can sort results using the `sort` keyword:

```fsharp
let sortedQuery =
    azureSearch {
        sort [ ByField("Age", Descending)
               ByField("Town", Ascending) ]
    }
```

You can also perform sorting on geo-location distance. This sort normally is performed in conjunction with a `GeoFilter` filter.

```fsharp
let sortedGeoQuery =
    azureSearch {
        // Sort by furthest away from London
        sort [ ByDistance(-0.127758, 51.507351, Descending) ]
    }
```

## TO DO
Lots of things.

* Review and get feedback on the API in general.
* Support for suggestions
* Insertion of data
* Expose more features from the Azure Search SDK.