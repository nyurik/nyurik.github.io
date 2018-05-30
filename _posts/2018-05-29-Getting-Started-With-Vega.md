---
layout: post
title: "Tutorial: Getting Started with Vega"
---

[Vega](https://vega.github.io/vega/examples/) declarative grammar is a powerful way to visualize your data. Let's learn Vega language with a few simple examples.

Open [Vega editor](https://vega.github.io/editor/) - a convenient tool to experiment with the raw Vega (it has no ElasticSearch customizations). Copy this code. You should see "Hello Vega!" text in the right panel.

![Text Mark Example](/assets/intro-textmark.png "Text Mark Example")

```json
{
  "$schema": "https://vega.github.io/schema/vega/v3.json",
  "width": 100, "height": 30,
  "marks": [
    {
      "type": "text",
      "encode": {
        "enter": {
          "text":     { "value": "Hello Vega!" },
          "align":    { "value": "center"},
          "baseline": { "value": "middle"},
          "stroke":   { "value": "#A32299" },
          "x":        { "signal": "width/2" },
          "y":        { "signal": "height/2" }
        }
      }
    }
  ]
}
```

The [marks](https://vega.github.io/vega/docs/marks/) block is an array of drawing primitives, such as text, lines, and rectangles. Each mark has a large number of parameters specified inside the [encoding set](https://vega.github.io/vega/docs/marks/#encode). Each parameter is set to either a constant (value) or a result of a computation (signal) in the "enter" stage.  For the text mark, we specify the text string, make sure text is properly positioned relative to given coordinates, and set text's color. The `x` and `y` coordinates are computed based on the width and height of the graph, positioning the text in the middle.  There are many other [text mark parameters](https://vega.github.io/vega/docs/marks/text/). There is also an interactive text mark demo graph to try different parameter values.

The `$schema` is simply an ID of the required Vega engine version. The `width` and `height` set the initial drawing canvas size. The final graph size may change in some cases, based on the content and [autosizing options](https://vega.github.io/vega/docs/specification/#autosize). Note that Kibana's default `autosize` is `fit` instead of `pad`, thus making `height` and `width` optional.

# Data-driven graph

Our next step is to draw a data-driven graph using the [rectangle mark](https://vega.github.io/vega/docs/marks/rect/).  The [data section](https://vega.github.io/vega/docs/data/) allows multiple data sources, either hardcoded, or as a URL. In Kibana, you may also use direct Elasticsearch queries. Our `vals` data table has 4 rows and two columns - `category` and `count`.  We use `category` to position the bar on the `x` axis, and `count` for the bar's height.  Note that `y` coordinate's 0 is at the top, and increases downwards.

![Rect Mark Example](/assets/intro-rectmark1.png "Rect Mark Example")

```json
{
  "$schema":"https://vega.github.io/schema/vega/v3.json",
  "width": 300, "height": 100,
  "data": [ {
    "name": "vals",
    "values": [
      {"category": 50,  "count": 30},
      {"category": 100, "count": 80},
      {"category": 150, "count": 10},
      {"category": 200, "count": 50}
    ]
  } ],
  "marks": [ {
    "type": "rect",
    "from": { "data": "vals" },
    "encode": {
      "enter": {
        "x":     {"field": "category"},
        "width": {"value": 30},
        "y":     {"field": "count"},
        "y2":    {"value": 0}
}}} ]}
```

The `rect` mark specifies `vals` as the source of data. The mark is drawn once per source data value (also known as a table row or a **datum**). Unlike the previous graph, the `x` and `y` parameters are not hardcoded, but come from the fields of the datum.

# Scaling

[Scaling](https://vega.github.io/vega/docs/scales/) is one of the most important, but somewhat tricky concept in Vega.  In the previous examples, screen pixel coordinates were hardcoded in the data. While it made things simpler, the real data almost never comes in that form. Instead, the source data comes in its own units (e.g. number of events), and it is up to the graph to scale source values to the desired graph size in pixels.

In this example, we use [linear scale](https://vega.github.io/vega/docs/scales/#linear) -- essentially a mathematical function to convert a value from the [domain](https://vega.github.io/vega/docs/scales/#domain) of the source data (in this graph `count` values of 1000..8000, and including `count=0`), to the desired [range](https://vega.github.io/vega/docs/scales/#range) (in our case the height of the graph - 0..299).  Adding `"scale": "yscale"` to both `y` and `y2` parameter uses `yscale` scaler to convert `count` to screen coordinates (0 becomes 299, and 8000 - largest value in the source data - to becomes 0).  Note that the `height` range parameter is a special case, making 0 appear at the bottom of the graph.

![Rect Mark with Scaling Example](/assets/intro-rectmark2.png "Rect Mark with Scaling Example")

```json
{
  "$schema":"https://vega.github.io/schema/vega/v3.json",
  "width": 400, "height": 100,
  "data": [ {
    "name": "vals",
    "values": [
      {"category": 50,  "count": 3000},
      {"category": 100, "count": 8000},
      {"category": 150, "count": 1000},
      {"category": 200, "count": 5000}
    ]
  } ],
 "scales": [
    {
      "name": "yscale",
      "type": "linear",
      "zero": true,
      "domain": {"data": "vals", "field": "count"},
      "range": "height"
    }
  ],
  "marks": [ {
    "type": "rect",
    "from": { "data": "vals" },
    "encode": {
      "enter": {
        "x":     {"field": "category"},
        "width": {"value": 30},
        "y":     {"scale": "yscale", "field": "count"},
        "y2":    {"scale": "yscale", "value": 0}
}}} ]}
```

# Band scaling

For our tutorial we will need another one of 15+ [Vega scale types](https://vega.github.io/vega/docs/scales/#types) - a band scale. This scale is used when we have a set of values (like categories), that need to be represented as bands - each occupying same proportional width of the graph's total width. Here, the band scale gives each one of the 4 unique categories the same proportional width (about 400/4, minus 5% padding between bars and on both ends).  The `{"scale": "xscale", "band": 1}` gets the 100% of the band's width for the mark's `width` parameter.

![Rect Mark with Band Scaling Example](/assets/intro-rectmark3.png "Rect Mark with Band Scaling Example")

```json
{
  "$schema":"https://vega.github.io/schema/vega/v3.json",
  "width": 400, "height": 100,
  "data": [ {
    "name": "vals",
    "values": [
      {"category": "Oranges", "count": 3000},
      {"category": "Pears",   "count": 8000},
      {"category": "Apples",  "count": 1000},
      {"category": "Peaches", "count": 5000}
    ]
  } ],
 "scales": [
    {
      "name": "yscale",
      "type": "linear",
      "zero": true,
      "domain": {"data": "vals", "field": "count"},
      "range": "height"
    },
    {
      "name": "xscale",
      "type": "band",
      "domain": {"data": "vals", "field": "category"},
      "range": "width",
      "padding": 0.05
    }
  ],
  "marks": [ {
    "type": "rect",
    "from": { "data": "vals" },
    "encode": {
      "enter": {
        "x":     {"scale": "xscale", "field": "category"},
        "width": {"scale": "xscale", "band": 1},
        "y":     {"scale": "yscale", "field": "count"},
        "y2":    {"scale": "yscale", "value": 0}
}}} ]}
```

# Axes
A typical graph wouldn't be complete without the axes labels.  Axis definition uses the same scales we defined earlier, so adding them is as simple as referencing the scale by its name, and specifying the placement side.  Add this code as the top level element to the last code example.

![Rect Mark with Axes Example](/assets/intro-rectmark-axes.png "Rect Mark with Axes Example")

```json
  "axes": [
    {"scale": "yscale", "orient": "left"},
    {"scale": "xscale", "orient": "bottom"}
  ],
```

Note that the total graph size has increased automatically to accommodate these axes.  The graph can be forced to stay in the original size by adding `"autosize": "fit",` top-level parameter.

# Data Transformations and Conditionals
Data often needs additional manipulation before it can be used for drawing. Vega provides numerous [transformations](https://vega.github.io/vega/docs/transforms/) to help with that. Let's use the most common [formula transformation](https://vega.github.io/vega/docs/transforms/formula/) to dynamically add a random `count` value field to each of the source datums. Also, in this graph we will manipulate the fill color of the bar, making it red if the value is less than 333, yellow for less than 666, and green otherwise. Note that this could have been done with a scale instead, mapping domain of the source data to the set of colors, or to a [color scheme](https://vega.github.io/vega/docs/schemes/).

![Random Data Example](/assets/intro-random.png "Random Data Example")

```json
{
  "$schema":"https://vega.github.io/schema/vega/v3.json",
  "width": 400, "height": 200,
  "data": [ {
    "name": "vals",
    "values": [
      {"category": "Oranges"},
      {"category": "Pears"},
      {"category": "Apples"},
      {"category": "Peaches"},
      {"category": "Bananas"},
      {"category": "Grapes"}
    ],
    "transform": [
      {"type": "formula", "as": "count", "expr": "random()*1000"}
    ]
  } ],
 "scales": [
    {
      "name": "yscale",
      "type": "linear",
      "zero": true,
      "domain": {"data": "vals", "field": "count"},
      "range": "height"
    },
    {
      "name": "xscale",
      "type": "band",
      "domain": {"data": "vals", "field": "category"},
      "range": "width",
      "padding": 0.05
    }
  ],
  "axes": [
    {"scale": "yscale", "orient": "left"},
    {"scale": "xscale", "orient": "bottom"}
  ],
  "marks": [ {
    "type": "rect",
    "from": { "data": "vals" },
    "encode": {
      "enter": {
        "x":     {"scale": "xscale", "field": "category"},
        "width": {"scale": "xscale", "band": 1},
        "y":     {"scale": "yscale", "field": "count"},
        "y2":    {"scale": "yscale", "value": 0},
        "fill":  [
          {"test": "datum.count < 333", "value": "red"},
          {"test": "datum.count < 666", "value": "yellow"},
          {"value": "green"}
        ]
}}} ]}
```

# Useful links
* [Kibana Vega docs](https://www.elastic.co/guide/en/kibana/master/vega-graph.html)
* [Vega documentation](https://vega.github.io/vega/docs/)
* [Vega examples](https://vega.github.io/vega/examples/)
* [Vega-Lite documentation](https://vega.github.io/vega-lite/docs/)
* [Vega-Lite examples](https://vega.github.io/vega-lite/examples/)
