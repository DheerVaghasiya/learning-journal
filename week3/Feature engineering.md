# Feature Engineering

Feature engineering is the process of preparing raw data so that a machine learning model can actually make sense of it. Real-world data is messy — values go missing, some classes barely show up, extreme values throw off calculations, and categorical columns like "city" or "color" can't be fed into a model as-is. This note walks through the core techniques used to fix each of these problems: handling missing values, handling imbalanced datasets (manual resampling and SMOTE), handling outliers, and the three main ways to encode categorical data.

---

## 1. Handling Missing Values

Missing data shows up in almost every real dataset, and before deciding how to handle it, it helps to understand *why* it's missing — because that changes what the right fix is. There are three recognized mechanisms:

**MCAR (Missing Completely at Random)** — the missing values have no relationship with anything else in the data. They're just randomly scattered, with no underlying pattern. Example: a few survey responses go missing purely due to a technical glitch, unrelated to who the respondent is.

**MAR (Missing at Random)** — the missingness is related to some *other observed* column, but not to the missing value itself. Example: younger respondents are less likely to report their income, but whether someone reports it or not has nothing to do with the actual income amount.

**MNAR (Missing Not at Random)** — the missingness is related to the value that's actually missing. Example: people less satisfied with their job are the ones who refuse to report income — here the missing values are tied to something that can't be directly observed.

Knowing which type you're dealing with matters because MCAR is usually safe to impute or drop, while MNAR needs more caution since the missingness itself is informative.

```python
import seaborn as sns
df = sns.load_dataset('titanic')
df.isnull().sum()
```

### Option 1: Dropping missing data
```python
df.dropna().shape          # drop rows with any NaN
df.dropna(axis=1)          # drop columns with any NaN
```
This is the simplest fix, but it throws away data. It only makes sense when the percentage of missing values is small — dropping large chunks of a dataset can seriously hurt model performance.

### Option 2: Imputation

**Mean imputation** — replaces missing values with the column average. Works well only when the data is roughly normally distributed, since the mean is sensitive to skew and outliers.
```python
df['Age_mean'] = df['age'].fillna(df['age'].mean())
```

**Median imputation** — the safer choice when the column has outliers or a skewed distribution, since the median isn't dragged around by extreme values.
```python
df['age_median'] = df['age'].fillna(df['age'].median())
```

**Mode imputation** — used for categorical columns, since mean/median don't apply. Replaces missing values with the most frequently occurring category.
```python
mode_value = df[df['embarked'].notna()]['embarked'].mode()[0]
df['embarked_mode'] = df['embarked'].fillna(mode_value)
```

**Quick rule of thumb:** numerical + normal distribution → mean. Numerical + skewed/has outliers → median. Categorical → mode. Always worth plotting the distribution (a histogram works fine) before picking mean vs. median, since choosing wrong can quietly bias the model.

---

## 2. Handling Imbalanced Dataset (Up/Down Sampling)

An imbalanced dataset is one where one class massively outnumbers another — for example, 900 samples of class 0 and only 100 of class 1. This is a common problem in things like fraud detection or disease prediction, where the "interesting" class is naturally rare. Left untreated, models tend to just predict the majority class every time and still get high accuracy, while being useless at catching the minority class.

```python
class_0 = pd.DataFrame({'feature_1': np.random.normal(0,1,900), 'feature_2': np.random.normal(0,1,900), 'target':[0]*900})
class_1 = pd.DataFrame({'feature_1': np.random.normal(2,1,100), 'feature_2': np.random.normal(2,1,100), 'target':[1]*100})
df = pd.concat([class_0, class_1]).reset_index(drop=True)
```

### Up-sampling (grow the minority class to match the majority)
```python
from sklearn.utils import resample
df_minority_upsampled = resample(df_minority, replace=True, n_samples=len(df_majority), random_state=42)
df_upsampled = pd.concat([df_majority, df_minority_upsampled])
```
`replace=True` means minority rows are duplicated (sampled with replacement) until the class sizes match — 900 vs 900.

### Down-sampling (shrink the majority class to match the minority)
Same `resample()` function, but flipped: it's applied to the *majority* class, with `replace=False` (no duplicates needed when shrinking) and `n_samples=len(df_minority)`:
```python
df_majority_downsampled = resample(df_majority, replace=False, n_samples=len(df_minority), random_state=42)
df_downsampled = pd.concat([df_majority_downsampled, df_minority])
```
This throws away majority-class rows instead of duplicating minority ones — the two functions are mirror images of each other, just pointed at different classes.

