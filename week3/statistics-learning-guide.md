# Statistics — A Learning Guide From Zero

Before I get into definitions and formulas, here's the one-sentence version of what statistics actually is: **it's a toolkit for turning a pile of numbers into an understanding.** That's it. Everything below — mean, variance, correlation, all of it — exists to answer one of two questions: *"What does this data look like?"* or *"What can I predict from it?"*

I'm writing this guide assuming you're starting from scratch. Every concept builds on the one before it, so read it roughly in order the first time.

---

## Chapter 1: What Is Statistics, Really?

Imagine you're handed a spreadsheet with the ages of 10,000 people. Just staring at 10,000 numbers tells you nothing useful — your brain can't hold that much raw information at once. Statistics is the set of tools that compresses that pile of numbers into something you can actually reason about: an average, a spread, a pattern, a prediction.

That's why statistics shows up everywhere:
- A **business** uses it to spot sales trends and test whether a new feature actually increases revenue.
- A **hospital** uses it to figure out if a new drug genuinely works, or if the results are just luck.
- **Machine learning** is basically statistics wearing a trench coat — every model is built on distributions, averages, spreads, and relationships between variables.
- **Governments** use it for census data, unemployment numbers, and election polling.

The common thread: raw data is noise until statistics turns it into signal.

---

## Chapter 2: The Two Flavors of Statistics

Statistics splits cleanly into two branches, and the split comes down to one question: **do you already have all the data you care about, or are you trying to guess about data you don't have?**

### Descriptive Statistics — "Here's what I already have"

This branch is just about **organizing and summarizing** the data sitting in front of you. No guessing, no generalizing beyond it.

Think of a class of 5 students with heights `{180cm, 170cm, 162cm, 150cm, 160cm}`. If I calculate the average and say "the average height in this class is 165cm" — that's descriptive statistics. I'm not claiming anything about students outside this class. I'm just summarizing what I measured.

Two tools live under this branch:
1. **Measures of Central Tendency** — mean, median, mode (Chapter 4)
2. **Measures of Dispersion** — variance, standard deviation (Chapter 5)

### Inferential Statistics — "Here's my best guess about what I don't have"

This branch uses a **small sample** to make a **conclusion about a much bigger population**, usually backed by some kind of statistical test (z-test, t-test, and so on).

The flow looks like this:

```
Collect a sample  →  Run an experiment/test  →  Draw a conclusion about the whole population
```

Here's the example that makes this click: imagine a college with 1,000 students. You want to know the average height of *all* 1,000 students, but measuring every single one is a hassle. So instead, you measure a sample of, say, 50 students, calculate their average height, and then use that to **infer** what the average height of the entire college probably is. You never touched the other 950 students — you inferred their existence statistically.

**The simplest way to remember the split:** Descriptive = mean, median, mode (describing what's already in your hand). Inferential = conclusion, inference (guessing about what's not in your hand).

---

## Chapter 3: Population vs. Sample — The Distinction Everything Else Depends On

This is one of those ideas that seems small but quietly changes every formula you'll learn later. So let's slow down here.

- The **population** is the *entire* group you actually care about. Its size is written as **N**.
- A **sample** is a smaller *subset* of that population, used to estimate something about the whole thing. Its size is written as **n**.

**Why would you ever settle for a sample instead of just measuring the whole population?** Because measuring the whole population is usually expensive, slow, or flat-out impossible.

Here's the analogy that makes this obvious: **exit polls during an election.** Let's say a country has 100,000 (100K) voters — that's your population, N. Nobody has the time or budget to ask all 100,000 people how they voted. So pollsters survey a sample of, say, 10,000 (10K) voters — that's your sample, n — and use the pattern in that sample to predict how the full 100,000 voted.

Another version of the same idea: if you truly wanted the *population mean* weight of every person in a country, you'd need to weigh every single person — obviously impractical. So instead, you weigh a sample of a few hundred people and use that to estimate the population's average weight.

This N-vs-n distinction is why you'll see two versions of almost every formula from here on — one for population, one for sample. Keep this table nearby, because I'll refer back to it constantly:

| Quantity | Population | Sample |
|---|---|---|
| Size | N | n |
| Mean | μ (mu) | x̄ (x-bar) |
| Variance | σ² | s² |
| Standard Deviation | σ | s |

