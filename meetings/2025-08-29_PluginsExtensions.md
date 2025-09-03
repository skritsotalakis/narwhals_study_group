# Plugins & Extensions

The definitions below are how I use them within the Narwhals project, your mileage may vary.

## What might break?

If a plugin Expression diverges from the upstream Narwhals Expression API,
end users will begin to see exceptions due to unsupported arguments

```python
## Narwhals (this is the `mode` expression in Narwhals)
# def mode(keep=...): …
# nw.col().mode(keep=...)

## Narwhals Daft
# def mode(): … (this is the `mode` expression in Narwhals Daft, note the difference in signature)
# narwhals_daft.functions.mode(keep=...) # this is what Narwhals will internally call

df = nw.DataFrame(…)
df.select(nw.col().mode()) # I only interact with the upstream Expression interface; this code probably works
df.select(nw.col().mode(keep=True)) # This code works everywhere except for Daft; at Run time
```

How do we keep plugins mostly up to date with the upstream API?

## What is a Plugin?

A plugin is a new backend for Narwhals. Effectively we are providing support for the current
Narwhals API to another DataFrame/Tabular library. Daft is a plugin because we are not changing
Narwhals API, just adding another library to support.

## What is an Extension?

Extensions can be thought to *add* functionality to Narwhals.
For example if I wanted to make a suite of linear algebra related
functions to use on pandas DataFrames, I could do:


```python
# lib.py
def linalg_dot(df, x, y):
    return df[x].dot(df[y]) # let's pretend pandas didn't already support dot products

# script.py
from lib import linalg_dot

df = DataFrame(…)
linalg_dot(df, 'x', 'y')
```

If we wanted to add some convenience, we can directly
add methods to both DataFrame and Series objects.

```python
# lib.py
def linalg_dot(df, x, y):
    return df[x].dot(df[y]) # let's pretend pandas didn't already support dot products

from pandas import Series
Series.linalg_dot = linalg_dot # monkey patch the method onto the series; turn this into a bound method for automatic `self`

# script.py
df = DataFrame(…)
df['x'].linalg_dot('y') # now we can directly use our new function without needing to import it; *convenient*.
```

However, monkey patching is a bit tricky because we may accidentally overwrite internally used functions.
Thankfully, both pandas and Polars support a extensions as first class citizens.

```python
import pandas as pd
from pandas.api.extensions import register_dataframe_accessor

@register_dataframe_accessor('linalg') # all DataFrames will now have a `.linalg` namespace
class Linalg:
    def __init__(self, df):
        self.df = df

    def __call__(self):
        return self.df.mean()

    def mean(self):
        return self.df.mean()

df.linalg        # returns an instance of `Linalg(…)`
df.linalg()      # returns the result of  `Linalg(…).__call__`
df.linalg.mean() # returns the result of  `Linalg(…).__call__`
```

## Discussion

Plugins vs extensions (with these meanings and our architecture) can be typically
divided across front/back end terminology. Front end is Narwhals Expression API, backend
is the implementation for each library.

- front end (Narwhals Expression interface)
- back end (pandas support, ...)

plugin
- back ends plug into the front end
- a plugin will simply add support for a new dataframe type (backend)

extension
- adds a feature to the front end (new function; `def geom_mean`)

## Some Old Code

```python
from narwhals import register_backend

import numpy as np

class NumpyDataFrame:
    def __init__(self, native):
        self.native = native # {'a': np.array([1,2,3])

    # def mean(self):
    #     cls = type(self)
    #     return cls({k: v.mean() for k, v in self.native.items()})


register_backend('numpy', nw.DataFrame, dict, NumPyDataFrame)

nw.from_native({'a': [1,2,3]})

```

```python
class T:
    pass

t = T()
t.mean # AttributeError
```
