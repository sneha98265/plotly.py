---
jupyter:
  jupytext:
    notebook_metadata_filter: all
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
      jupytext_version: 1.16.4
  kernelspec:
    display_name: Python 3 (ipykernel)
    language: python
    name: python3
  language_info:
    codemirror_mode:
      name: ipython
      version: 3
    file_extension: .py
    mimetype: text/x-python
    name: python
    nbconvert_exporter: python
    pygments_lexer: ipython3
    version: 3.11.10
  plotly:
    description: Recommendations for increased speed, improved interactivity, and
      the ability to plot even more data!
    display_as: basic
    language: python
    layout: base
    name: High Performance Visualization
    order: 14
    permalink: python/performance/
    redirect_from: python/webgl-vs-svg/
    thumbnail: thumbnail/webgl.jpg
---

## WebGL

`plotly` figures are rendered by web browsers, which broadly speaking have two families of capabilities for rendering graphics:

- The SVG API, which supports vector rendering.
- The Canvas API, which supports raster rendering, and can exploit GPU hardware acceleration via a browser technology known as WebGL.

Each `plotly` trace type is rendered with either SVG or WebGL. The following trace types use WebGL for rendering:

* Accelerated versions of SVG trace types: `scattergl`, `scatterpolargl`,
* High-performance multidimensional trace types: `splom`, or `parcoords`
* 3-d trace types `scatter3d`, `surface`, `mesh3d`, `cone`, `streamtube`
* Mapbox Gl JS-powered trace types: `scattermapbox`, `choroplethmapbox`, `densitymapbox`

### WebGL Limitations and Tradeoffs

WebGL is a powerful technology for accelerating computation but comes with some strict limitations:

1. GPU requirement: WebGL is a GPU (graphics card) technology and therefore requires specific hardware which is available in most but not all cases and is supported by most but not all browsers.
2. Rasterization: WebGL-rendered data is drawn as a grid of pixels rather than as individual shapes, so can appear pixelated or fuzz in certain cases, and when exported to static file formats will appear pixelated on zoom. In addition: text rendering will differ between SVG and WebGL-powered traces.
3. Context limits: browsers impose a strict limit on the number of WebGL "contexts" that any given web document can access. WebGL-powered traces in `plotly` can use multiple contexts in some cases but as a general rule, **it may not be possible to render more than 8 WebGL-involving figures on the same page at the same time.** See the following section, Multiple WebGL Contexts, for more details.
4. Size limits: browsers impose hardware-dependent limits on the height and width of figures using WebGL which users may encounter with extremely large plots (e.g. tens of thousands of pixels of height).

In addition to the above limitations, the WebGL-powered version of certain SVG-powered trace types (`scattergl`, `scatterpolargl`) are not complete drop-in replacements for their SVG counterparts yet
* Available symbols will differ.
* Area fills are not yet supported in WebGL.
* Range breaks on time-series axes are not yet supported.
* Axis range heuristics may differ.

#### Multiple WebGL Contexts

*New in 5.19*

Most browsers have a limit of between 8 and 16 WebGL contexts per page. A Plotly WebGL-based figure may use multiple WebGL contexts, but generally you'll be able to render between 4 and 8 figures on one page.

If you exceed the browser limit on WebGL contexts, some figures won't render and you'll see an error. In the console in Chrome, for example, you'll see the error: "Too many active WebGL contexts. Oldest context will be lost".