**When to use which:** up-sampling is preferred when the dataset is small overall, since down-sampling would throw away data that's already scarce. Down-sampling can make sense when there's plenty of majority-class data to spare and training time/memory is a concern.

**Limitation to know:** both of these techniques just duplicate or delete existing rows — they don't create anything new. Up-sampling in particular can cause overfitting, since the model ends up seeing the exact same minority rows multiple times.

---

## 3. Handling Imbalanced Dataset Using SMOTE

SMOTE (Synthetic Minority Oversampling Technique) solves the exact limitation above. Instead of duplicating existing minority rows, it generates brand-new *synthetic* data points by interpolating between existing minority-class neighbors — picking a minority point, finding its nearest minority neighbors, and creating new points along the line connecting them.

```python
from sklearn.datasets import make_classification
X, y = make_classification(n_samples=1000, n_redundant=0, n_features=2,
                            n_clusters_per_class=1, weights=[0.90], random_state=12)
```

The imbalance can be visualized with a scatter plot colored by target before applying SMOTE. Then:

```python
from imblearn.over_sampling import SMOTE
oversample = SMOTE()
X, y = oversample.fit_resample(final_df[['f1','f2']], final_df['target'])
```

Before: 900 vs 100. After SMOTE: 900 vs 900 — but unlike plain up-sampling, the new 900 minority points aren't copies of the original 100. Plotting the result shows the minority class filling in the space *around* the original cluster rather than stacking repeatedly on the same points.

**When to use it:** SMOTE is generally preferred over plain up-sampling because it reduces overfitting — the model learns from a denser, more varied minority region instead of memorizing the same 100 points repeated 9 times over. It needs the `imblearn` library (`pip install imblearn`).

---

## 4. Handling Outliers Using Python

Outliers are extreme values that sit far outside the normal range of the data — they can badly skew averages, distort model training, and hide real patterns if left unchecked.

The standard method for detecting them is the **5-number summary**: Minimum, Q1 (25th percentile), Median, Q3 (75th percentile), Maximum — combined with the **IQR (Interquartile Range)**.

```python
import numpy as np
lst_marks = [45,32,56,75,89,54,32,89,90,87,67,54,45,98,99,67,74]
minimum, Q1, median, Q3, maximum = np.quantile(lst_marks, [0, 0.25, 0.50, 0.75, 1.0])

IQR = Q3 - Q1
lower_fence = Q1 - 1.5 * IQR
higher_fence = Q3 + 1.5 * IQR
```

Any value below `lower_fence` or above `higher_fence` is flagged as an outlier. The multiplier `1.5` is a widely used statistical convention, not a hard rule — it can be adjusted for stricter or looser outlier detection depending on the use case.

Boxplots are the standard way to visualize this — the "box" covers Q1 to Q3, the line inside is the median, and the "whiskers" extend to the fences, with anything beyond plotted as individual dots:
```python
import seaborn as sns
sns.boxplot(lst_marks)
```

Adding clearly extreme values to a list (like `-100, -200, 150, 170, 180`) and re-plotting shows those points isolated well outside the whiskers — a useful way to see the fence logic in action rather than just reading the formula.

**What to do once outliers are found:** common options are removing them (if they're clearly data-entry errors), capping them at the fence values, or transforming the data (e.g. log transform) to reduce their influence — the right choice depends on whether the outliers are genuine extreme cases or mistakes.

---

## 5. Data Encoding — Nominal / One-Hot Encoding (OHE)

Machine learning models work with numbers, not text, so categorical columns need to be converted into numeric form before they can be used. There are three main ways to do this, and which one to use depends on whether the categories have a natural order.

**One-Hot Encoding (OHE)** is used when categories have **no inherent order** — like colors, city names, or product types. Each category becomes its own binary column (1 if that row belongs to that category, 0 otherwise):
- Red → [1, 0, 0]
- Green → [0, 1, 0]
- Blue → [0, 0, 1]

```python
from sklearn.preprocessing import OneHotEncoder
df = pd.DataFrame({'color': ['red','blue','green','green','red','blue']})

encoder = OneHotEncoder()
encoded = encoder.fit_transform(df[['color']]).toarray()
encoder_df = pd.DataFrame(encoded, columns=encoder.get_feature_names_out())
pd.concat([df, encoder_df], axis=1)
```

To transform a new, unseen value using the same fitted encoder:
```python
encoder.transform([['blue']]).toarray()
```

**Downside to know:** OHE creates one new column per category, so it can blow up the number of features when a column has many unique values (e.g. hundreds of cities) — this is called the "curse of dimensionality," and it's the reason other encoding methods exist for high-cardinality columns.

---

## 6. Label And Ordinal Encoding

### Label Encoding
Assigns each category a unique integer with no notion of order — typically alphabetical. Best suited for the *target* variable in classification problems, or for categories where order genuinely doesn't matter.
```python
from sklearn.preprocessing import LabelEncoder
lbl_encoder = LabelEncoder()
lbl_encoder.fit_transform(df['color'])       # note: single brackets — LabelEncoder expects a 1D input, not a DataFrame
lbl_encoder.transform(['red'])
```

**Caution:** using Label Encoding on an *input feature* can accidentally introduce a fake ranking that doesn't exist (e.g. blue=0, green=1, red=2 implies red > green > blue, which isn't true for colors), so it's generally safer for target variables than feature columns.

