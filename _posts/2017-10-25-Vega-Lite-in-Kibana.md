---
layout: post
title: Getting Started with Elasticsearch, Kibana, and Vega-Lite graphs
---

Vega offer a way to create custom visualizations, beyond the ones that come standard with Kibana.  In this post we will use Vega-Lite (simplified Vega) syntax to show data from Elasticsearch.

* First, lets create some sample data. In Kibana, on the left hand side, click [Dev Tools Console](https://www.elastic.co/guide/en/kibana/current/console-kibana.html), copy this text, click in the middle of it, and hit the green “play” button.  This creates a new `cars` index, and defines the fields structure. You should see `"acknowledged": true, "shards_acknowledged": true` on the right.


```
PUT /cars
{
 "mappings" : {
  "_default_" : {
   "properties" : {
      "Name": {"type": "keyword" },
      "Miles_per_Gallon": {"type": "float" },
      "Cylinders": {"type": "integer" },
      "Horsepower": {"type": "float" },
      "Weight_in_lbs": {"type": "float" },
      "Year": {"type": "date" }
}}}}
```

* Next, clear the dev console, open [this link](cars-sample-data.json), copy its entire content to the dev console, and also hit the green play button on it.  You should see a long response with  `"errors": false,`.


* Now, open Visualizers, create a new Vega visualizer (it should be at the bottom), and copy/paste this Vega-Lite graph.  You should immediately see all of the imported data as a scatter plot.

```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v2.json",
  "data": {
    "url": {
      "index": "cars",
      "%context_query%": true,
      "body": {"size": 10000}
    },
    "format": {"property": "hits.hits"}
  },
  "mark": "circle",
  "encoding": {
    "tooltip": {"field": "_source.Name", "type": "ordinal"},
    "y": {"field": "_source.Horsepower", "type": "quantitative"},
    "x": {"field": "_source.Miles_per_Gallon", "type": "quantitative"}
  }
}
```


Let’s analyse this graph piece by piece.

```
“version”: "...”
```

With version, we specify that this is a Vega-Lite graph, as opposed to a Vega graph. Vega offers much more flexibility, at the expense of being slightly more verbose.

```
data:
```
Data could be either a static URL, or an object that describes ElasticSearch query.  For our example, we simply get the maximum number of the original documents (10,000) to keep things simple. Unlike Vega, Vega-Lite `data` can only have a single data source.  The data will be returned as:

```json
{
  "hits": {
    "hits": [
      {
        "_id": "AV9P65tJNuzjRfBwoDV5",
        "_source": {
          "Name": "ford mustang boss 302",
          "Miles_per_Gallon": null,
          "Cylinders": 8,
          "Horsepower": 140,
          "Weight_in_lbs": 3353,
          "Year": "1970-01-01"
        }
      },
      {
        "_id": "AV9P65tKNuzjRfBwoDV8",
        "_score": 1,
        "_source": {
          "Name": "toyota corona mark ii",
...
        }
      },
	...
    ]
}}
```

Some values were removed for brevity. For our graph, we are only interested in the list of values inside the `hits.hits`, so `property` parameter tells Vega-Lite to ignore everything else. Note that the actual values are stored inside the `_source` subfield, so later we need to use `_source.<field>` to access them.

**Hint:** you can use browser debug tools to view the raw data responses from ES - hit F12, and make a minor graph modification (e.g. add a space).  You can also view [Vega debugging](https://vega.github.io/vega/docs/api/debugging/) state with `VEGA_DEBUG` variable, e.g. `VEGA_DEBUG.view.data('source_0')[0]` would show the first data element.

[`marks`](https://vega.github.io/vega-lite/docs/mark.html) is what determines how data should be shown. It can also be a `bar`, `line`, and many other.

[encoding channels](https://vega.github.io/vega-lite/docs/encoding.html) set the mapping between our data fields, and how they should be drawn, e.g. that certain fields should represent the X or Y coordinate, the color of the line or the shape of the symbol. Each field needs to indicate [the type](https://vega.github.io/vega-lite/docs/type.html) of data it contains - quantitative, temporal, ordinal, or nominal.