---

## Chapter 4: Measures of Central Tendency — Finding the "Center" of Your Data

If someone asks "what's typical in this dataset?", central tendency is how you answer. There are three tools: mean, median, and mode — and each one has a specific weakness the others cover for.

### 4.1 Mean — The Everyday Average

This is the one everyone already knows: add everything up, divide by how many values there are.

**Population mean:**
$$\mu = \frac{\sum_{i=1}^{N} x_i}{N}$$

**Sample mean** (identical idea, just swap N for n):
$$\bar{x} = \frac{\sum_{i=1}^{n} x_i}{n}$$

**Example:** Ages = {1, 3, 4, 5}
$$\mu = \frac{1+3+4+5}{4} = \frac{13}{4} = 3.25$$

Simple enough. But here's the catch that trips people up constantly:

**The mean is extremely sensitive to outliers.** Watch what happens when I add a single unusual value to the same dataset. Ages = {1, 3, 4, 5, **100**}:
$$\mu = \frac{1+3+4+5+100}{5} = \frac{113}{5} = 22.6$$

One weird data point (100) dragged the "typical value" from 3.25 all the way up to 22.6 — a number that doesn't represent *any* of the original people in the group. If you reported "the average age here is 22.6," you'd be technically correct and practically misleading. This single flaw is the entire reason the next tool exists.

### 4.2 Median — The Middle Value, and Outlier-Proof

The median is simply **the middle value once you sort the data**. Because it only cares about position, not magnitude, one crazy outlier barely moves it.

**How to find it:**
1. Sort the data smallest to largest.
2. If there's an **odd** number of values, the median is the exact middle one.
3. If there's an **even** number of values, the median is the average of the two middle ones.

**Example (odd count):** {4, 3, 1, 5, 100} → sort it → {1, 3, **4**, 5, 100} → median = **4**

Compare this to the mean from the same dataset, which was 22.6. The median barely flinched at 4, while the mean got wrecked by the outlier. This is exactly why real-world numbers like household income or house prices are almost always reported using the **median**, not the mean — a single billionaire in a sample of "average income" would otherwise make everyone look rich on paper.

**Example (even count):** {1, 3, 4, 5, 100, 200} → the two middle values are 4 and 5 → median = (4+5)/2 = **4.5**

### 4.3 Mode — The Most Frequent Value

The mode is just **whichever value shows up the most often**. It's the only one of the three that works for categories (not just numbers) — you can't take the "mean" of favorite colors, but you can absolutely find the mode (the most popular color).

**Rule of thumb for choosing between the three:** clean, roughly symmetric numeric data → use mean. Numeric data with outliers → use median. Categorical data → use mode.

---

## Chapter 5: Measures of Dispersion — How Spread Out Is the Data?

Central tendency only tells you *where the center is*. It says nothing about *how tightly or loosely the data is scattered around that center* — and that gap is exactly what dispersion measures.

### Why this matters: two datasets, identical mean, wildly different story

$$\text{Ages1} = \{2, 2, 4, 4\} \rightarrow \text{mean} = 3$$
$$\text{Ages2} = \{1, 1, 5, 5\} \rightarrow \text{mean} = 3$$

Both datasets have **exactly the same mean (3)**. But Ages2 is obviously far more spread out than Ages1. If all you reported was "the mean is 3" for both, you'd be hiding a real and important difference between them. This is the entire motivation for variance and standard deviation.

### 5.1 Variance — Average Squared Distance From the Mean

Variance measures, on average, how far each data point sits from the mean — but it squares the distances first. Why square them? Two reasons: (1) it makes every distance positive so they don't cancel each other out when you sum them, and (2) it punishes bigger deviations more heavily than small ones.

**Population variance:**
$$\sigma^2 = \frac{\sum_{i=1}^{N}(x_i - \mu)^2}{N}$$