### Ordinal Encoding
Used when categories *do* have a genuine, real-world order — like small/medium/large, or low/medium/high. The order is explicitly defined rather than left to alphabetical default:
```python
from sklearn.preprocessing import OrdinalEncoder
df = pd.DataFrame({'size': ['small','medium','large','medium','small','large']})

encoder = OrdinalEncoder(categories=[['small','medium','large']])
encoder.fit_transform(df[['size']])
```

**Why this matters:** if Label Encoding were used on "size" instead, it might alphabetically assign large=0, medium=1, small=2 — completely wrong order for a model that should understand small < medium < large. Ordinal Encoding lets the order be defined manually via the `categories` parameter, which is the key difference from Label Encoding.

---

## 7. Target Guided Ordinal Encoding

This technique encodes a category based on its **relationship with the target variable**, rather than alphabetical order or a manually defined rank. It's especially useful for categorical columns with a large number of unique values (high cardinality), where OHE would create too many columns and plain ordinal encoding has no natural order to rely on.

```python
df = pd.DataFrame({
    'city': ['New York','London','Paris','Tokyo','New York','Paris'],
    'price': [200, 150, 300, 250, 180, 320]
})

mean_price = df.groupby('city')['price'].mean().to_dict()
df['city_encoded'] = df['city'].map(mean_price)
```

Result: `London → 150.0, New York → 190.0, Tokyo → 250.0, Paris → 310.0`. Each city gets replaced by the average target value (price) for that city, so the resulting number carries a meaningful, monotonic relationship with the target — cities with higher average prices simply get higher encoded values.

**Why it's useful:** this creates a single numeric column instead of many one-hot columns, and unlike plain label/ordinal encoding, the order isn't arbitrary — it's derived directly from the data itself. The main thing to watch for is target leakage, since the encoding is built using the target variable, so it should be computed only on training data and then applied to test data (not recalculated on the full dataset).

---

## Summary: what to use when

| Problem | Quick/simple method | Smarter method | When to prefer the smarter method |
|---|---|---|---|
| Missing values | Drop rows/columns | Mean / Median / Mode imputation | When missing % is too high to just drop |
| Imbalanced classes | Up/down sampling (`resample`) | SMOTE | When overfitting from duplicate rows is a concern |
| Outliers | Eyeball / ignore | IQR-based fences + boxplot | Any time extreme values could skew the model |
| Categorical (no order) | — | One-Hot Encoding | Low-cardinality nominal columns |
| Categorical (order, few categories) | Label Encoding | Ordinal Encoding | When category order actually matters |
| Categorical (high cardinality) | One-Hot Encoding (blows up columns) | Target Guided Ordinal Encoding | Many unique categories, order tied to the target |

The common thread across all of these: the "smarter" method almost always wins because it uses some extra piece of information — a distribution shape, a real category order, or a relationship with the target — instead of treating every row or category as interchangeable.

---

# Part 2 — Applied EDA & Feature Engineering Case Studies

The techniques above are the individual tools. This part walks through four real datasets, in increasing order of messiness, to see how those tools actually get used together on real data — from a clean numeric dataset all the way to a genuinely broken one that needs real cleanup before any analysis is possible.

