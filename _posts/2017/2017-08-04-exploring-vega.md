# Exploring Vega

__04/08/2017__

![trace](/assets/vega.jpg)

When I start to build web visualizations I consider a low level
library like `d3.js` or high level charting library like
`chart.js`. I'll usually choose a high level charting library unless
I intend to make something custom. The problem is, as with most web
libraries, the charting libs are constantly changing and that means
new APIs every bloody time I want to create some new charts. Seriously,
choosing a charting library can be a pain and there are [plenty of
options](https://www.google.com.au/search?q=javascript+charting+library&oq=javascript+ch&gs_l=psy-ab.3.0.0i67k1l2j0l2.21166.23843.0.25104.13.12.0.0.0.0.211.1520.0j8j1.9.0....0...1.1.64.psy-ab..4.9.1520...35i39k1j0i131k1.r2PQQDCKBWk).
Every time I revisit this problem I ask "why the hell are there not any
standards?" The fundamentals of a pie chart are known, yet for example
`chart.js` and `highcharts` have different APIs to create and modify them.
That's where [Vega](https://vega.github.io/vega/) comes in.

Vega is a visualization grammar. It is a standard
way of describing visualizations which can be implementation agnostic.
You create a chart specification in JSON and then pass it on to some
library to actually draw the visualization. Vega comes with built in
libraries to generate web-based views using Canvas or SVG but it's
real power is in the spec, and I could imagine people creating libraries
that generate vega views on other platforms. A simple bar chart in
Vega-lite, which is a higher level API to generate Vega specs,
will look like:

``` javascript
const spec = {
    "$schema": "https://vega.github.io/schema/vega-lite/v2.json",
    "description": "Simple bar chart",
    "data": {
        "values": [
            {"x": "A", "y": 28}, {"x": "y", "B": 55}, {"x": "C", "y": 43},
            {"x": "D", "y": 91}, {"x": "E", "y": 81}, {"x": "F", "y": 53},
            {"x": "G", "y": 19}, {"x": "H", "y": 87}, {"x": "I", "y": 52}
        ]
    },
    "mark": "bar",
    "encoding": {
        "x": {"field": "x", "type": "ordinal"},
        "y": {"field": "y", "type": "quantitative"}
    }
};
```

Which will produce a chart like this:

![Vega Bar Chart](/assets/vega-example.png)

Full angular example source code available [here](https://github.com/shusson/vega-lite-demo)
