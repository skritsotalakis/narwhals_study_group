# How does narwhals.from_native work?

**fact: narwhals uses a subset of the Polars API (e.g. narwhals.DataFrame.with_columns)**

## Object Construction

observe:
```
from_native(pd.DataFrame(…)) → nw.DataFrame;
nw.DataFrame.with_columns(…)
nw.DataFrame.to_native()
```

problem: `pandas.DataFrame.with_columns` does not exist; so how does this work?
```
nw.DataFrame.with_columns(…) → ... → pandas.DataFrame.assign(…) (and some other code)
```

But we deal with much more than just `pandas.DataFrames` so instead of a straight forward 2 step process,
we have a 3rd level that is opaque to the user.
`nw.DataFrame.with_columns(…)` → `nw._pandas_like.PandasLikeDataFrame` → pandas.DataFrame.assign(…) (and some other code)

implying that
`nw.DataFrame` is an interface (defining publicly available methods)
`nw.DataFrame` delegates operations to its compliant counterpart (a class that has the same methods as the `nw.DataFrame;` e.g. `narwhals._pandas_like.PandasLikeDataFrame)`
the compliant level retains the backend specific code to make with_columns (and other methods) work on a given DataFrame

For example, `PandasLikeDataFrame.with_columns` operates on an underlying `pandas.DataFrame` object creation

So, the hierarchy (from the native object up) looks like:
1. `pandas.DataFrame` (user created)
2. → `nw._pandas_like.PandasLikeDataFrame` (internally managed; completely abstracted away from user)
3. → `nw.DataFrame` (exposed to the user)

In order to do this, we first determine what was passed in (`pandas`, `pd.DataFrame`)
Then, grab the correct backend namespace (the namespace houses the compliant abstractions: `PandasLikeDataFrame`, `PandasLikeSeries`, and more)

For an applied example we have the following parallels
- `pandas` → `narwhals._pandas_like.namespace`
- `pd.DataFrame` → `narwhals._pandas_like.PandasLikeDataFrame`

We wrap everything up in a singular `nw.DataFrame`
- `nw.DataFrame._compliant_frame` → `nw.PandasLikeDataFrame`
- `nw.DataFrame._compliant_frame._native_frame` → `pd.DataFrame`

Once we get a `nw.DataFrame` back from `nw.from_native(pd.DataFrame),` we can now use its methods to make changes to the underlying `pd.DataFrame`

**method calls:**
when one calls a `nw.DataFrame` method we see a top down stack formed:
`nw.DataFrame` → `nw._pandas_like.PandasLikeDataFrame` → `pandas.DataFrame`

**method call results**
Then the results are passed back up the stack
`nw.DataFrame` ← `nw._pandas_like.PandasLikeDataFrame` ← `pandas.DataFrame`