1. **Red Wine Quality** — a clean, fully numeric dataset. Good for learning the *core EDA workflow* without fighting the data itself.
2. **Flight Price Prediction** — a messy, text-heavy dataset. This is where feature engineering takes over: turning dates, times, and categories buried in strings into clean numeric features a model can use.
3. **Google Play Store (Data Cleaning)** — an even messier dataset, where numbers are hidden inside broken strings (`'19M'`, `'$4.99'`, `'10,000+'`) and one row is flat-out corrupted.
4. **Google Play Store (EDA on the cleaned data)** — once the data above is actually clean, this is where the real business-question analysis happens: most popular category, most installed category, top apps, and more.

`.info()`, `.isnull().sum()`, and `.duplicated()` show up in every single section because that five-second check is the first thing to run on *any* new dataset, no exceptions.

---

## 8. EDA Case Study — Red Wine Quality Dataset

This is a good dataset to learn the *process* of EDA on, because it's small, clean, and entirely numeric — 11 physicochemical (lab-measured) input features, plus one output: a quality score from a human taster.

**Input features (all numeric):** fixed acidity, volatile acidity, citric acid, residual sugar, chlorides, free sulfur dioxide, total sulfur dioxide, density, pH, sulphates, alcohol.

**Target:** `quality` — a score between 0 and 10. The classes are ordered but not balanced — most wines cluster around average quality (5, 6), and very few are terrible (3) or excellent (8). That's a hint of class imbalance before the data is even loaded.

### Loading and first look
```python
import pandas as pd
df = pd.read_csv('winequality-red.csv')
df.head()          # first 5 rows — always look before doing anything else
df.info()           # column names, non-null counts, dtypes
df.describe()       # count, mean, std, min, 25/50/75%, max per numeric column
df.shape             # (rows, columns)
df['quality'].unique()
```
`.describe()` is where things like a huge min-max range (possible outliers) or a big gap between mean and median (skew) start to show up.

### Missing values and duplicates
```python
df.isnull().sum()               # count of missing values per column
df[df.duplicated()]              # show duplicate rows
df.drop_duplicates(inplace=True)
df.shape                         # confirm rows were removed
```
Duplicate rows matter because a repeated sample gives the model artificially more "votes" for that data point than it should have, which can quietly skew training.

### Correlation between features
```python
df.corr()
```
Returns a matrix of Pearson correlation coefficients (-1 to +1) between every pair of numeric columns. Turned into a heatmap for readability:
```python
import matplotlib.pyplot as plt
import seaborn as sns
plt.figure(figsize=(10,6))
sns.heatmap(df.corr(), annot=True)
```
`annot=True` prints the actual number inside each cell. In this dataset, `alcohol` shows the strongest positive correlation with `quality` — an early signal about which chemical property matters most.

### Checking the target for imbalance
```python
df.quality.value_counts().plot(kind='bar')
```
This confirms the class imbalance suspected earlier — most wines score 5 or 6, very few score 3 or 8. That matters because a model could get high accuracy just by always predicting "5" without learning anything real.

### Distribution of each feature (univariate)
```python
for column in df.columns:
    sns.histplot(df[column], kde=True)
```
`kde=True` overlays a smoothed curve on the histogram, making it easier to see whether a feature is roughly normal, skewed, or has obvious outliers sitting off to one side.

### Pairwise relationships
```python
sns.pairplot(df)
```
Plots every numeric column against every other one (scatter plots) plus each column against itself (histogram on the diagonal) — in one image, this covers univariate, bivariate, and a rough sense of multivariate structure.

### Target vs. a continuous feature
```python
sns.catplot(x='quality', y='alcohol', data=df, kind="box")
```
Draws a box plot of `alcohol` for each quality score, making it easy to see that higher-quality wines tend to have higher median alcohol content.

### Three variables on one 2D plot
```python
sns.scatterplot(x='alcohol', y='pH', hue='quality', data=df)
```
`hue='quality'` colors points by quality score, effectively adding a third variable to a standard 2D scatter plot.

**What this dataset teaches:** correlation only measures *linear* relationships — two variables can be strongly related in a non-linear way and still show near-zero correlation, so `.corr()` shouldn't be treated as the final word. Class imbalance found during EDA should shape modeling choices later (stratified splits, F1-score instead of plain accuracy, resampling). And `.head()` → `.info()` → `.describe()` → `.isnull().sum()` → `.duplicated()` is a five-step ritual worth running on any new dataset before anything else.

---

## 9. EDA & Feature Engineering Case Study — Flight Price Prediction

Where the wine dataset was clean and numeric, this one is the opposite — almost everything useful is buried inside text strings: dates as `24/03/2019`, times as `22:20`, durations as `2h 50m`. This is where feature engineering earns its name: converting raw, human-readable text into clean numeric features a model can actually use.