**Sample variance** (notice the denominator — it's `n-1`, not `n`. Chapter 6 is dedicated entirely to explaining why):
$$s^2 = \frac{\sum_{i=1}^{n}(x_i - \bar{x})^2}{n-1}$$

**Worked example — Ages1 = {2, 2, 4, 4}, mean = 3:**

| xᵢ | mean | (xᵢ − mean)² |
|---|---|---|
| 2 | 3 | 1 |
| 2 | 3 | 1 |
| 4 | 3 | 1 |
| 4 | 3 | 1 |

Sum = 4 → variance = 4/4 = **1**

**Worked example — Ages2 = {1, 1, 5, 5}, mean = 3:**

| xᵢ | mean | (xᵢ − mean)² |
|---|---|---|
| 1 | 3 | 4 |
| 1 | 3 | 4 |
| 5 | 3 | 4 |
| 5 | 3 | 4 |

Sum = 16 → variance = 16/4 = **4**

There it is, in numbers: Ages2's variance (4) is four times bigger than Ages1's variance (1), which matches exactly what your eyes told you when you looked at the two datasets. Variance turned an intuitive "this one looks more spread out" into a precise, comparable number.

---

## Chapter 6: Why Is Sample Variance Divided by n−1?

This is genuinely one of the most commonly asked interview questions in data science, so it's worth understanding properly rather than memorizing.

**The setup:** whenever you take a *sample* out of a bigger population, that sample tends to cluster a little closer to its own sample mean than the full population clusters around the true population mean. This happens because the sample mean was literally *calculated from* those same data points — so of course the points sit unusually close to it. It's a bit self-referential.

**The consequence:** if you calculated sample variance the same way you calculate population variance (dividing by n), you would consistently get a number that's a little *too small* — an underestimate of the true population variance.

**The fix:** divide by `n − 1` instead of `n`. This makes the resulting number slightly larger, correcting for that built-in underestimation. The technical name for this correction is **Bessel's correction**, and the result is called an **unbiased estimator** of the population variance.

Picture a number line of ages. A small sample of points (say, three x's) will typically sit clustered fairly close together — tighter than the true spread of the whole population around the true population mean. Dividing by n−1 nudges the calculated variance upward just enough to compensate for that artificial tightness.

**A question you might be asking:** why not n−2 or n−3 for an even bigger correction? People have experimented with variations in specialized/robust statistics contexts, but the standard, mathematically proven correction — the one that actually produces an unbiased estimator — is **n−1**. That's the version you'll use essentially every time you compute sample variance.

---

## Chapter 7: Standard Deviation — Variance, But in Human-Readable Units

Standard deviation is nothing more than **the square root of variance**. So why bother taking that extra step?

Because variance is measured in *squared* units. If your original data is in centimeters, variance comes out in cm² — which is a strange, hard-to-interpret unit ("the variance is 4 cm-squared" doesn't mean much intuitively). Taking the square root brings the units back to normal (cm), so standard deviation can be read as "on average, how far a typical data point sits from the mean" — in units that actually make sense.

**Population standard deviation:**
$$\sigma = \sqrt{\sigma^2} = \sqrt{\text{Population Variance}}$$

**Sample standard deviation:**
$$s = \sqrt{s^2} = \sqrt{\text{Sample Variance}}$$

**Full recap — all four formulas side by side, so you never have to hunt for them again:**

| | Population | Sample |
|---|---|---|
| Mean | μ = Σxᵢ / N | x̄ = Σxᵢ / n |
| Variance | σ² = Σ(xᵢ−μ)² / N | s² = Σ(xᵢ−x̄)² / (n−1) |
| Standard Deviation | σ = √σ² | s = √s² |

---

## Chapter 8: What Is a Variable?

A **variable** is simply a property that's allowed to take on different values. Age, gender, height, temperature — all variables.

Variables split into two big families:

### 8.1 Quantitative Variables (numbers)
These split again into two types:

- **Discrete** — countable, whole-number values, with no "in-between" values that make sense. Example: number of students in a class (350 — there's no such thing as 350.5 students). Grades like A, B, C also get treated as discrete buckets.
- **Continuous** — can take *any* value along a range, including decimals, with infinite precision possible. Example: height = 175.5cm, weight = 72.7kg. Between any two heights, there's always another possible height in between.

**Quick test to tell them apart:** can you always find a value "between" two neighboring values? If yes → continuous. If the values are naturally whole and countable with nothing sensible in between → discrete.

### 8.2 Qualitative / Categorical Variables
These describe *categories*, not numbers.

- **Gender** → Male, Female
- **Colors** → Red, Green, Blue

The dead giveaway that a variable is categorical: you can't meaningfully average it. "The mean color is 1.5" makes no sense — but "red is the most common color" (the mode) does.

---

## Chapter 9: What Is a Random Variable?

A **random variable (X)** is a variable whose value comes out of some random process or experiment. Think of it as a function that takes "the outcome of an experiment" as input and spits out a number.

```
X  →  (some random process / experiment)  →  a numeric value
```

**Example — tossing a coin.** Define X so that:
$$X = \begin{cases} 0 & \text{if the coin lands Heads} \\ 1 & \text{if the coin lands Tails} \end{cases}$$

You don't know in advance whether you'll get 0 or 1 — that's exactly what makes it "random."

**Example — rolling a fair die.**
$$X \in \{1, 2, 3, 4, 5, 6\}$$

Random variables also split into two types — and yes, this maps directly onto the discrete/continuous split from Chapter 8:

### 9.1 Discrete Random Variable
Takes on a countable set of possible values.
- Tossing a coin → {0, 1}
- Rolling a die → {1, 2, 3, 4, 5, 6}

### 9.2 Continuous Random Variable
Can take any value across a range, often with infinite possible outcomes.
- Height of random people showing up at an event tomorrow — could be 170cm, 160cm, 160.1cm, endlessly precise
- How much it rains tomorrow, in inches — could be 0, 1.1, 5.5, 10.5, 10.75, and so on

---

## Chapter 10: Histograms — Seeing the Shape of Your Data

A **histogram** is a chart that shows how your data is distributed by grouping values into ranges (called **bins**) and plotting how many data points fall into each bin.

**Example dataset (ages):**
```
X = {23, 24, 25, 30, 34, 36, 40, 50, 60, 75, 80}
```

If you group these into bins of width 10:
- 20–30 → 4 values
- 30–40 → 3 values
- 41–50 → 1 value
- 51–60 → 1 value
- 61–70 → 0 values
- 71–80 → 2 values

Plot these bin counts as bars, and suddenly the shape of the data jumps out visually: most people cluster in the 20–40 range, then there's a noticeable empty gap around 61–70 before a small bump at the end.

**Smoothing it out.** If instead of blocky bars you draw a smooth curve tracing the tops of the bins, you get something that looks like a continuous probability curve rather than choppy steps. This smoothing idea connects to **Kernel Density Estimation (KDE)** — a technique that estimates the underlying continuous distribution the data was likely drawn from, instead of just showing raw counts in arbitrary buckets.

**One practical note:** bin width is a real design choice. Too few bins and you lose important detail (everything blurs together). Too many bins and the histogram gets noisy and hard to read. There's no single "correct" number — it's a balance you adjust based on what the data looks like.

---

## Chapter 11: Percentiles and Quartiles — Locating a Value Within the Crowd

### 11.1 Percentage vs. Percentile — Two Words That Sound Alike but Mean Different Things

- **Percentage** answers: *what fraction of the group shares a certain property?*
- **Percentile** answers: *what value does a certain fraction of the data fall below?*

**Percentage example:** Data = {1, 2, 3, 4, 5, 6}. Odd numbers in this set = {1, 3, 5} → that's 3 out of 6 → percentage = 3/6 × 100 = **50%**.

### 11.2 Percentiles

> A **percentile** is a value below which a certain percentage of observations lie.

**Formula:**
$$\text{Percentile of value } x = \frac{\#\text{ of values below } x}{n} \times 100$$

**Example:** Data (sorted) = {2, 2, 3, 4, 5, 5, 6, 7, 8, 8, 9, 9, 10} — n = 14. To find the percentile of the value 9 (its first occurrence in the sorted list): count how many values sit *below* it — that turns out to be 11 values.
$$\text{Percentile} = \frac{11}{14} \times 100 = 78.57\%$$

Interpretation: the value 9 sits above roughly 78.57% of this dataset.

### 11.3 Going Backwards — Finding the Value at a Given Percentile

Sometimes you want the reverse: not "what percentile is this value at?" but "what value sits at the 25th percentile?"

$$\text{Value} = \frac{\text{Percentile} \times (n+1)}{100}$$

**Example:** find the value at the 25th percentile, with n = 14:
$$\text{Value} = \frac{25 \times 15}{100} = \frac{375}{100} = 3.75$$

This 3.75 is telling you a *position* — specifically, 3.75th place in the sorted list, meaning you land between the 3rd and 4th sorted values and interpolate between them to get the actual number.

### 11.4 Quartiles — The Three Percentiles Everyone Actually Uses

Quartiles are just three specific, especially useful percentiles that cut the data into four equal chunks:

- **25%** → 1st Quartile (**Q1**)
- **50%** → 2nd Quartile (**Q2**) — this is exactly the same thing as the median
- **75%** → 3rd Quartile (**Q3**)

```
|--------|--------|--------|
  Q1(25%)  Q2(50%)  Q3(75%)
```

---

## Chapter 12: The 5-Number Summary — A Whole Dataset in Five Numbers

The **5-number summary** is a compact way to describe an entire dataset's spread using exactly five values:

1. **Minimum**
2. **Q1** (1st Quartile, 25%)
3. **Median** (Q2, 50%)
4. **Q3** (3rd Quartile, 75%)
5. **Maximum**

**Worked example.** Data = {1, 2, 2, 2, 3, 3, 4, 5, 5, 5, 6, 6, 6, 6, 7, 8, 8, 9} plus one suspicious outlier tacked on, making n = 20 total.

**Finding Q1:**
$$Q1 = \frac{25}{100} \times (n+1) = \frac{25}{100} \times 20 = 5^{th}\text{ position} = 3$$

**Finding Q3:**
$$Q3 = \frac{75}{100} \times 20 = 15^{th}\text{ position} = 7$$

**Finding the IQR (Interquartile Range)** — this is the width of the "middle 50%" of the data, and it's a spread measure that mostly shrugs off outliers:
$$IQR = Q3 - Q1 = 7 - 3 = 4$$

### 12.1 Using the IQR to Catch Outliers — the "Fence" Trick

$$\text{Lower Fence} = Q1 - 1.5 \times IQR$$
$$\text{Higher Fence} = Q3 + 1.5 \times IQR$$

Plugging in our numbers:
$$\text{Lower Fence} = 3 - 1.5(4) = 3 - 6 = -3$$
$$\text{Higher Fence} = 7 + 1.5(4) = 7 + 6 = 13$$

Any data point that falls **outside** the range [-3, 13] gets flagged as an **outlier**. This exact rule is what a **box plot** is doing under the hood — the box spans from Q1 to Q3, the median line splits it, the whiskers stretch out to the fences, and any point beyond the fences gets drawn as a separate dot to call it out as unusual.

**Final 5-number summary for this dataset:**
- Minimum = 1
- Q1 = 3
- Median = 5
- Q3 = 7
- Maximum = 9 (with a stray value like 20 flagged separately, since it falls far outside the [-3, 13] fence)

---

## Chapter 13: Covariance and Correlation — Do Two Variables Move Together?

Everything up to now has been about describing *one* variable at a time. Covariance and correlation are the tools for the natural next question: **how do two variables relate to each other?**

### 13.1 Covariance

> Covariance measures how much two variables **change together**. Positive covariance means they tend to rise and fall together. Negative covariance means when one rises, the other tends to fall.

**Formula:**
$$Cov(X,Y) = \frac{\sum_{i=1}^{n}(x_i - \bar{x})(y_i - \bar{y})}{n-1}$$

**The intuition behind why the sign works out the way it does:** for each data point, check whether X is above or below its own mean, and whether Y is above or below its own mean.
- X above mean **and** Y above mean → that product term comes out positive
- X below mean **and** Y below mean → also positive (a negative times a negative)
- X above mean but Y below mean (or vice versa) → that product term comes out negative

Add all these terms up, and the overall sign tells you the overall direction of the relationship.

**Worked example — hours studied (X) vs. exam score (Y):**

| Hours Studied (X) | Exam Score (Y) |
|---|---|
| 2 | 50 |
| 3 | 60 |
| 4 | 70 |
| 5 | 80 |
| 6 | 90 |

x̄ = (2+3+4+5+6)/5 = **4**
ȳ = (50+60+70+80+90)/5 = **70**

$$Cov(X,Y) = \frac{(2-4)(50-70)+(3-4)(60-70)+(4-4)(70-70)+(5-4)(80-70)+(6-4)(90-70)}{4} = \frac{80}{4} = 20$$

A positive covariance of 20 confirms what common sense already told us: studying more hours goes hand-in-hand with scoring higher.

**A neat sanity check:** if you compute Cov(X, X) instead of Cov(X, Y), you get exactly Var(X). Covariance of a variable with itself is just its own variance — a good way to double check you actually understand what covariance is doing.

### 13.2 The Problem With Covariance

Covariance has **no fixed range** — the number can be anywhere from -∞ to +∞. That makes it hard to compare relationships across different datasets: is a covariance of 20 "strong"? It depends entirely on the scale of the variables involved, which makes covariance alone kind of useless for comparison. That gap is exactly what **correlation** was invented to fix.

### 13.3 Correlation — Covariance, Rescaled Into a Comparable Number

Correlation squeezes covariance into a fixed, always-comparable range of **-1 to 1**. There are two common versions:

#### Pearson Correlation Coefficient
$$\rho_{x,y} = \frac{Cov(X,Y)}{\sigma_x \cdot \sigma_y}$$

- Closer to **+1** → strong positive relationship
- Closer to **-1** → strong negative relationship
- Closer to **0** → little to no *linear* relationship

Pearson is best suited for relationships that are roughly **straight-line (linear)**.

#### Spearman Rank Correlation
$$r_s = \frac{Cov(R(X), R(Y))}{\sigma(R(X)) \cdot \sigma(R(Y))}$$

Instead of using the raw values of X and Y, Spearman first converts each into its **rank** (1st smallest, 2nd smallest, and so on), then computes the correlation on those ranks instead.

**Why bother ranking first?** Because it lets Spearman catch relationships that are **consistently increasing or decreasing but not actually a straight line** — a curved relationship still counts, as long as it never reverses direction. A neat real example: for a curved-but-always-increasing dataset, Spearman correlation can come out as a perfect **1**, while Pearson correlation on the same data is only **0.88** — because Pearson is strictly checking "how well does a straight line fit," while Spearman only checks "does Y keep increasing every time X increases."

### 13.4 Where This Actually Gets Used — Feature Selection in ML

Correlation is one of the first tools you reach for when deciding which features actually matter for a model. Take predicting house price as an example:

- **Size of house ↑** → correlates with **Price ↑** (strong positive correlation → keep this feature)
- **Number of rooms ↑** → also positively correlated → keep it
- **Number of people currently staying in the house** → correlation ≈ 0 → basically irrelevant to price → safe to drop
- **Whether a TV is mounted on the wall** → can sometimes show a surprising *negative* correlation with price in real datasets — not because mounted TVs cause lower prices, but because of some hidden confounding pattern in the data

**The one rule to never forget:** correlation tells you two variables move together — it does **not** tell you that one *causes* the other. Always sanity-check a correlated feature before assuming it's actually driving the outcome, especially in real-world messy data where spurious correlations show up more often than you'd expect.

---

## Quick Recap — The Whole Guide in One Table

| Concept | One-line takeaway |
|---|---|
| Statistics | Turning raw data into understanding |
| Descriptive vs Inferential | Describing what you have vs. guessing about what you don't |
| Population vs Sample | N = everyone, n = the subset you actually measured |
| Mean | Average — sensitive to outliers |
| Median | Middle value — resistant to outliers |
| Mode | Most frequent value — works for categories too |
| Variance | Average squared distance from the mean (spread) |
| n−1 in sample variance | Corrects for samples underestimating true spread (Bessel's correction) |
| Standard deviation | Square root of variance — same units as original data |
| Variable | A property that can take different values |
| Random variable | A variable whose value comes from a random process |
| Histogram | Visualizing the shape of data via binned frequency counts |
| Percentile | Value below which X% of data falls |
| Quartile | The 25%, 50%, 75% percentiles specifically |
| 5-number summary | Min, Q1, Median, Q3, Max — a dataset in five numbers |
| IQR + fences | Used to mathematically flag outliers |
| Covariance | Direction two variables move together — unbounded range |
| Correlation | Covariance rescaled to a fixed -1 to 1 range for comparability |
