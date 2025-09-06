# What is a Namespace

A namespace
1. provides convenient access the the backends classes (LikeDataFrame/LikeSeries/LikeExpr)
2. defines the floating functions required by a backend

This makes it easy for us to write functions when we need to interact with *more*
than just the DataFrame/Expressions we have on hand.

```python
import narwhals as nw
from narwhals._pandas_like.namespace import PandasLikeNamespace
from narwhals._utils import Version

# is predicated on an Implementation & Version (2 enums)
ns = PandasLikeNamespace(nw.Implementation.PANDAS, version=Version.MAIN)

##  the above is behaviorally equivalent to:
# import pandas as pd
# from narwhals._namespace import Namespace
# ns = Namespace.from_backend(pd)._compliant_namespace

print(
    f'{ns                      = }',

    # convenience (polymorphic accession to backend types)
    f'{ns._dataframe           = }',
    f'{ns._series              = }',
    f'{ns._expr                = }',

    # access to top-level functions
    f'{ns.mean_horizontal      = }',
    sep='\n',
)
```

But how do we know all of the attributes/methods we need when writing our own `backend.…Namespace`? That’s the role of `typing`!

```python
from narwhals._compliant.namespace import CompliantNamespace

print(dir(CompliantNamespace))
```

If you poke around you `from narwhals._compliant.namespace` then you
may notice that there are different namespaces available for different
kinds of backends (e.g. pandas which has no "lazy" concept, duckdb which only has "lazy")

```python
from narwhals._compliant.namespace import CompliantNamespace, EagerNamespace, LazyNamespace

# note that Eager/LazyNamespace inherit from CompliantNamespace (directly or indirectly)
all_namespace_should_have = dir(CompliantNamespace)
eager_namespace_should_have = dir(EagerNamespace)
lazy_namespace_should_have = dir(LazyNamespace)

print(
    f'{set(eager_namespace_should_have) - set(all_namespace_should_have) = }',
    # {'_concat_horizontal', 'from_native', '_concat_diagonal', '_backend_version', '_dataframe', '_series', '_concat_vertical', 'from_numpy'}

    f'{set(lazy_namespace_should_have) - set(all_namespace_should_have)  = }',
    # {'_backend_version', '_lazyframe', 'from_native'}

    sep='\n'
)
```

## What does a Namespace Provide to Our Code?

When writing generic functions, we may need to use more code than just the methods
exposed on the passed in DataFrame.

```python
import narwhals as nw
import pandas as pd
import polars as pl

def concat(*frames: nw.DataFrame):
    native_frames = [df._native_frame for df in frames]
    if all(isinstance(df, pd.DataFrame) for df in native_frames):
        result =  pd.concat(frames, axis=1)
    elif all(isinstance(df, pl.DataFrame) for df in native_frames):
        result = pl.concat(frames, how="horizontal")
    return nw.DataFrame(result)
```

but if I need lots of these kinds of functions (mean_horizontal, concat, col, etc.)
am I really going to maintain this long line of if/elif separately for EVERY function?

No way.

```python
import narwhals as nw
import pandas as pd
import polars as pl
from functools import partial

class PandasNamespace:
    concat = partial(pd.concat, axis=1)

class PolarsNamespace:
    concat = partial(pl.concat, how='horizontal')

NAMESPACE_MAPPER = { # this mimics Narwhals capability to grab a namespace when given an native object
    pd.DataFrame: PandasNamespace,
    pl.DataFrame: PolarsNamespace,
}

def concat(*frames: nw.DataFrame):
    native_frames = [df.to_native() for df in frames]
    ns = NAMESPACE_MAPPER[type(native_frames[0])]
    native_result = ns.concat(native_frames) # we can use all functions we wrote within a namespace!
    return nw.from_native(native_result)

dfs = [
    pd.DataFrame({'a': [1,2,3]}),
    pd.DataFrame({'b': [4,5,6]})
]
nw_dfs = [nw.from_native(df) for df in dfs]
print(concat(*nw_dfs))

dfs = [
    pl.DataFrame({'a': [1,2,3]}),
    pl.DataFrame({'b': [4,5,6]})
]
nw_dfs = [nw.from_native(df) for df in dfs]
print(concat(*nw_dfs))

```

## Always Keep in Mind

The 3 levels of our abstractions: backend → adapter → Generic API level
Namespaces are all about creating a uniform adapter layer for many top-level
functions (e.g. col, concat, mean_horizontal) so make it more convnient for
the API level to find and use these adaptations.

| backend                                          | adapter                                          | API level           |
| ------------------------------------------------ | ------------------------------------------------ |-------------------- |
|pandas.DataFrame                                  | narwhals._pandas_like.PandasLikeDataFrame        | narwhals.DataFrame  |
|misc functions/composition of functions in pandas | narwhals._pandas_like.namespace.PandasNamespace  | narwhals            |