```python
import pandas as pd, numpy as np, matplotlib.pyplot as plt, seaborn as sns
%matplotlib inline
df = pd.read_excel('flight_price.xlsx')
```
`pd.read_excel()` needs `openpyxl` installed for pandas to read `.xlsx` files. `%matplotlib inline` is a Jupyter magic command that renders plots directly under the code cell instead of a separate window.

**Raw columns:** Airline, Flight, Source, Destination, Route, Date_of_Journey, Dep_Time, Arrival_Time, Duration, Total_Stops, Additional_Info, and the target — Price. Every column except Price starts out as raw text (`dtype: object`).

### Breaking the date apart
```python
df['Date'] = df['Date_of_Journey'].str.split('/').str[0]
df['Month'] = df['Date_of_Journey'].str.split('/').str[1]
df['Year'] = df['Date_of_Journey'].str.split('/').str[2]

df['Date'] = df['Date'].astype(int)
df['Month'] = df['Month'].astype(int)
df['Year'] = df['Year'].astype(int)

df.drop('Date_of_Journey', axis=1, inplace=True)
```
`.str.split('/')` splits `"24/03/2019"` into `['24','03','2019']` for every row; `.str[0]`/`[1]`/`[2]` picks out day/month/year. The pieces are still text after splitting, so `.astype(int)` converts them to real numbers. The original column is dropped once its useful information has been extracted — keeping it adds noise without adding information. This split-extract-cast-drop pattern applies well beyond dates: full names → first/last name, addresses → city/state/zip, and so on.

### Breaking apart arrival and departure times
```python
df['Arrival_Time'] = df['Arrival_Time'].apply(lambda x: x.split(' ')[0])
```
Some `Arrival_Time` values look like `"22:20 22 Mar"` — an arrival date gets appended when a flight lands after midnight. This strips that trailing text, keeping just the time. Checking `.unique()` on a messy column before writing a parser is what catches inconsistencies like this.
```python
df['Arrival_hour'] = df['Arrival_Time'].str.split(':').str[0].astype(int)
df['Arrival_min'] = df['Arrival_Time'].str.split(':').str[1].astype(int)
df.drop('Arrival_Time', axis=1, inplace=True)

df['Departure_hour'] = df['Dep_Time'].str.split(':').str[0].astype(int)
df['Departure_min'] = df['Dep_Time'].str.split(':').str[1].astype(int)
df.drop('Dep_Time', axis=1, inplace=True)
```
Same split → cast → drop recipe applied to both time columns.

### Total_Stops — ordinal encoding + missing value handling in one step
```python
df['Total_Stops'].unique()
# ['non-stop', '2 stops', '1 stop', '3 stops', nan, '4 stops']
```
This column is naturally **ordinal** — the categories have a genuine order (non-stop < 1 stop < 2 stops < ...), unlike something like `Airline`, where there's no inherent ranking between two airline names.
```python
df['Total_Stops'].mode()   # check the most common value before deciding a fill value

df['Total_Stops'] = df['Total_Stops'].map({
    'non-stop': 0, '1 stop': 1, '2 stops': 2, '3 stops': 3, '4 stops': 4, np.nan: 1
})
```
`.map()` with a lookup dictionary does manual ordinal encoding — the numbers 0–4 preserve the real order of the categories. Including `np.nan: 1` in the same dictionary handles the missing-value imputation and the encoding in one call, filling any missing entry with `1` (the mode) at the same time everything else gets converted.
```python
df.drop('Route', axis=1, inplace=True)
```
`Route` (e.g. `"BLR → DEL"`) is dropped since its information is already covered by the separate `Source` and `Destination` columns.

### Duration — a good example of an unfinished feature
```python
df['Duration'].str.split(' ').str[0].str.split('h').str[0]
```
This parses `"2h 50m"` down to just `'2'` (the hour count as text) — but the result is never assigned back to a column or cast to a number. A complete version worth writing:
```python
def duration_to_minutes(duration):
    duration = duration.strip()
    hours, minutes = 0, 0
    if 'h' in duration:
        hours = int(duration.split('h')[0].strip())
        duration = duration.split('h')[1]
    if 'm' in duration:
        minutes = int(duration.replace('m', '').strip())
    return hours * 60 + minutes

df['Duration_mins'] = df['Duration'].apply(duration_to_minutes)
```
Converting duration into a single "total minutes" number is usually more model-friendly than two separate hour/minute columns.

