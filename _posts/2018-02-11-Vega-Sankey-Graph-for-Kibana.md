---
layout: post
title: "Kibana Vega Visualizations: Sankey"
---

Continuing the series on building custom Vega graphs in Kibana, our today's topic is a simple two level Sankey graph to show network traffic patterns. Each entry in the sample data has source and destination country code.  The graph will have two modes: all-to-all (default), plus it will allow users to select either the source or the destination country, and show only related traffic.

![Traffic between all countries](/assets/sankey-traffic.png "Traffic between all countries")

![Traffic from a selected country](/assets/sankey-traffic-1-cc.png "Traffic from a selected country")

## Prerequisites
See [Trends example](/Vega-Trends-Graph-for-Kibana/#prerequisites) on how to install Elasticsearch and Kibana, and load some sample data.

## Data

For this example, let's use the makelogs' field called `geo.srcdest`. This field has a pair of countries (ISO code) - a source and a destination.

```json
GET logstash-*/_search
{
  "size": 0,
  "aggs": {
    "pairs": {
      "terms": {"field": "geo.srcdest", "size": 10}
    }
  }
}
```

Try running this query in the **Dev Tools** tab - you should see something like tihs:

```yaml
{
  // ... skipping some meta information ...
  "aggregations": {
    "pairs": {
      "doc_count_error_upper_bound": 215,
      "sum_other_doc_count": 71481,
      "buckets": [
        {
          "key": "CN:CN",
          "doc_count": 3438
        },
        {
          "key": "IN:CN",
          "doc_count": 3018
        },
        // ... 8 more objects
    ]
  }
}
```

## Visualization Strategy

We can think of the sankey graph as having nodes organized into stacks, and edges connecting the nodes.  For this graph we only have two stacks - the source and the destination. The two stacks are drawn as vertical bars, one on each side of the screen.  Each stack has countries stacked one on top of the other.  Each country consists of nodes - portions of the source country that are connected with the nodes in the destination country.

![Sankey components](/assets/sankey-components.png "Sankey components")

We need to have several data tables to draw the above graph:

* nodes - this table will not be used for drawing directly, but it will be used as the data source for the other tables. We can visualize them as black boxes in the example above - we have 12 of them here. Each node needs to have its own (y0,y1) range, the stack, the country code, and the number of documents it represents.
* edges - a list of lines (6 in this case), each connecting a node on the left and the right sides. The line will need a pair of (x, y) coordinates, line thickness (strikeWidth), and color (same as source node)
* groups - each group combines all of the nodes for the same country for the same stack. In this graph we also have 6 of them, 3 on each side. Each group needs to have its stack (src or dst), the starting and the ending Y value (y0 and y1), and the country code.


(incomplete)


## Vega code

See code comments for explanation of what each line does, or read the [vega documentation](https://vega.github.io/vega/docs/).  Note that this code uses [HJSON](https://hjson.org/), a more readable form of JSON.

```yaml
{
  $schema: https://vega.github.io/schema/vega/v3.0.json
  data: [
    {
      name: data
      url: {
        %context%: true
        %timefield%: @timestamp
        index: logstash-*
        body: {
          aggs: {
            pairs: {
              terms: {field: "geo.srcdest", size: 100000}
            }
          }
          size: 0
        }
      }
      format: {property: "aggregations.pairs.buckets"}
      transform: [
        {type: "formula", expr: "slice(datum.key, 0, 2)", as: "src"}
        {type: "formula", expr: "slice(datum.key, 3, 5)", as: "dst"}
      ]
    }
    {
      name: nodes
      source: data
      transform: [
        {
          type: filter
          expr: !groupSelector || groupSelector.src == datum.src || groupSelector.dst == datum.dst
        }
        {
          type: fold
          fields: ["src", "dst"]
          as: ["stack", "cc"]
        }
        {
          type: formula
          expr: if(datum.stack == 'src', datum.src+datum.dst, datum.dst+datum.src)
          as: sortField
        }
        {
          type: stack
          groupby: ["stack"]
          sort: {field: "sortField"}
          field: doc_count
        }
        {type: "formula", expr: "(datum.y0+datum.y1)/2", as: "yc"}
      ]
    }
    {
      name: groups
      source: nodes
      transform: [
        {
          type: aggregate
          groupby: ["stack", "cc"]
          fields: ["doc_count"]
          ops: ["sum"]
          as: ["total"]
        }
        {
          type: stack
          groupby: ["stack"]
          sort: {field: "cc"}
          field: total
        }
        {type: "formula", expr: "scale('y', datum.y0)", as: "scaledY0"}
        {type: "formula", expr: "scale('y', datum.y1)", as: "scaledY1"}
        {type: "formula", expr: "datum.stack == 'src'", as: "rightLabel"}
        {
          type: formula
          expr: datum.total/domain('y')[1]
          as: percentage
        }
      ]
    }
    {
      name: destinationNodes
      source: nodes
      transform: [
        {type: "filter", expr: "datum.stack == 'dst'"}
      ]
    }
    {
      name: edges
      source: nodes
      transform: [
        {type: "filter", expr: "datum.stack == 'src'"}
        {
          type: lookup
          from: destinationNodes
          key: key
          fields: ["key"]
          as: ["target"]
        }
        {
          type: linkpath
          orient: horizontal
          shape: {signal: "'diagonal'"}
          sourceY: {expr: "scale('y', datum.yc)"}
          sourceX: {expr: "scale('x', 'src') + bandwidth('x')"}
          targetY: {expr: "scale('y', datum.target.yc)"}
          targetX: {expr: "scale('x', 'dst')"}
        }
        {
          type: formula
          expr: range('y')[0]-scale('y', datum.doc_count)
          as: strokeWidth
        }
        {
          type: formula
          expr: datum.doc_count/domain('y')[1]
          as: percentage
        }
      ]
    }
  ]
  scales: [
    {
      name: x
      type: band
      range: width
      domain: ["src", "dst"]
      paddingOuter: 0.05
      paddingInner: 0.95
    }
    {
      name: y
      type: linear
      range: height
      domain: {data: "nodes", field: "y1"}
    }
    {
      name: color
      type: ordinal
      range: category
      domain: {data: "data", field: "src"}
    }
  ]
  axes: [
    {orient: "bottom", scale: "x"}
    {orient: "left", scale: "y"}
  ]
  marks: [
    {
      type: path
      name: edgeMark
      from: {data: "edges"}
      encode: {
        update: {
          stroke: [
            {
              test: groupSelector && groupSelector.stack=='src'
              scale: color
              field: dst
            }
            {scale: "color", field: "src"}
          ]
          strokeWidth: {field: "strokeWidth"}
          path: {field: "path"}
          strokeOpacity: {
            signal: if(!groupSelector && (groupHover.src == datum.src || groupHover.dst == datum.dst), 0.6, 0.2)
          }
          tooltip: {
            signal: datum.src + ' → ' + datum.dst + '    ' + format(datum.doc_count, ',.0f') + '   (' + format(datum.percentage, '.1%') + ')'
          }
        }
        hover: {
          strokeOpacity: {value: 1}
        }
      }
    }
    {
      type: rect
      name: nodeMark
      from: {data: "groups"}
      encode: {
        enter: {
          fill: {scale: "color", field: "cc"}
          width: {scale: "x", band: 1}
        }
        update: {
          x: {scale: "x", field: "stack"}
          y: {field: "scaledY0"}
          y2: {field: "scaledY1"}
          fillOpacity: {signal: "if(groupSelector, 1, 0.6)"}
          tooltip: {
            signal: datum.cc + '   ' + format(datum.total, ',.0f') + '   (' + format(datum.percentage, '.1%') + ')'
          }
        }
        hover: {
          fillOpacity: {value: 1}
        }
      }
    }
    {
      type: text
      from: {data: "groups"}
      interactive: false
      encode: {
        update: {
          x: {
            signal: scale('x', datum.stack) + if(datum.rightLabel, bandwidth('x') + 5, -5)
          }
          yc: {signal: "(datum.scaledY0 + datum.scaledY1)/2"}
          align: {signal: "if(datum.rightLabel, 'left', 'right')"}
          baseline: {value: "middle"}
          fontWeight: {value: "bold"}
          text: {signal: "if(abs(datum.scaledY0-datum.scaledY1) > 13, datum.cc, '')"}
        }
      }
    }
    {
      type: group
      data: [
        {
          name: dataForShowAll
          values: [
            {}
          ]
          transform: [
            {type: "filter", expr: "groupSelector"}
          ]
        }
      ]
      encode: {
        enter: {
          xc: {signal: "width/2"}
          y: {value: 30}
          width: {value: 80}
          height: {value: 30}
        }
      }
      marks: [
        {
          type: group
          name: groupReset
          from: {data: "dataForShowAll"}
          encode: {
            enter: {
              opacity: {value: 1}
              cornerRadius: {value: 6}
              fill: {value: "#f5f5f5"}
              stroke: {value: "#c1c1c1"}
              strokeWidth: {value: 2}
              height: {signal: "item.mark.group.height"}
              width: {signal: "item.mark.group.width"}
            }
            hover: {
              opacity: {value: 0.7}
            }
          }
          marks: [
            {
              type: text
              interactive: false
              encode: {
                enter: {
                  xc: {signal: "item.mark.group.width/2"}
                  yc: {signal: "item.mark.group.height/2 + 2"}
                  align: {value: "center"}
                  baseline: {value: "middle"}
                  fontWeight: {value: "bold"}
                  text: {value: "Show All"}
                }
              }
            }
          ]
        }
      ]
    }
  ]
  signals: [
    {
      name: groupHover
      value: {}
      on: [
        {
          events: @nodeMark:mouseover
          update: "{src:datum.stack=='src' && datum.cc, dst:datum.stack=='dst' && datum.cc}"
        }
        {events: "mouseout", update: "{}"}
      ]
    }
    {
      name: groupSelector
      value: false
      on: [
        {
          events: @nodeMark:click!
          update: "{stack:datum.stack, src:datum.stack=='src' && datum.cc, dst:datum.stack=='dst' && datum.cc}"
        }
        {
          events: [
            {type: "click", markname: "groupReset"}
          ]
          update: "false"
        }
      ]
    }
  ]
}
```

## Highlighting components

To highlight components of the stacks and generate the above explanation image, replace `url` section with these hardcoded values, and add an additional mark after all other marks.

```yaml
values: {
  aggregations: { pairs: { buckets: [
    {key: "aa:cc", doc_count: 7}
    {key: "aa:bb", doc_count: 4}
    {key: "bb:aa", doc_count: 8}
    {key: "bb:bb", doc_count: 6}
    {key: "bb:cc", doc_count: 3}
    {key: "cc:aa", doc_count: 9}
]}}}
```

```yaml
{
  type: rect
  from: {data: "nodes"}
  encode: {
    enter: {
      stroke: {value: "#000"}
      strokeWidth: {value: 2}
      width: {scale: "x", band: 1}
      x: {scale: "x", field: "stack"}
      y: {field: "y0", scale: "y"}
      y2: {field: "y1", scale: "y"}
    }
  }
}
```

## Useful links:
* [Kibana Vega docs](https://www.elastic.co/guide/en/kibana/master/vega-graph.html)
* [Vega documentation](https://vega.github.io/vega/docs/)
* [Vega examples](https://vega.github.io/vega/examples/)
* [Vega-Lite documentation](https://vega.github.io/vega-lite/docs/)
* [Vega-Lite examples](https://vega.github.io/vega-lite/examples/)
