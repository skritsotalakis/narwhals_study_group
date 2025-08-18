# Narwhals Objects and Expression Overview

## Objects

terms:
- native    → an object not from Narwhals (e.g. `pandas.DataFrame`, `pyarrow.Table`)
- compliant → an object within Narwhals that follows a specific polymorphism (e.g. expressions have methods like `.mean`, `min`, `max`, `mean`. DataFrames/LazyFrames have `.select`, `.with_columns`; see `narwhals._compliant.Expr` and `narwhals._compliant.DataFrame`)
    - we wrap a native object in a compliant object so that we can use its uniform interface

The native object (like `pandas.DataFrame`) will have its own attributes/methods that
we do not control.

```python
from pandas import DataFrame
from pyarrow import table

pd_df = DataFrame(data := {'a': [1,2,3], 'b': [4,5,6]})
print(pd_df.assign(c=10))

print('=' * 40)

pa_table = table(data)
print(pa_table.append_column('c', [[10, 10, 10]]))
```

To create a uniform interface, we need to smooth across `pandas.DataFrame.assign` and `pyarrow.Table.append_column`.
We do this at the compliant level

```python
from pandas import DataFrame
from pyarrow import table
import narwhals as nw

pd_df = DataFrame(data := {'a': [1,2,3], 'b': [4,5,6]})
pa_table = table(data)

nw_pd_df = nw.from_native(pd_df)
nw_pa_table = nw.from_native(pa_table)

print(
    # ALWAYS returns a narwhals.dataframe.DataFrame (for tabular input)
    #   Creates a uniform interface for end users to write their code with
    type(nw_pd_df),
    type(nw_pd_df),
    '',

    # the compliant level: narwhals.DataFrame → narwhals.{backend}.*DataFrame
    type(nw_pd_df._compliant_frame),    # narwhals._pandas_like.dataframe.PandasLikeDataFrame
    type(nw_pa_table._compliant_frame), # narwhals._arrow.dataframe.ArrowDataFrame
    # these `_compliant_frame`s share the same interface as narwhals.DataFrame
    #  but have the backend-specific code to carry out the actual work (e.g. adding a column)

    sep='\n',
)
```

### Object Nesting

```
narwhals.DataFrame → narwhals.{backend}.*DataFrame → {backend}.DataFrame
Interface          → Compliant                     → Native
```

Top-down accession for each of these levels

```python
import narwhals as nw
import pandas as pd
import pyarrow as pa

# pandas
df = pd.DataFrame({'a': [1,2,3], 'b':[4,5,6]})
nw_df = nw.from_native(df)
print(type(nw_df))
print(nw_df._compliant_frame)
print(nw_df._compliant_frame._native_frame)

print('=' * 40)

# arrow
table = pa.table({'a': [1,2,3], 'b': [4,5,6]})
nw_df = nw.from_native(table)
print(type(nw_df))
print(nw_df._compliant_frame)
print(nw_df._compliant_frame._native_frame)
```

## Expressions

### Eager vs Lazy

Thunks/higher order functions for delayed evaluation

```python
from functools import partial
from time import sleep

# def f(x):
#     sleep(3)
#     return x + 5

def f(x):
    def _f():
        return x + 5
    return _f

# print(f(10))

# above → library level
# if __name__ == '__main__':
# below → scripting level

# call_f = lambda: f(10)
# print(call_f)

call_f = f(10)
print(call_f)
print(call_f())
```

Instead of wrapping the function in the script level, we can also return a function
at the library level to delay evaluation (and pass further arguments out-of-band)

```python
def col(*names):
    def _col(df):
        return df[list(names)]
    return _col

import pandas as pd

df = pd.DataFrame({'a': [1,2,3], 'b': [4,5,6]})

expr = col('a')
print(expr) # <function col.<locals>._col at …>
print(expr(df))
#    a
# 0  1
# 1  2
# 2  3
```

However passing functions is *opaque*, meaning that we do not know what the function does
at runtime, just that it is a function. So in addition to passing functions around
we also include metadata


```python
from narwhals import col

from dataclasses import dataclass

@dataclass
class Expr:
    func: callable
    root_name: …
    arity: …

# narwhals.Expr is not very far from this
# narwhals.expr.py
# class Expr:
#     def __init__(self, to_compliant_expr: _ToCompliant, metadata: ExprMetadata) -> None:
#     to_compliant_expr is a function. metadata describes properties of that function

```