### One-Hot Encoding the non-ordinal categories
```python
from sklearn.preprocessing import OneHotEncoder
encoder = OneHotEncoder()
encoded = encoder.fit_transform(df[['Airline', 'Source', 'Destination']]).toarray()
pd.DataFrame(encoded, columns=encoder.get_feature_names_out())
```
`Airline`, `Source`, and `Destination` are nominal, not ordinal — there's no real ranking between "SpiceJet" and "IndiGo." Encoding them as plain integers (like `Total_Stops` was) would wrongly imply an order or distance between categories that doesn't exist, which is exactly what OHE avoids. The sklearn result comes back as a sparse matrix by default; `.toarray()` turns it into a normal dense array.

**The decision rule this dataset makes clear:**

| Encoding | Use when... | Example here |
|---|---|---|
| Ordinal / manual `.map()` | Categories have a real, meaningful order | `Total_Stops` |
| One-Hot Encoding | Categories have no order — just distinct labels | `Airline`, `Source`, `Destination` |

Getting this backwards is a common mistake — one-hot encoding an ordinal column throws away order information, and ordinal-encoding a nominal column invents a false order the model will wrongly learn from.

**What this dataset teaches:** splitting compound text fields (`.str.split()` + indexing) is the core technique for extracting structure from messy strings — dates, times, addresses, names. Always check `.unique()` before writing a parser. Values are still text after splitting until explicitly cast with `.astype()`. Drop original columns once their information has been extracted. And missing-value handling can be folded into an encoding step via `.map()` with a `np.nan` key.

---

## 10. Data Cleaning Case Study — Google Play Store Dataset

The goal of this dataset is to eventually answer questions like "which app category is most popular" or "which apps have the most installs" — but before any of that, the raw data needs serious cleanup. This is a good example of why data cleaning is its own skill, separate from analysis.

```python
import pandas as pd, numpy as np, matplotlib.pyplot as plt, seaborn as sns, warnings
warnings.filterwarnings("ignore")
%matplotlib inline

df = pd.read_csv('https://raw.githubusercontent.com/krishnaik06/playstore-Dataset/main/googleplaystore.csv')
df.shape     # (10841, 13)
df.info()
df.isnull().sum()
```
`pd.read_csv()` can read directly from a URL, not just a local file. `.info()` reveals the real problem here: columns that should obviously be numeric — `Reviews`, `Size`, `Installs`, `Price` — are all stored as text (`object` dtype) instead.

### Cleaning `Reviews`
```python
df['Reviews'].astype(int)   # throws an error somewhere in this column
df['Reviews'].str.isnumeric().sum()      # how many rows ARE clean numbers
df[~df['Reviews'].str.isnumeric()]        # ~ = NOT, so: show the rows that AREN'T
```
This reveals one row where `Reviews` contains `"3.0M"` instead of a plain number — a corrupted row where values are shifted out of position.
```python
df_copy = df.copy()
df_copy = df_copy.drop(df_copy.index[10472])
df_copy['Reviews'] = df_copy['Reviews'].astype(int)
```
`.copy()` keeps the original `df` safe before making destructive changes. When one row out of ~10,000 is genuinely broken, dropping it is usually safer than trying to guess-repair it. **General lesson:** never force a type conversion blindly — when it fails, that failure is telling you something true about the data; find *which* rows are breaking it before deciding what to do.

### Cleaning `Size`
```python
df_copy['Size'].unique()   # '19M', '14M', '8.7M', '25k', 'Varies with device'

df_copy['Size'] = df_copy['Size'].str.replace('M', '000')
df_copy['Size'] = df_copy['Size'].str.replace('k', '')
df_copy['Size'] = df_copy['Size'].replace('Varies with device', np.nan)
df_copy['Size'] = df_copy['Size'].astype(float)
```
Megabytes are treated as thousands of kilobytes to put everything on one consistent scale. `'Varies with device'` (a real placeholder in the source data) is converted to an actual `np.nan` rather than staying as text that would silently break numeric analysis later — note the difference between `.str.replace` (substring match, used for `M`/`k`) and plain `.replace` (exact full-value match, used for the placeholder text). `float` is used instead of `int` because values include decimals and `NaN` can only exist in a float column.

