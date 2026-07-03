# Data Analysis with Python — NumPy, Pandas & Seaborn Notes

Course: [krishnaik06/Complete-Python-Bootcamp](https://github.com/krishnaik06/Complete-Python-Bootcamp/tree/main/10-Data%20Analysis%20With%20Python)

After OOPs and exception handling, this module was the first time the course actually felt like "data science" rather than "Python fundamentals." This is the toolkit basically every data analysis or ML pipeline is built on top of, so I went slower here than usual.

## NumPy — the foundation everything else sits on

Before pandas, before any ML library, there's NumPy. Core idea: instead of Python lists (slow, flexible, general-purpose), NumPy gives you **arrays** — fixed-type, fast, and built for math on entire collections of numbers at once instead of looping.

```python
import numpy as np

arr = np.array([1, 2, 3, 4, 5])
matrix = np.array([[1, 2, 3], [4, 5, 6]])

print(arr.shape)      # (5,)
print(matrix.shape)   # (2, 3)
```

Things that stood out:
- **Vectorized operations** — `arr * 2` multiplies every element instantly, no `for` loop needed. This is the whole reason NumPy is fast.
- **`np.zeros()`, `np.ones()`, `np.arange()`, `np.linspace()`** — quick ways to generate arrays without typing every value.
- **Broadcasting** — NumPy can apply operations between arrays of different shapes automatically (e.g. adding a single number to every element of a 2D array) without me writing extra code for it.
- **Indexing/slicing** works like Python lists but extends to multiple dimensions: `matrix[0, 1]`, `matrix[:, 1]` (whole column).

## Pandas — DataFrame and Series

If NumPy is the engine, pandas is the dashboard. Two core structures:
- **Series** — a single labeled column of data (basically a 1D array with an index attached).
- **DataFrame** — a full table: rows and columns, like an Excel sheet inside Python.

```python
import pandas as pd

data = {"name": ["Dheer", "Alex", "Sam"], "score": [88, 92, 79]}
df = pd.DataFrame(data)

print(df.head())      # first 5 rows
print(df.info())      # column types, nulls, memory
print(df.describe())  # quick stats: mean, std, min, max
```

`.head()`, `.info()`, and `.describe()` are the three commands I now run on literally every new dataset before touching anything else — they tell you what you're working with in about 5 seconds.

## Data manipulation with pandas

This is where the actual "cleaning and shaping data" work happens:

```python
# Filtering rows
high_scorers = df[df["score"] > 80]

# Adding a new column
df["passed"] = df["score"] >= 80

# Grouping and aggregating
df.groupby("passed")["score"].mean()

# Handling missing values
df.fillna(0)
df.dropna()

# Sorting
df.sort_values("score", ascending=False)
```

The pattern that clicked: pandas operations are mostly **chainable** — filter, then group, then aggregate, all in one readable line instead of writing multiple loops. Coming from plain Python, this feels like a shortcut once it clicks.

Also learned `.apply()` — running a custom function across a column or row when there isn't a built-in method for what you need:

```python
df["grade"] = df["score"].apply(lambda x: "A" if x >= 90 else "B" if x >= 80 else "C")
```

## Reading data from different sources

Real data almost never starts as a hand-typed dictionary — it comes from files. Pandas makes this almost annoyingly simple:

```python
df_csv = pd.read_csv("data.csv")
df_excel = pd.read_excel("sample_data.xlsx")
```

Small things that mattered:
- `pd.read_csv()` has a ton of optional parameters (`sep`, `header`, `usecols`, `na_values`) for messy real-world files that don't come in perfectly clean.
- Excel files need `openpyxl` installed under the hood for `.xlsx` — pandas just handles it once the library's there.
- Once loaded, a CSV and an Excel file behave identically as a DataFrame — the loading step is the only thing that changes based on source.

## Data visualization with Seaborn

Numbers in a table only tell you so much — Seaborn is where the data actually starts talking. It sits on top of Matplotlib but with way less boilerplate for statistical plots.

```python
import seaborn as sns
import matplotlib.pyplot as plt

sns.histplot(df["score"], kde=True)
plt.show()

sns.scatterplot(x="score", y="passed", data=df)
plt.show()

sns.heatmap(df.corr(), annot=True, cmap="coolwarm")
plt.show()
```

Plots I now reach for depending on the question:
- **`histplot`** — distribution of a single column (is it skewed? normal? clustered?).
- **`scatterplot`** — relationship between two numeric columns.
- **`boxplot`** — spotting outliers at a glance.
- **`heatmap`** on `.corr()` — which columns move together, instantly visual instead of squinting at a correlation table.

## My takeaway from this module

Python fundamentals taught me how to write logic. This module taught me how to actually look at data before making any decisions about it — and that "looking" step (`.head()`, `.describe()`, a quick histogram) is apparently what separates a real analysis from guessing.

---

## NumPy practice assignments — my solutions

Went through 10 rounds of NumPy drills, two questions each. Grouping them here by what they actually taught me rather than just dumping code.

**1. Array creation & manipulation** — replacing a whole column, replacing the diagonal
```python
import numpy as np

array = np.random.randint(1, 21, size=(5, 5))
array[:, 2] = 1  # replace entire 3rd column with 1

array2 = np.arange(1, 17).reshape((4, 4))
np.fill_diagonal(array2, 0)  # replace diagonal with 0
```
`np.fill_diagonal()` was new to me — much cleaner than looping and setting `array[i][i] = 0` manually.

**2. Indexing & slicing** — sub-arrays and border elements
```python
array = np.arange(1, 37).reshape((6, 6))
sub_array = array[2:5, 1:4]  # rows 3-5, cols 2-4

array = np.random.randint(1, 21, size=(5, 5))
border = np.concatenate((array[0, :], array[-1, :], array[1:-1, 0], array[1:-1, -1]))
```
The border-elements one made me actually think in terms of "top row + bottom row + left column (excluding corners already counted) + right column" instead of just guessing at slice syntax.

**3. Element-wise operations & row/column sums**
```python
array1 = np.random.randint(1, 11, size=(3, 4))
array2 = np.random.randint(1, 11, size=(3, 4))
addition, subtraction = array1 + array2, array1 - array2
multiplication, division = array1 * array2, array1 / array2

matrix = np.arange(1, 17).reshape((4, 4))
row_sum = np.sum(matrix, axis=1)
col_sum = np.sum(matrix, axis=0)
```

**4. Statistics & normalization**
```python
array = np.random.randint(1, 21, size=(5, 5))
mean, median = np.mean(array), np.median(array)
std_dev, variance = np.std(array), np.var(array)

# Normalize to mean 0, std 1
array2 = np.arange(1, 10).reshape((3, 3))
normalized = (array2 - np.mean(array2)) / np.std(array2)
```
Normalization is literally just "subtract the mean, divide by the standard deviation" — I'd seen the term thrown around a lot before this and never actually knew it was this simple.

**5. Broadcasting**
```python
array = np.random.randint(1, 11, size=(3, 3))
row_array = np.random.randint(1, 11, size=(3,))
result = array + row_array  # broadcasts across each row

array2 = np.random.randint(1, 11, size=(4, 4))
col_array = np.random.randint(1, 11, size=(4,))
result2 = array2 - col_array[:, np.newaxis]  # broadcasts down each column
```
The `[:, np.newaxis]` trick is the key difference between "broadcast across rows" and "broadcast down columns" — without it NumPy tries to match shapes the wrong way and errors out.

**6. Linear algebra**
```python
matrix = np.random.randint(1, 11, size=(3, 3))
determinant = np.linalg.det(matrix)
inverse = np.linalg.inv(matrix)
eigenvalues = np.linalg.eigvals(matrix)

a1 = np.random.randint(1, 11, size=(2, 3))
a2 = np.random.randint(1, 11, size=(3, 2))
product = np.dot(a1, a2)  # matrix multiplication
```
Good reminder that `*` on two matrices is element-wise, but `np.dot()` (or `@`) is actual matrix multiplication — mixing those two up gives you a wrong answer with no error, which is worse than a crash.

**7. Reshaping & flattening**
```python
array = np.arange(1, 10).reshape((3, 3))
flat = array.reshape((1, 9))
back_to_col = flat.reshape((9, 1))

array2 = np.random.randint(1, 21, size=(5, 5))
flattened = array2.flatten()
reshaped = flattened.reshape((5, 5))
```

**8. Fancy & boolean indexing**
```python
array = np.random.randint(1, 21, size=(5, 5))
corners = array[[0, 0, -1, -1], [0, -1, 0, -1]]  # 4 corner elements in one line

array2 = np.random.randint(1, 21, size=(4, 4))
array2[array2 > 10] = 10  # cap every value above 10
```
Boolean indexing (`array[array > 10] = 10`) is one of those lines that felt like magic the first time — no loop, no `if`, just a condition directly inside the brackets.

**9. Structured arrays**
```python
dtype = [('name', 'U10'), ('age', 'i4'), ('weight', 'f4')]
data = np.array([('Alice', 25, 55.5), ('Bob', 30, 85.3), ('Charlie', 20, 65.2)], dtype=dtype)
sorted_data = np.sort(data, order='age')
```
First time seeing a NumPy array hold mixed types (string, int, float) in named fields — basically a lightweight version of what a DataFrame does, minus the pandas overhead.

**10. Masked arrays**
```python
import numpy.ma as ma

array = np.random.randint(1, 21, size=(4, 4))
masked = ma.masked_greater(array, 10)
sum_unmasked = masked.sum()  # ignores masked values automatically

array2 = np.random.randint(1, 21, size=(3, 3))
masked2 = ma.masked_array(array2, mask=np.eye(3, dtype=bool))
masked2 = masked2.filled(masked2.mean())  # replace masked with mean
```
Masked arrays were completely new to me — instead of deleting or filtering values you don't want, you "hide" them from calculations while keeping the array's shape intact. Feels like the NumPy equivalent of `NaN` handling in pandas.

## Pandas practice assignments — my solutions

**1. DataFrame creation & indexing**
```python
import pandas as pd
import numpy as np

df = pd.DataFrame(np.random.randint(1, 100, size=(6, 4)), columns=['A', 'B', 'C', 'D'])
df.set_index('A', inplace=True)

df2 = pd.DataFrame(np.random.randint(1, 100, size=(3, 3)), columns=['A', 'B', 'C'], index=['X', 'Y', 'Z'])
element = df2.at['Y', 'B']  # fast single-value lookup
```
`.at[]` for a single cell lookup is faster than `.loc[]` for this — worth using when I only need one value, not a slice.

**2. DataFrame operations**
```python
df = pd.DataFrame(np.random.randint(1, 100, size=(5, 3)), columns=['A', 'B', 'C'])
df['D'] = df['A'] * df['B']  # new column from existing ones

df2 = pd.DataFrame(np.random.randint(1, 100, size=(4, 3)), columns=['A', 'B', 'C'])
row_sum = df2.sum(axis=1)
col_sum = df2.sum(axis=0)
```

**3. Data cleaning**
```python
df = pd.DataFrame(np.random.randint(1, 100, size=(5, 3)), columns=['A', 'B', 'C'])
df.iloc[0, 1] = np.nan
df.fillna(df.mean(), inplace=True)  # fill NaN with column mean

df2 = pd.DataFrame(np.random.randint(1, 100, size=(6, 4)), columns=['A', 'B', 'C', 'D'])
df2.iloc[1, 2] = np.nan
df2.dropna(inplace=True)  # drop any row with a NaN
```

**4. Grouping & aggregation**
```python
df = pd.DataFrame({'Category': np.random.choice(['A', 'B', 'C'], size=10),
                    'Value': np.random.randint(1, 100, size=10)})
grouped = df.groupby('Category')['Value'].agg(['sum', 'mean'])

df2 = pd.DataFrame({'Product': np.random.choice(['Prod1', 'Prod2', 'Prod3'], size=10),
                     'Category': np.random.choice(['A', 'B', 'C'], size=10),
                     'Sales': np.random.randint(1, 100, size=10)})
total_sales = df2.groupby('Category')['Sales'].sum()
```

**5. Merging & concatenating**
```python
df1 = pd.DataFrame({'Key': ['A', 'B', 'C', 'D'], 'Value1': np.random.randint(1, 100, size=4)})
df2 = pd.DataFrame({'Key': ['A', 'B', 'C', 'E'], 'Value2': np.random.randint(1, 100, size=4)})
merged = pd.merge(df1, df2, on='Key')  # only keeps matching keys (A, B, C)

concat_rows = pd.concat([df1, df2], axis=0)     # stack on top of each other
concat_columns = pd.concat([df1, df2], axis=1)  # side by side
```
`merge()` only kept rows where the key existed in *both* DataFrames — D and E dropped out. That's an inner join by default, which caught me off guard until I checked the `how=` parameter exists for outer/left/right joins too.

**6. Time series**
```python
date_rng = pd.date_range(start='2022-01-01', end='2022-12-31', freq='D')
df = pd.DataFrame(date_rng, columns=['date'])
df['data'] = np.random.randint(0, 100, size=len(date_rng))
df.set_index('date', inplace=True)

monthly_mean = df.resample('M').mean()   # collapse daily data into monthly averages
rolling_mean = df.rolling(window=7).mean()  # smoothed 7-day moving average
```
`.resample()` vs `.rolling()` was the key distinction here: resample changes the actual frequency of the data (daily → monthly), rolling keeps the same frequency but smooths it with a moving window.

**7. MultiIndex**
```python
arrays = [['A', 'A', 'B', 'B'], ['one', 'two', 'one', 'two']]
index = pd.MultiIndex.from_arrays(arrays, names=('Category', 'SubCategory'))
df = pd.DataFrame(np.random.randint(1, 100, size=(4, 3)), index=index, columns=['Value1', 'Value2', 'Value3'])

df.loc['A']              # everything under Category A
df.loc[('B', 'two')]     # drill into a specific sub-category
```

**8. Pivot tables**
```python
date_rng = pd.date_range(start='2022-01-01', end='2022-01-10', freq='D')
df = pd.DataFrame({'Date': np.random.choice(date_rng, size=20),
                    'Category': np.random.choice(['A', 'B', 'C'], size=20),
                    'Value': np.random.randint(1, 100, size=20)})

pivot = df.pivot_table(values='Value', index='Date', columns='Category', aggfunc='sum')
```
Pivot tables clicked once I saw it as "groupby, but reshaped into a spreadsheet-style grid" — same aggregation logic, different output shape.

**9. Applying functions**
```python
df = pd.DataFrame(np.random.randint(1, 100, size=(5, 3)), columns=['A', 'B', 'C'])
doubled = df.applymap(lambda x: x * 2)  # applies to every single cell

df2 = pd.DataFrame(np.random.randint(1, 100, size=(6, 3)), columns=['A', 'B', 'C'])
df2['Sum'] = df2.apply(lambda row: row.sum(), axis=1)  # applies across each row
```
`.applymap()` (cell-by-cell) vs `.apply(axis=1)` (row-by-row) — easy to reach for the wrong one if you're not paying attention to what you're actually trying to transform.

**10. Text data**
```python
text_data = pd.Series(['apple', 'banana', 'cherry', 'date', 'elderberry'])
uppercase = text_data.str.upper()
first_three = text_data.str[:3]
```
The `.str` accessor is what unlocks normal string methods on an entire Series at once — without it you'd be back to writing a loop.

## What tripped me up across both sets

Mixing up `axis=0` vs `axis=1` was the recurring mistake in both NumPy and pandas — I finally locked it in as "0 moves down the rows (collapses rows), 1 moves across the columns (collapses columns)." Also learned the hard way that `.applymap()` and `.apply()` are not interchangeable, and that `pd.merge()` defaults to an inner join, which silently drops non-matching rows if you're not paying attention.
