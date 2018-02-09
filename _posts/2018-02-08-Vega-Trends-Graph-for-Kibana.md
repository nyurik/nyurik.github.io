---
layout: post
title: Kibana's Missing Vis - One Vega graph at a time - Trends
---

There are plenty of [pending requests](https://github.com/elastic/kibana/issues?q=is%3Aopen+label%3A%22%3ANew+Vis%22+sort%3Acomments-desc) for Kibana visualizations. Now with Vega support, we no longer need to wait, anyone can do most of them by themselves!  This is the first in a series of blog posts on how to build Vega visualizations in Kibana.

## Prerequisites

* Download [Elasticsearch 6.2 or later](https://www.elastic.co/downloads/elasticsearch)
* Unzip and run `bin/elasticsearch`
* Download [Kibana 6.2 or later](https://www.elastic.co/downloads/kibana) _(same version as Elasticsearch)_
* Unzip and run `bin/kibana`
* Use [makelogs utility](https://github.com/elastic/makelogs#makelogs) to generate sample data. Install it with `npm install -g makelogs`, run with `makelogs`. You may want to generate bigger dataset with a `-c 100k` parameter. This assumes you already have [NPM](https://www.npmjs.com/get-npm). **Don't do this on a production cluster!**
* Navigate to http://localhost:5601/
* On **Management** tab (last icon on the left), create a new index pattern: enter `logstash-*` for the index pattern, click **Next step**, choose `@timestamp` for the time filter, click **Create index pattern**.
* Test that everything works by using **Discover** tab, setting the time filter in the upper right corner to the last 1 hour, and observe randomly generated data.

## [Trends Indicator](https://github.com/elastic/kibana/issues/2647)
![Trends indicator](/assets/trends.png "Trends indicator")

Lets build a very simple trend indicator to compare the number of events in the last 10 minutes vs the 10 minutes before that.  We can ask Elasticsearch for the 10 min aggregates, but those aggregates would be aligned on 10 minute boundaries, rather than being "last 10 minutes". Instead, we will ask for the last 20 aggregates 1 minute each, excluding the current (incomplete) minute. The `extended_bounds` param ensures that even when there is no data, we still get a count=0 result for each bucket.  Try running this query in the **Dev Tools** tab - copy/paste it, and hit the green play button.

```json
GET logstash-*/_search
{
  "aggs": {
    "time_buckets": {
      "date_histogram": {
        "field": "@timestamp",
        "interval": "1m",
        "extended_bounds": { "min": "now-20m/m", "max": "now-1m/m" }
      }
    }
  },
  "query": {
    "range": {
      "@timestamp": { "gte": "now-20m/m", "lte": "now-1m/m" }
    }
  },
  "size": 0
}
```

Results:

```yaml
{
  // ... skipping some meta information ...
  "aggregations": {
    "time_buckets": {
      "buckets": [
        {
          "key_as_string": "2018-02-09T00:52:00.000Z",
          "key": 1518137520000,
          "doc_count": 1
        },
        {
          "key_as_string": "2018-02-09T00:53:00.000Z",
          "key": 1518137580000,
          "doc_count": 3
        },
        // ... 18 more objects
    ]
  }
}
```

Now that we know how to request needed data, and what that data would look like, we can build the visualization itself. Click **visualization** tab, click the blue plus sign, scroll down and click the **Vega** visualization.  You should now see the default demo visualization. Delete all the code from the left panel, paste this code instead, and click the blue **play** button. Voila, you now have an indicator that shows if the number of events in the last 10 minutes increased or decreased.  See code comments for explanation of what each line does, or read the [vega documentation](https://vega.github.io/vega/docs/).  Note that this code uses [HJSON](https://hjson.org/), a more readable form of JSON.

```yaml
{
  # Indicates if this code is Vega or Vega-Lite
  $schema: https://vega.github.io/schema/vega/v3.0.json
  # All our data sources are listed in this section
  data: [
    {
      name: values
      # when url is an object, it is treated as an Elasticsearch query
      url: {
        index: logstash-*
        body: {
          aggs: {
            time_buckets: {
              date_histogram: {
                field: @timestamp
                interval: 1m
                extended_bounds: {min: "now-20m/m", max: "now-1m/m"}
              }
            }
          }
          query: {
            range: {
              @timestamp: {gte: "now-20m/m", lte: "now-1m/m"}
            }
          }
          size: 0
        }
      }
      # We only need a specific array of values from the response
      format: {property: "aggregations.time_buckets.buckets"}
      # Perform these transformations on each of the 20 values from ES
      transform: [
        # Add "row_number" field to each value -- 1..20
        {
          type: window
          ops: ["row_number"]
          as: ["row_number"]
        }
        # Break results into 2 groups, group 0 with row_number 1..10, and 1 - 11..20
        {type: "formula", expr: "floor((datum.row_number-1)/10)", as: "group"}
        # Group 20 values into an array of two elements, one for each group, and sum up the doc_count fields as "count"
        {
          type: aggregate
          groupby: ["group"]
          ops: ["sum"]
          fields: ["doc_count"]
          as: ["count"]
        }
        # At this point "values" data source should look like this:
        # [ {group:0, count: nnn}, {group:1, count: nnn} ]
        # Check this with F11 (browser debug tools), and running this in console:
        #   VEGA_DEBUG.view.data('values')
      ]
    }
    {
      # Here we create an artificial dataset with just a single empty object
      name: results
      values: [
        {}
      ]
      # we use transforms to add various dynamic values to the single object
      transform: [
        # from the 'values' dataset above, get the first count as "last",
        # and the one before that as "prev" fields.
        {type: "formula", expr: "data('values')[0].count", as: "last"}
        {type: "formula", expr: "data('values')[1].count", as: "prev"}
        # Set two boolean fields "up" and "down" to simplify drawing
        {type: "formula", expr: "datum.last>datum.prev", as: "up"}
        {type: "formula", expr: "datum.last<datum.prev", as: "down"}
        # Calculate the change as percentage, with special handling of 0
        {
          type: formula
          expr: if(datum.last==0, if(datum.prev==0,0,-1), (datum.last-datum.prev)/datum.last)
          as: percentChange
        }
        # Calculate which symbol to show - up or down arrow, or a no-change dot
        {type: "formula", expr: "if(datum.up,'🠹',if(datum.down,'🠻','🢝'))", as: "symbol"}

      ]
    }
  ]
  # Marks is a list of all drawing elements. Here we only need a simple textual element.
  marks: [
    {
      type: text
      # Text mark executes once for each of the values in the results, but results has just one value in it. We could have also used it to draw a list of values.
      from: {data: "results"}
      encode: {
        update: {
          # Combine the symbol, last value, and the formatted percentage change into one string
          text: {
            signal: datum.symbol + ' ' + datum.last + ' ('+ format(datum.percentChange, '+.1%') + ')'
          }
          # decide which color to use, depending on the value being up, down, or unchanged
          fill: {signal: "if(datum.up, '#00ff00', if(datum.down, '#ff0000', '#0000ff'))"}
          # positioning the text in the center of the window
          align: {value: "center"}
          baseline: {value: "middle"}
          xc: {signal: "width/2"}
          yc: {signal: "height/2"}
          # Make the size of the font adjust based on the size of the graph
          fontSize: {signal: "min(width/10, height)/1.3"}
        }
      }
    }
  ]
}
```

Useful links:
* [Kibana Vega docs](https://www.elastic.co/guide/en/kibana/master/vega-graph.html)
* [Vega documentation](https://vega.github.io/vega/docs/)
* [Vega examples](https://vega.github.io/vega/examples/)
* [Vega-Lite documentation](https://vega.github.io/vega-lite/docs/)
* [Vega-Lite examples](https://vega.github.io/vega-lite/examples/)