### Cleaning `Installs` and `Price`
```python
df_copy['Installs'].unique()   # '10,000+', '500,000+', ...
df_copy['Price'].unique()      # '0', '$4.99', '$3.99'

chars_to_remove = ['+', ',', '$']
cols_to_clean = ['Installs', 'Price']
for item in chars_to_remove:
    for cols in cols_to_clean:
        df_copy[cols] = df_copy[cols].str.replace(item, '')

df_copy['Installs'] = df_copy['Installs'].astype('int')
df_copy['Price'] = df_copy['Price'].astype('float')
```
Both columns have symbols glued onto otherwise-numeric values. Instead of writing six near-identical `.str.replace()` lines, a nested loop (characters × columns) handles every combination — a pattern worth reusing whenever the same cleaning line would otherwise get repeated with only one thing changing.

### Handling the `Last Updated` date column
```python
df_copy['Last Updated'] = pd.to_datetime(df_copy['Last Updated'])
df_copy['Day'] = df_copy['Last Updated'].dt.day
df_copy['Month'] = df_copy['Last Updated'].dt.month
df_copy['Year'] = df_copy['Last Updated'].dt.year
```
`pd.to_datetime()` auto-recognizes a wide range of date formats — here, the "Month Day, Year" written-out style — and converts the column to a proper `datetime64` type, unlocking the `.dt` accessor (`.dt.day`, `.dt.month`, `.dt.year`, plus `.dt.dayofweek`, `.dt.quarter`, etc.). **General rule:** reach for `pd.to_datetime()` first for anything date-shaped; only fall back to manual `.str.split()` for formats that are simple *and* consistent.

### Saving the cleaned data
```python
df_copy.to_csv('data/google_cleaned.csv')
```
Never overwrite the original raw file — save the cleaned version under a new name so the source is always still there if the cleaning logic turns out to have a bug.

**The cleaning checklist this dataset teaches:**

| Problem | Technique |
|---|---|
| Numbers stored as text with extra characters | `.str.replace()` to strip symbols, then `.astype()` |
| A type conversion throws an error | `.str.isnumeric()` to find exactly which rows break it |
| One corrupted/misaligned row | Locate by index, drop it explicitly |
| Inconsistent units within a column (M vs k) | Manual conversion to a common scale before casting |
| A placeholder string that really means "missing" | `.replace('text', np.nan)` — exact match, not `.str.replace` |
| Repeated near-identical cleaning steps | Nested loop over characters × columns |
| Human-readable date text | `pd.to_datetime()` + `.dt` accessor |
| Wanting to preserve raw data | `.copy()` before destructive edits, `.to_csv()` to a new file |

The general shape almost every time: inspect (`.unique()`), strip/transform (`.str.replace`, `.replace`, `pd.to_datetime`), then cast (`.astype()`) — always inspect before stripping, since you can't clean what you haven't actually looked at.

---

## 11. EDA Case Study — Google Play Store (Cleaned Dataset)

This continues directly from the cleaned `df_copy` above — now with numeric `Reviews`, `Size`, `Installs`, `Price` and a parsed `Last Updated` date. With clean data in hand, this is where the analysis that answers real business questions actually happens.

### One more cleaning step hiding inside "EDA"
```python
df_copy[df_copy.duplicated('App')].shape
df_copy = df_copy.drop_duplicates(subset=['App'], keep='first')
```
`.duplicated('App')` checks for repeated app names specifically (not whole-row duplicates) — likely from the same app being scraped more than once. `subset=['App']` tells `.drop_duplicates()` to judge duplicates by that column only; `keep='first'` keeps the earliest entry. Real workflows rarely separate "cleaning" and "analysis" as cleanly as a course outline suggests — issues like this often only surface once exploration is already underway.