If you encounter WebGL context limits when using WebGL-based figures, you can use [Virtual WebGL](https://github.com/greggman/virtual-webgl), which virtualizes a single WebGL context into multiple contexts.

To use it, in the environment where your Plotly figures are being rendered, load the Virtual WebGL script, "https://unpkg.com/virtual-webgl@1.0.6/src/virtual-webgl.js", for example, using a `<script>` tag.

In a Jupyter notebook environment that supports magic commands, you can load it with the [HTML magic command](https://ipython.readthedocs.io/en/stable/interactive/magics.html#cellmagic-html):


```
%%html
<script src=“https://unpkg.com/virtual-webgl@1.0.6/src/virtual-webgl.js”></script>
```


### WebGL for Scatter Performance

In the examples below we show that it is possible to represent up to around a million points with WebGL-enabled traces.
For larger datasets, or for a clearer visualization of the density of points,
it is also possible to use [datashader](/python/datashader/).

### WebGL with Plotly Express

The `render_mode` argument to supported Plotly Express functions (e.g. `scatter` and `scatter_polar`) can be used to enable WebGL rendering.

> **Note** The default `render_mode` is `"auto"`, in which case Plotly Express will automatically set `render_mode="webgl"` if the input data is more than 1,000 rows long. In this case, WebGl can be disabled by setting `render_mode=svg`.

Here is an example that creates a 100,000 point scatter plot using Plotly Express with WebGL rendering explicitly enabled.

```python
import plotly.express as px

import pandas as pd
import numpy as np

np.random.seed(1)

N = 100000

df = pd.DataFrame(dict(x=np.random.randn(N),
                       y=np.random.randn(N)))

fig = px.scatter(df, x="x", y="y", render_mode='webgl')

fig.update_traces(marker_line=dict(width=1, color='DarkSlateGray'))

fig.show()
```


#### WebGL with 1,000,000 points with Graph Objects

If Plotly Express does not provide a good starting point, it is also possible to use [the more generic `go.Scattergl` class from `plotly.graph_objects`](/python/graph-objects/).

```python
import plotly.graph_objects as go

import numpy as np

N = 1_000_000

fig = go.Figure()

fig.add_trace(
    go.Scattergl(
        x = np.random.randn(N),
        y = np.random.randn(N),
        mode = 'markers',
        marker = dict(
            line = dict(
                width = 1,
                color = 'DarkSlateGrey')
        )
    )
)

fig.show()
```

See https://plotly.com/python/reference/scattergl/ for more information and chart attribute options!

## NumPy and NumPy Convertible Arrays for Improved Performance

Improve the performance of generating Plotly figures that use a large number of data points by using NumPy arrays and other objects that Plotly can convert to NumPy arrays, such as Pandas and Polars Series.

Plotly.py uses Plotly.js for rendering, which supports typed arrays. In Plotly.py, NumPy array and NumPy-convertible arrays are base64 encoded before being passed to Plotly.js for rendering.

### Arrays and Data Types Supported

The following types of objects in Python are supported for improved performance:

- Numpy `numpy.ndarray` objects.
- Pandas Index, Pandas Series, Polars Series, and PyArrow Chunked Array objects. 
- Array objects that can be converted to `numpy.ndarray` objects. i.e., they implement `"__array__"` or `"__array_interface__"` and return a `numpy.ndarray`.

The following [array data types](https://numpy.org/devdocs/reference/arrays.scalars.html) are supported:

- float32
- float64
- int8
- uint8
- int16
- uint16
- int32
- uint32

*If the array dtype is **int64** and **uint64**, often the default dtype for arrays in NumPy when no dtype is specified, those dtypes will be changed to other types internally by Plotly.py where possible. When working with NumPY directly, you can [specify the `dtype`](https://numpy.org/doc/stable/user/basics.types.html#array-types-and-conversions-between-types) when creating `ndarray` objects.

### Unsupported Attributes

Arrays passed to attributes with the following names are not supported:

`geojson`, `layers`, and `range`.


### Example with NumPy Arrays

Here, we use NumPy arrays with a `go.Scatter3d` figure.

```python
import plotly.graph_objects as go
import numpy as np

np.random.seed(1)

# Number of data points
N = 10000

# Generate random data
x = np.random.randn(N)
y = np.random.randn(N).astype('float32')
z = np.random.randint(size=N, low=0, high=256, dtype='uint8')
c = np.random.randint(size=N, low=-10, high=10, dtype='int8')

fig = go.Figure(data=[go.Scatter3d(
    x=x,
    y=y,
    z=z,
    marker=dict(color=c),
    mode='markers',
    opacity=0.2
)])

fig.show()
```