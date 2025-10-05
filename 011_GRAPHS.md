

# Python Data Visualization Fundamentals — updated Oct 5, 2025

Beginner‑friendly overview of plotting in Python. Covers the core stack (**matplotlib**), convenient wrappers (**pandas**), and modern interactive charts (**Plotly**). Includes layout, styling, dates, categories, subplots, dual axes, saving figures, and quick dashboards. Use with `002_SETUP.md` for environment setup.

---

**Assumptions and Conventions**
- macOS + zsh, Python 3.12+ or 3.13 in a venv.  
- Use plain matplotlib for static charts. Use Plotly for interactive charts.  
- Keep examples copy‑pasteable.

---

## Table of Contents
- [0) Install and sample data](#0-install-and-sample-data)
- [1) Matplotlib basics: figure, axes, and state](#1-matplotlib-basics-figure-axes-and-state)
- [2) Lines, scatter, bar, hist](#2-lines-scatter-bar-hist)
- [3) Subplots and shared axes](#3-subplots-and-shared-axes)
- [4) Styling: labels, ticks, grids, legends, colors](#4-styling-labels-ticks-grids-legends-colors)
- [5) Dates and times](#5-dates-and-times)
- [6) Categorical and stacked bars](#6-categorical-and-stacked-bars)
- [7) Pandas plotting shortcuts](#7-pandas-plotting-shortcuts)
- [8) Interactive charts with Plotly Express](#8-interactive-charts-with-plotly-express)
- [9) Saving, export formats, and DPI](#9-saving-export-formats-and-dpi)
- [10) Small dashboard example](#10-small-dashboard-example)
- [11) Performance tips](#11-performance-tips)
- [12) Troubleshooting](#12-troubleshooting)
- [13) Recap](#13-recap)

---

## 0) Install and sample data
**What**: Install plotting libs and create reproducible data.
```sh
pip install matplotlib pandas numpy plotly
```
Create a small CSV:
```sh
python - <<'PY'
import pandas as pd
import numpy as np
rng = pd.date_range('2025-01-01', periods=30, freq='D')
df = pd.DataFrame({
    'date': rng,
    'sales': (np.random.rand(30)*100).round(2),
    'region': np.random.choice(['AU','NZ','US'], size=30)
})
df.to_csv('sample.csv', index=False)
print('Wrote sample.csv')
PY
```

---

## 1) Matplotlib basics: figure, axes, and state
**What**: `Figure` is the canvas, `Axes` is a plot.  
**Why**: Most confusion is state vs object API.
```python
import matplotlib.pyplot as plt

# Object-oriented API (preferred)
fig, ax = plt.subplots(figsize=(6,4))
ax.plot([1,2,3,4], [10, 8, 9, 12])
ax.set_title('Basic Line')
ax.set_xlabel('x')
ax.set_ylabel('y')
plt.show()
```
Notes:
- Use `fig, ax = plt.subplots()` and call methods on `ax`.  
- Avoid global state unless quick‑and‑dirty.

---

## 2) Lines, scatter, bar, hist
```python
import matplotlib.pyplot as plt
import numpy as np

x = np.arange(0, 10, 0.5)
y = np.sin(x)
fig, axs = plt.subplots(2,2, figsize=(8,6))

# line
axs[0,0].plot(x, y)
axs[0,0].set_title('line')

# scatter
axs[0,1].scatter(x, y)
axs[0,1].set_title('scatter')

# bar
cats = ['A','B','C']
vals = [5, 3, 7]
axs[1,0].bar(cats, vals)
axs[1,0].set_title('bar')

# hist
samples = np.random.randn(1000)
axs[1,1].hist(samples, bins=30)
axs[1,1].set_title('hist')

fig.tight_layout()
plt.show()
```

---

## 3) Subplots and shared axes
```python
import matplotlib.pyplot as plt
import numpy as np

x = np.linspace(0, 4*np.pi, 400)
fig, (ax1, ax2) = plt.subplots(2, 1, sharex=True, figsize=(7,5))
ax1.plot(x, np.sin(x))
ax1.set_title('Sine')
ax2.plot(x, np.cos(x))
ax2.set_title('Cosine')
ax2.set_xlabel('x')
fig.tight_layout()
plt.show()
```
`sharex`/`sharey` keeps scales aligned. Use `gridspec_kw` for custom layouts.

---

## 4) Styling: labels, ticks, grids, legends, colors
```python
import matplotlib.pyplot as plt
import numpy as np

x = np.arange(6)
fig, ax = plt.subplots()
ax.plot(x, x**2, label='quadratic')
ax.plot(x, x**3, label='cubic')
ax.set_title('Title')
ax.set_xlabel('X label')
ax.set_ylabel('Y label')
ax.grid(True, which='both', axis='both')
ax.legend(loc='best')
# ticks
ax.set_xticks(x)
ax.set_xticklabels([f'{i}' for i in x])
fig.tight_layout(); plt.show()
```
Tip: prefer data‑ink. Reduce clutter. Label axes and units.

---

## 5) Dates and times
```python
import pandas as pd
import matplotlib.pyplot as plt

df = pd.read_csv('sample.csv', parse_dates=['date'])
df = df.sort_values('date')
fig, ax = plt.subplots(figsize=(7,4))
ax.plot(df['date'], df['sales'])
ax.set_title('Daily Sales')
ax.set_xlabel('Date')
ax.set_ylabel('Sales')
fig.autofmt_xdate()
plt.show()
```
For dense dates, use `ax.xaxis.set_major_locator` and formatters from `matplotlib.dates`.

---

## 6) Categorical and stacked bars
```python
import pandas as pd
import matplotlib.pyplot as plt

df = pd.read_csv('sample.csv', parse_dates=['date'])
piv = df.pivot_table(index='date', columns='region', values='sales', aggfunc='sum').fillna(0)
fig, ax = plt.subplots(figsize=(7,4))
ax.stackplot(piv.index, *[piv[c] for c in piv.columns], labels=list(piv.columns))
ax.legend(loc='upper left')
ax.set_title('Sales by Region (stacked)')
ax.set_xlabel('Date'); ax.set_ylabel('Sales')
fig.autofmt_xdate()
plt.show()
```
For discrete categories per period, consider `ax.bar` with `bottom=` to stack, or `barh` for horizontal stacks.

---

## 7) Pandas plotting shortcuts
```python
import pandas as pd
import matplotlib.pyplot as plt

df = pd.read_csv('sample.csv', parse_dates=['date'])
df.set_index('date', inplace=True)
ax = df['sales'].rolling(7, min_periods=1).mean().plot(figsize=(7,4), title='7‑day MA')
ax.set_xlabel('Date'); ax.set_ylabel('Sales')
plt.show()
```
Pandas uses matplotlib under the hood. Quick for EDA; switch to OO API when customizing.

---

## 8) Interactive charts with Plotly Express
**What**: Zoom, pan, tooltips, export to HTML.
```python
import pandas as pd
import plotly.express as px

df = pd.read_csv('sample.csv', parse_dates=['date'])
fig = px.line(df, x='date', y='sales', color='region', title='Interactive Sales by Region')
fig.show()         # opens a browser window
fig.write_html('interactive_sales.html')
```
Scatter with facets:
```python
fig = px.scatter(df, x='date', y='sales', color='region', facet_col='region')
fig.show()
```

---

## 9) Saving, export formats, and DPI
```python
import matplotlib.pyplot as plt
fig, ax = plt.subplots()
ax.plot([0,1,2], [0,1,0])
fig.savefig('chart.png', dpi=150)
fig.savefig('chart.pdf')
fig.savefig('chart.svg')
```
PNG/JPG for raster. SVG/PDF for vector (best for print). Control DPI for clarity in docs.

---

## 10) Small dashboard example
```python
# file: mini_dashboard.py
import pandas as pd
import matplotlib.pyplot as plt
import numpy as np

# load
df = pd.read_csv('sample.csv', parse_dates=['date'])
df = df.sort_values('date')

# derive
df['week'] = df['date'].dt.to_period('W').apply(lambda r: r.start_time)
dw = df.groupby(['week','region'])['sales'].sum().unstack(fill_value=0)

fig, axs = plt.subplots(2, 2, figsize=(10,7))

# 1) daily
axs[0,0].plot(df['date'], df['sales'])
axs[0,0].set_title('Daily')

# 2) weekly by region stacked bars
bottom = None
for col in dw.columns:
    axs[0,1].bar(dw.index, dw[col], bottom=bottom, label=col)
    bottom = (bottom if bottom is not None else 0) + dw[col]
axs[0,1].legend(); axs[0,1].set_title('Weekly by Region')

# 3) distribution
axs[1,0].hist(df['sales'], bins=20)
axs[1,0].set_title('Distribution')

# 4) rolling mean
axs[1,1].plot(df['date'], df['sales'].rolling(7, min_periods=1).mean())
axs[1,1].set_title('7‑day MA')

for ax in axs.flat:
    ax.grid(True, alpha=0.3)
fig.autofmt_xdate(); fig.tight_layout(); plt.show()
```

---

## 11) Performance tips
- Downsample or aggregate before plotting. Plot millions of points only if needed.  
- Use line transparency (`alpha`) for dense scatter.  
- Prefer vector formats for static docs; large PNGs bloat repos.  
- Cache expensive data prep; avoid recomputing per plot.

---

## 12) Troubleshooting
- **Blank chart**: you created data but never called `plt.show()` or saved the figure.  
- **Overlapping labels**: call `fig.tight_layout()` and rotate ticks.  
- **Wrong dates**: ensure `parse_dates=` and sort by datetime.  
- **Colors unreadable**: avoid custom colors with low contrast; rely on defaults first.  
- **Plotly not opening**: use `fig.write_html('out.html')` and open the file in a browser.

---

## 13) Recap
```plaintext
Load data → create fig/axes → plot primitives → style labels/ticks → handle dates/categories → subplots → save/export → go interactive with Plotly when needed
```

**Next**: Add error bars, twin axes (`ax.twinx()`), secondary y‑scales, and domain‑specific charts (candlesticks, choropleths). For heavy analytics, look at Altair/Vega‑Lite and Holoviews/Bokeh for linked Brushing and big data.