### Splitting features into numeric vs. categorical, programmatically
```python
numeric_features = [f for f in df_copy.columns if df_copy[f].dtype != 'O']
categorical_features = [f for f in df_copy.columns if df_copy[f].dtype == 'O']
```
This is a list comprehension: for every column, keep it if its dtype is (or isn't) `'O'` — pandas' shorthand for "object"/text. Building this list programmatically avoids typing out column names by hand and stays correct if columns change later.

### Proportions within categorical columns
```python
for col in categorical_features:
    print(df_copy[col].value_counts(normalize=True) * 100)
```
`normalize=True` turns raw counts into proportions (summing to 1); multiplying by 100 gives percentages — more useful than raw counts for answering "what *share* of apps are Paid vs Free."

### Distribution shape of numeric features
```python
plt.figure(figsize=(15, 15))
for i, feature in enumerate(numeric_features):
    plt.subplot(5, 3, i+1)
    sns.kdeplot(x=df_copy[feature], shade=True, color='r')
    plt.xlabel(feature)
    plt.tight_layout()
```
One smoothed KDE curve per numeric column, arranged in a grid instead of one plotting cell per column. `Rating` and `Year` come out left-skewed (long tail toward lower values — most apps are decently rated, most updates are recent); `Reviews`, `Size`, `Installs`, and `Price` come out right-skewed (a small number of mega-popular apps pull the tail out to the right). **Why this matters:** heavily skewed features often benefit from a transformation (like a log transform) before modeling, since a long outlier tail can distort things like distance calculations or linear model coefficients.

### Which category has the most apps? (share of the whole)
```python
df_copy['Category'].value_counts().plot.pie(y=df_copy['Category'], figsize=(15,16), autopct='%1.1f')
```
A pie chart fits here specifically because the question is about "share of the whole." Result: Family, Games, and Tools have the most apps; Beauty, Comics, Art, and Weather have very few by comparison.

### Which category has the most installs? (a different question)
```python
df_cat_installs = df_copy.groupby(['Category'])['Installs'].sum().sort_values(ascending=False).reset_index()
df_cat_installs.Installs = df_cat_installs.Installs / 1_000_000_000   # billions, for readable axis labels

sns.barplot(x='Installs', y='Category', data=df_cat_installs.head(10))
```
This is the key distinction from the pie chart above: **most apps in a category is not the same as most installs for a category.** A category can have thousands of small niche apps (high app *count*) while a different category has fewer apps installed by hundreds of millions of people (high install *sum*). `.groupby().sum()` measures the second thing; `.value_counts()` measures the first. `.reset_index()` turns `Category` back into a normal column after grouping made it the index. Result: GAME leads by a wide margin with roughly 35 billion total installs — even though it wasn't the single largest category by app count.

### Top apps within each category
```python
dfa = df_copy.groupby(['Category', 'App'])['Installs'].sum().reset_index()
dfa = dfa.sort_values('Installs', ascending=False)

for i, category in enumerate(['GAME', 'COMMUNICATION', 'PRODUCTIVITY', 'SOCIAL']):
    df3 = dfa[dfa.Category == category].head(5)
    plt.subplot(4, 2, i+1)
    sns.barplot(data=df3, x='Installs', y='App')
```
Grouping by two columns at once (`Category`, `App`) computes installs per individual app within its category, rather than collapsing everything to one row per category. Result matches real-world intuition: Subway Surfers leads Games, Hangouts leads Communication, Google Drive leads Productivity, Instagram leads Social.

### How many apps have a perfect 5.0 rating?
```python
rating = df_copy.groupby(['Category', 'Installs', 'App'])['Rating'].sum().sort_values(ascending=False).reset_index()
toprating_apps = rating[rating.Rating == 5.0]
toprating_apps.shape[0]   # 271 apps
```
271 apps carry a perfect 5.0 rating. Worth a grain of salt though — a perfect rating with very few reviews is a very different signal than a perfect rating with millions of reviews, which this particular analysis doesn't account for.

**The bigger-picture idea from this dataset:** `groupby()` turns "count of rows" questions (`value_counts`) into "sum/mean/etc. of some other column, per group" questions — and picking the right aggregation depends entirely on the actual business question. "Most apps" and "most installs" sound similar but need two different aggregations to answer correctly; mixing them up gives a technically-working chart with the wrong story behind it.

---

## Case study takeaways

| Dataset | Main challenge | Core skill practiced |
|---|---|---|
| Red Wine Quality | Clean, numeric — no fighting the data | The core EDA workflow itself (info → describe → nulls → duplicates → correlation → distributions) |
| Flight Price | Useful data buried inside text strings | Feature engineering: splitting compound text, casting types, ordinal vs. one-hot encoding |
| Play Store (cleaning) | Numbers stored as broken/inconsistent strings, one corrupted row | Real data cleaning: inspect → strip/transform → cast, handling placeholders and unit inconsistencies |
| Play Store (EDA) | Answering specific business questions correctly | Choosing the right aggregation (`value_counts` vs `groupby().sum()`) for the actual question being asked |

Across all four, the same five-step opening ritual shows up every time — `.head()`, `.info()`, `.describe()`, `.isnull().sum()`, `.duplicated()` — because it's the fastest way to find out what's actually broken in a dataset before deciding how to fix it.
