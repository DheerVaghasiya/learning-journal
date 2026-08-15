# Probability — A Learning Guide From Zero

Probability is the math of uncertainty. Whenever you don't know exactly what's going to happen but you want to reason about it anyway — will it rain tomorrow, will this coin land heads, will this card be a king — probability gives you a precise number to work with instead of just a shrug.

This guide starts with the core rules — the **Addition Rule** (for "OR" questions) and the **Multiplication Rule** (for "AND" questions) — then builds up to probability distributions: the different "shapes" that randomness can take, how to describe them (PMF, PDF, CDF), the specific distributions you'll run into constantly in ML and data science (Bernoulli, Binomial, Poisson, Normal, Uniform, Log-Normal, Power Law, Pareto), and finally the two ideas that connect probability back to real-world statistics: the Central Limit Theorem and Estimates. Read it in order — later chapters lean on vocabulary introduced earlier.

---

## Chapter 1: What Is Probability?

**Probability is about determining the likelihood of an event.** It's always a number between 0 and 1 (or equivalently, 0% to 100%) — 0 means "definitely won't happen," 1 means "definitely will happen," and everything in between is a degree of confidence.

**Example — tossing a coin.** The set of possible outcomes is {H, T}. Since both outcomes are equally likely:
$$Pr(H) = \frac{1}{2} = 50\%$$
$$Pr(T) = \frac{1}{2} = 50\%$$

**Example — rolling a die.** The set of possible outcomes is {1, 2, 3, 4, 5, 6}. Each face is equally likely, so the probability of any single specific number is:
$$Pr(X=1) = \frac{1}{6}$$

The general pattern behind both examples: **probability of an outcome = (number of ways that outcome can happen) / (total number of possible outcomes)** — assuming every outcome is equally likely, which is true for a fair coin or a fair die.

With the basics in place, the real skill in probability is combining events — asking things like "what's the probability of *this* happening OR *that* happening?" or "what's the probability of *this* AND *that* both happening?" That's exactly what the next two chapters cover.

---

## Chapter 2: The Addition Rule — Answering "OR" Questions

Whenever a question has the word **"or"** in it — "what's the probability of rolling a 1 or a 5?" — you're in Addition Rule territory. But before you can apply it, you need to answer one setup question first: **can both events happen at the same time, or not?** That answer determines which version of the rule you use.

### 2.1 Mutually Exclusive Events

> Two events are **mutually exclusive** if they cannot occur at the same time.

**Example — tossing a coin.** A single coin toss gives you either Heads or Tails — never both at once. So "getting Heads" and "getting Tails" are mutually exclusive events.

$$Pr(H) = \frac{1}{2} \qquad Pr(T) = \frac{1}{2}$$

**The Addition Rule for mutually exclusive events:**
$$Pr(H \text{ or } T) = Pr(H) + Pr(T)$$

Plugging in:
$$Pr(H \text{ or } T) = \frac{1}{2} + \frac{1}{2} = 1$$

That result of exactly 1 makes total sense — a coin toss is *guaranteed* to land on either Heads or Tails, so the combined probability has to be 100%.

**Why does simple addition work here?** Because the two events never overlap. There's no scenario where you'd be double-counting an outcome that belongs to both "Heads" and "Tails" at once — they're completely separate buckets, so you can just add their individual probabilities directly.

**Example — rolling a die, "1 or 5".** Since a single roll can never produce both a 1 and a 5 simultaneously, these two outcomes are mutually exclusive too:
$$Pr(1 \text{ or } 5) = Pr(1) + Pr(5) = \frac{1}{6} + \frac{1}{6} = \frac{2}{6} = \frac{1}{3}$$

**The general formula to remember:**
$$Pr(A \text{ or } B) = Pr(A) + Pr(B) \quad \text{(only valid when A and B are mutually exclusive)}$$

### 2.2 Non-Mutually Exclusive Events

Here's where it gets more interesting: what if the two events *can* actually happen at the same time?

> Two events are **non-mutually exclusive** if they *can* occur together — there's overlap between them.

**Example — drawing a card from a deck, "King or Heart".** A standard deck has 52 cards. There are 4 Kings, and there are 13 Hearts. But here's the catch: **the King of Hearts is both a King and a Heart at the same time.** So "King" and "Heart" overlap — they're non-mutually exclusive.

If you naively just added Pr(King) + Pr(Heart), you'd be counting the King of Hearts **twice** — once as part of "King" and once as part of "Heart." That double-count needs to be subtracted back out.

**The Addition Rule for non-mutually exclusive events:**
$$Pr(A \text{ or } B) = Pr(A) + Pr(B) - Pr(A \text{ and } B)$$

That last term — subtracting Pr(A and B) — is the entire difference from the mutually exclusive version. It exists purely to correct for double-counting the overlap.

**Worked example:**
$$Pr(K \text{ or } \heartsuit) = Pr(K) + Pr(\heartsuit) - Pr(K \text{ and } \heartsuit)$$
$$= \frac{4}{52} + \frac{13}{52} - \frac{1}{52}$$
$$= \frac{17}{52} - \frac{1}{52} = \frac{16}{52}$$

Picture this as a Venn diagram: one circle for "Kings" (4 cards), one circle for "Hearts" (13 cards), overlapping in exactly one spot — the King of Hearts. If you just add the sizes of both circles, you count that overlapping card twice. Subtracting Pr(A and B) once removes exactly one of those duplicate counts, leaving you with the correct total.

### 2.3 The Big Picture — One Formula, Two Cases

Here's the thing worth noticing: the mutually exclusive formula is actually just a **special case** of the general one.

$$Pr(A \text{ or } B) = Pr(A) + Pr(B) - Pr(A \text{ and } B)$$

When A and B are mutually exclusive, they can *never* happen together, so Pr(A and B) = 0 — and the formula collapses right back down to Pr(A) + Pr(B). That's why the "simple" addition rule you learned first is really just this general formula with the overlap term equal to zero.

**Quick decision guide:**
- Can the two events happen at the same time? **No** → mutually exclusive → just add the probabilities.
- Can the two events happen at the same time? **Yes** → non-mutually exclusive → add the probabilities, then subtract the overlap.

---

## Chapter 3: The Multiplication Rule — Answering "AND" Questions

If the Addition Rule handles "OR," the **Multiplication Rule** handles **"AND"** — questions like "what's the probability of getting Heads *and* then Tails?" or "what's the probability of drawing a King *and* then a Queen?"

Just like with the Addition Rule, there's a setup question you need to answer first before picking the right version: **does the first event affect the probability of the second event, or not?**

### 3.1 Independent Events

> Two events are **independent** if they do not affect one another.

**Example — tossing a coin, then tossing it again.** Getting Heads on your first toss has zero influence on what happens on your second toss. The coin has no memory. So these are independent events.

$$Pr(H) = \frac{1}{2} \qquad Pr(T) = \frac{1}{2}$$

**Example — rolling a die.** Rolling a 1 on your first roll doesn't change the odds of rolling a 2 on your next roll — the die doesn't "remember" its previous result either.
$$Pr(1) = \frac{1}{6} \qquad Pr(2) = \frac{1}{6}$$

**The Multiplication Rule for independent events:**
$$Pr(A \text{ and } B) = Pr(A) \times Pr(B)$$

**Worked example — coin toss, Heads and then Tails:**
$$Pr(H \text{ and } T) = Pr(H) \times Pr(T) = \frac{1}{2} \times \frac{1}{2} = \frac{1}{4}$$

**Why multiply instead of add?** Addition answers "does at least one of these happen?" — multiplication answers "do *both* of these specific things happen, in sequence?" Since each event's outcome is independent of the other, you multiply the individual probabilities together to get the probability of both specific outcomes lining up.

### 3.2 Dependent Events

> Two events are **dependent** if they *do* affect each other — the outcome of the first event changes the probability of the second.

**Example — drawing a King from a deck, then drawing a Queen from the same deck (without putting the King back).**

Before drawing anything:
$$Pr(K) = \frac{4}{52}$$

But here's the key difference from the independent case: once you've pulled out a King and set it aside, the deck now only has **51 cards left** (not 52) — you removed a card, so the total pool shrank. This changes the probability of the second draw:
$$Pr(Q) = \frac{4}{51}$$

Notice the denominator changed from 52 to 51 — that's the fingerprint of a dependent event. The first draw physically altered the conditions of the second draw.

This idea — "the probability of the second event, *given* that the first event already happened" — has a name: **conditional probability**, usually written as Pr(B | A), read as "probability of B given A."

**The Multiplication Rule for dependent events:**
$$Pr(A \text{ and } B) = Pr(A) \times Pr(B \mid A)$$

**Worked example — King and then Queen:**
$$Pr(K \text{ and } Q) = Pr(K) \times Pr(Q \mid K) = \frac{4}{52} \times \frac{4}{51}$$

Multiplying these out: $\frac{4}{52} \times \frac{4}{51} = \frac{16}{2652} \approx 0.006$ — a fairly small probability, which makes intuitive sense since you need two specific, unlikely events to both happen in sequence.

### 3.3 The Big Picture — Independent Is a Special Case Too

Just like with the Addition Rule, the independent-events version is really a **special case** of the more general dependent-events formula.

$$Pr(A \text{ and } B) = Pr(A) \times Pr(B \mid A)$$

When A and B are independent, event A has zero effect on event B — so Pr(B | A) is exactly the same as plain Pr(B), regardless of whether A happened. Substitute that in, and the formula collapses back to the simple version:

$$Pr(A \text{ and } B) = Pr(A) \times Pr(B)$$

**Quick decision guide:**
- Does the first event change the odds of the second? **No** → independent → just multiply the two plain probabilities.
- Does the first event change the odds of the second? **Yes** → dependent → multiply Pr(A) by the *conditional* probability Pr(B | A), not plain Pr(B).

**A fast way to sniff out dependence:** ask yourself "did anything get removed, used up, or changed as a result of the first event?" Drawing a card without replacement removes a card from the deck → dependent. Flipping a coin twice, or rolling a die twice, changes nothing about the object between rolls → independent.

---

## Chapter 4: The Relationship Between PMF, PDF, and CDF

Before diving into specific distributions, you need three tools for *describing* a distribution — because "distribution" just means "how probability is spread across all the possible values a random variable can take." These three tools are how you actually write that down mathematically.

> **Probability distribution functions describe how probability is distributed over the values of a random variable.**

Which tool you use depends entirely on one thing: is your random variable **discrete** or **continuous**?

### 4.1 Probability Mass Function (PMF) — for Discrete Random Variables

The PMF gives the exact probability of a random variable taking on one *specific* value. It only makes sense for discrete variables, because you can only ask "what's the probability of exactly this value?" when the values are countable and separate.

**Example — rolling a fair die.** X = {1, 2, 3, 4, 5, 6}, and since the die is fair, every outcome is equally likely:
$$Pr(1) = Pr(2) = Pr(3) = Pr(4) = Pr(5) = Pr(6) = \frac{1}{6}$$

If you plot this — value on the x-axis, probability on the y-axis — you get a set of separate bars, each at height 1/6. That bar chart *is* the PMF.

### 4.2 Probability Density Function (PDF) — for Continuous Random Variables

Here's the twist that trips people up when they first meet continuous variables: **for a continuous random variable, the probability of it landing on any one exact value is technically 0.** Think about it — if X can be any real number like age = 30, 30.0001, 30.00001, and so on infinitely, the chance of hitting *exactly* 30.000000... is vanishingly small.

So for continuous variables, you don't ask "what's the probability of exactly this value?" — you ask "what's the probability the value falls *within a range*?" That's what the PDF is built for.

**The key idea: with a PDF, probability is read off as *area under the curve*, not the height of the curve at a point.**

**Example.** Say X = Ages, and you have a smooth PDF curve. To find Pr(X ≤ 40), you don't look at the height of the curve at x=40 — you find the *area under the curve* from the left edge up to x=40. If that shaded area works out to 0.5, then Pr(X ≤ 40) = 50%.

**PDF Properties** — every valid PDF must satisfy both of these:
1. **Non-negativity:** f(x) ≥ 0 for all x (probability density can never be negative)
2. **Total area = 1:**
$$\int_{-\infty}^{\infty} f(x)\,dx = 1$$

That second property makes intuitive sense — the random variable has to land *somewhere* in its full range, so the total probability across all possible values must add up to 100%.

**One important nuance:** the exact shape of f(x) changes depending on which distribution you're dealing with — a Normal distribution has a bell-shaped f(x), a Log-Normal has a right-skewed f(x), and so on. The two properties above (non-negative, area = 1) are the only things *every* PDF has in common.

### 4.3 Cumulative Distribution Function (CDF) — Works for Both

The CDF answers a slightly different question: **"what's the probability that X is less than or equal to some value x?"** It accumulates probability as you move left to right — hence "cumulative."

**For discrete variables (built from the PMF):**
$$Pr(X \leq 2) = Pr(X=1) + Pr(X=2)$$

Using the die example: Pr(X ≤ 2) = 1/6 + 1/6 = 2/6 = 1/3. Keep adding, and by the time you reach Pr(X ≤ 6), you've summed every possible outcome, so it equals exactly 1 (100% — the value has to be *something* between 1 and 6).

**For continuous variables (built from the PDF):** the CDF at a point x is just the running total area under the PDF curve from -∞ up to x. This is why, visually, a CDF for continuous data looks like a smooth S-shaped curve climbing from 0 up to 1.

### 4.4 The Relationship — Density Is the *Slope* of the Cumulative Curve

Here's the connective tissue between PDF and CDF, and it's worth sitting with:

> **Probability density = the gradient (slope) of the cumulative distribution function.**

If you pick two nearby points on the CDF curve, say at x=38 and x=44, and calculate the slope between them:
$$\text{slope} = \frac{0.65 - 0.45}{44 - 38}$$

that slope value is telling you the *probability density* at that region of the curve — which is exactly what the PDF is plotting. In other words: **PDF is the derivative of the CDF, and CDF is the integral (accumulated area) of the PDF.** They're two views of the exact same information — one shows you the instantaneous "density" at each point, the other shows you the running total.

**Quick summary table:**

| | Used for | Question it answers | Visual |
|---|---|---|---|
| PMF | Discrete random variable | Probability of exactly this value | Separate bars |
| PDF | Continuous random variable | Density at this point (probability = area under curve between two points) | Smooth curve |
| CDF | Both | Probability of X ≤ this value | Non-decreasing curve from 0 to 1 |

---

## Chapter 5: Types of Probability Distributions — The Map Before the Territory

Before going through each distribution one by one, it helps to see the full list up front, because they're not random — each one models a specific *kind* of real-world randomness.

| Distribution | Type | Used for |
|---|---|---|
| Bernoulli | PMF (discrete) | A single yes/no, success/failure trial |
| Binomial | PMF (discrete) | Counting successes across repeated yes/no trials |
| Poisson | PMF (discrete) | Counting events in a fixed time interval |
| Normal/Gaussian | PDF (continuous) | Naturally symmetric, bell-shaped real-world data |
| Uniform | PMF or PDF | Every outcome equally likely |
| Log-Normal | PDF (continuous) | Right-skewed data whose *logarithm* is normal |
| Power Law | PDF (continuous) | "Few dominate the many" phenomena |
| Pareto | PDF (continuous) | The specific power-law behind the 80-20 rule |

**A useful mental anchor:** picture a real dataset for house price prediction, with columns like size of house, number of rooms, location, floor, sea-facing (yes/no), and price. Size of house and price are **continuous** → described with a PDF. Number of rooms is **discrete** → PMF. Sea-facing is **binary (0 or 1)** → PMF, and this is actually a Bernoulli-flavored variable. Recognizing which "shape" a real column matches is exactly the skill this chapter set builds toward — and it's core to exploratory data analysis (EDA) and feature engineering as a data analyst or data scientist.

---

## Chapter 6: Bernoulli Distribution

> The **Bernoulli distribution** is the simplest discrete probability distribution. It models a random variable with exactly two possible outcomes — success (with probability **p**) and failure (with probability **q = 1-p**). Think: a single coin flip, or a single yes/no question.

**Properties:**
1. Discrete Random Variable, described with a PMF
2. Outcomes are strictly binary

**Example — tossing a coin.**
$$Pr(X=H) = 0.5 = p \qquad Pr(X=T) = 1 - 0.5 = 0.5 = q$$

**Example — pass/fail.** If the probability of passing is 0.4:
$$Pr(X=\text{Pass}) = 0.4 \qquad Pr(X=\text{Fail}) = 1 - 0.4 = 0.6$$

**Notation convention:** success is labeled **k=1**, failure is labeled **k=0**.

**Parameters:**
- 0 ≤ p ≤ 1
- q = 1 - p
- k ∈ {0, 1}

### 6.1 The PMF Formula

$$PMF = p^k (1-p)^{1-k}$$

This single formula is a compact way of writing both cases at once. Plug in k=1:
$$Pr(k=1) = p^1(1-p)^{1-1} = p^1(1-p)^0 = p$$

Plug in k=0:
$$Pr(k=0) = p^0(1-p)^{1-0} = 1 \cdot (1-p) = (1-p) = q$$

**Simplified, this is really just:**
$$PMF = \begin{cases} q = 1-p & \text{if } k=0 \\ p & \text{if } k=1 \end{cases}$$

**Worked example — a company launches a new smartphone.** Say 60% of surveyed people say they'd use it, 40% say they wouldn't:
- Pr(X=1, "would use") = 0.6 = p
- Pr(X=0, "would not use") = 0.4 = q

Plotted as a PMF, this is just two bars: one at height 0.4 above k=0, one at height 0.6 above k=1.

### 6.2 Mean, Median, Mode, and Variance of a Bernoulli Distribution

**Mean:**
$$E(X) = \sum_{k=0}^{1} k \cdot p(k) = (0 \times q) + (1 \times p) = p$$

Using the smartphone example: E(X) = 0×0.40 + 1×0.60 = **0.60** — which is just p. Makes sense: the "average outcome" of a binary variable coded as 0/1 is simply the probability of getting the 1.

**Median:**
$$\text{Median} = \begin{cases} 0 & \text{if } p < 0.5 \\ [0,1]\text{ (any value in this range)} & \text{if } p = 0.5 \\ 1 & \text{if } p > 0.5 \end{cases}$$

**Mode:** whichever of p or q is larger wins as the mode. If p > q, the mode is 1 (success is more common). Otherwise, the mode is 0.

**Variance:**
$$\sigma^2 = q(0-p)^2 + p(1-p)^2$$

Worked out with the smartphone numbers (p=0.6, q=0.4):
$$\sigma^2 = 0.40(0-0.6)^2 + 0.6(1-0.6)^2 = 0.40(0.36) + 0.6(0.16) = 0.144 + 0.096 = 0.24$$

There's also a shortcut formula that skips the manual expansion entirely:
$$\sigma^2 = pq \qquad \sigma = \sqrt{pq}$$

---

## Chapter 7: Binomial Distribution

If Bernoulli is a *single* yes/no trial, the **Binomial distribution** is what happens when you repeat that same yes/no trial **n times** and count how many successes you get.

> The **binomial distribution**, with parameters n and p, is the discrete probability distribution of the number of successes in a sequence of **n independent** yes/no trials, each with the same success probability p. A single trial (n=1) is just a Bernoulli trial — so Binomial is really "many Bernoullis stacked together."

**Properties:**
1. Discrete Random Variable
2. Every individual outcome of the experiment is binary (success/failure)
3. The experiment is repeated for **n** trials

**Example — tossing a coin 10 times.** Here n = 10, and each toss is {H, T}.

**Notation:** B(n, p)

**Parameters:**
- n ∈ {0, 1, 2, ...} — number of trials
- p ∈ [0, 1] — success probability for each individual trial
- q = 1 - p

**Support (the possible values the outcome can take):**
- k ∈ {0, 1, 2, 3, ..., n} — the number of successes, which can range anywhere from 0 (no successes at all) to n (every single trial succeeded)

### 7.1 The PMF Formula

$$Pr(k, n, p) = {^n C_k}\, p^k (1-p)^{n-k}$$

Where ⁿCₖ is the **binomial coefficient**:
$$^nC_k = \frac{n!}{k!(n-k)!}$$

**What each piece is doing:** ${p^k}$ is the probability of getting k successes in a row, $(1-p)^{n-k}$ is the probability of the remaining (n-k) trials all being failures, and ⁿCₖ counts *how many different orderings* could produce exactly k successes out of n trials (since the successes don't have to come first — they could be scattered anywhere across the n trials).

### 7.2 Mean, Variance, Standard Deviation

$$\text{Mean} = n \cdot p \qquad \text{Variance} = npq \qquad \sigma = \sqrt{npq}$$

### 7.3 Worked Example — Coin Flips

Setup: n = 5 trials, p = 0.5 (fair coin), and k (number of successes) can range from 0 to 5.

**Question: what's the probability of getting exactly 3 heads in 5 flips?**

n = 5, k = 3:
$$Pr(X=3) = {^5C_3}(0.5)^3(1-0.5)^{5-3} = 0.3125$$

### 7.4 Worked Example — Quality Control

**Scenario:** inspecting 10 items in a factory, where each item independently has a 10% chance of being defective.
- Number of trials (n) = 10
- Probability of success (p) = 0.1 (here, "success" = "found defective")
- Number of successes (k) can range from 0 to 10

**Question: what's the probability of finding exactly 2 defective items in a sample of 10?**
$$Pr(X=2) = {^{10}C_2}(0.1)^2(1-0.1)^{10-2} \approx 0.1937 \; (19.37\%)$$

This kind of setup — repeated independent trials, counting how many "hit" some condition — shows up constantly in QA/manufacturing, A/B testing (number of conversions out of n visitors), and reliability engineering.

---

## Chapter 8: Poisson Distribution

> The **Poisson distribution** is a discrete probability distribution that expresses the probability of a given **number of events occurring in a fixed interval of time**, assuming the events happen at a known constant average rate, and independently of how much time has passed since the last event.

The key difference from Binomial: Binomial asks "out of n discrete trials, how many succeed?" Poisson asks "in a continuous stretch of time (or space), how many events happen?" — there's no fixed "n" of trials, just a rate.

**Properties:**
1. Discrete Random Variable, PMF
2. Describes the number of events occurring in fixed time intervals

**Examples:**
- Number of people visiting a hospital every hour
- Number of people visiting a bank every hour

**The parameter λ (lambda):**
$$\lambda = \text{expected number of events occurring in each time interval}$$

**Example — a bank sees, on average, λ = 3 people per hour.** If you plot actual observed counts per hour (say 1pm: 2 people, 2pm: 3 people, 3pm: 3 people, 4pm: 5 people, 5pm: 1 person), the Poisson distribution is the model that describes how likely each of those counts is, centered around the expected rate λ=3.

### 8.1 The PMF Formula

$$Pr(X=x) = \frac{e^{-\lambda}\lambda^x}{x!}$$

**Worked example — with λ = 3, find Pr(X=5):**
$$Pr(X=5) = \frac{e^{-3} \cdot 3^5}{5!} = 0.101 = 10.1\%$$

**Combining outcomes.** Just like with any PMF, you can add up individual probabilities to answer range questions:
$$Pr(X=4) + Pr(X=5) = \text{probability that either 4 or 5 events occur}$$

### 8.2 Mean and Variance

$$\text{Mean} = E(X) = \mu = \lambda \times t$$

where **t** is the time interval you're measuring over. (Variance of a Poisson distribution is also equal to λ — one of its distinctive properties is that mean and variance are the same number.)

**Why Poisson matters in practice:** it's the go-to model for call-center call volume, website traffic spikes, defect counts per batch, radioactive decay events, and pretty much any "how many times does X happen in this window" question.

---

## Chapter 9: Normal / Gaussian Distribution

This is the single most important distribution in all of statistics — the classic "bell curve."

> A **normal distribution** (or **Gaussian distribution**) is a type of **continuous probability distribution** for a real-valued random variable.

**Properties:**
1. Continuous Random Variable, described with a PDF
2. As variance (σ²) increases, the spread of the curve increases — same center, wider or narrower bell
3. It's a **symmetric distribution** — a true bell curve, perfectly mirrored on either side of the center
4. Because it's symmetric: **mean = median = mode**, all landing on the exact same point in the middle

**Real-world examples that tend to follow a normal distribution:**
- Weights of students in a class
- Heights of students in a class
- Iris dataset measurements — petal length, petal width, sepal length, sepal width (a dataset researchers use constantly for exactly this reason)

**Notation:** N(μ, σ²)

**Parameters:**
- μ ∈ ℝ — the mean
- σ² ∈ ℝ, σ² > 0 — the variance
- x ∈ ℝ — any real number is a valid input

### 9.1 The PDF Formula

$$f(x) = \frac{1}{\sigma\sqrt{2\pi}}\, e^{-\frac{1}{2}\left(\frac{x_i - \mu}{\sigma}\right)^2}$$

You won't need to compute this by hand often, but it's worth recognizing: the $(x_i - \mu)/\sigma$ term inside is exactly the **z-score** (Chapter 10), and the exponential shape is what produces that classic bell curve.

**Mean and Variance (same formulas from the Statistics guide):**
$$\mu = \frac{\sum_{i=1}^{n} x_i}{n} \qquad \sigma^2 = \frac{\sum_{i=1}^{n}(x_i-\mu)^2}{n} \qquad \sigma = \sqrt{\text{Variance}}$$

### 9.2 The Empirical Rule (68-95-99.7 Rule)

This rule is one of the most useful shortcuts in all of statistics — once you know data is normally distributed, this tells you exactly how it's spread out without any extra calculation:

$$Pr(\mu - \sigma \leq X \leq \mu + \sigma) \approx 68\%$$
$$Pr(\mu - 2\sigma \leq X \leq \mu + 2\sigma) \approx 95\%$$
$$Pr(\mu - 3\sigma \leq X \leq \mu + 3\sigma) \approx 99.7\%$$

In plain words: about 68% of your data sits within 1 standard deviation of the mean, 95% within 2 standard deviations, and a whopping 99.7% within 3 standard deviations. This is exactly why a data point sitting beyond 3 standard deviations from the mean is treated as a strong outlier candidate — under a normal distribution, landing out there is genuinely rare (less than 0.3% chance).

**A quick way to check if your real-world data actually is normally distributed:** a **QQ plot** (quantile-quantile plot) compares your data's distribution against a theoretical normal distribution — if the points fall roughly along a straight diagonal line, your data is a good match for normal.

---

## Chapter 10: Standard Normal Distribution and Z-Score

Every normal distribution has its own unique μ and σ — a dataset of ages might be N(30, 5), while a dataset of salaries might be N(50000, 12000). That makes it hard to directly compare "how unusual" a value is across different datasets, since the scales are completely different. The **Standard Normal Distribution** fixes this by putting every normal distribution on the exact same footing.

> The **Standard Normal Distribution (SND)** is a special case of the normal distribution where **μ = 0** and **σ = 1**.

### 10.1 The Z-Score — Converting Any Value Onto the Standard Scale

The **z-score** tells you how many standard deviations a data point sits away from its mean — and once converted, any dataset's values become directly comparable, no matter their original units or scale.

$$z\text{-score} = \frac{x_i - \mu}{\sigma}$$

**Worked example.** X = {1, 2, 3, 4, 5}, with μ = 3 and σ ≈ 1.414 (rounded to 1 for simplicity).

Converting every value to a z-score:
1. (1-3)/1 = **-2**
2. (2-3)/1 = **-1**
3. (3-3)/1 = **0**
4. (4-3)/1 = **1**
5. (5-3)/1 = **2** (note: this example uses a rounded σ of 1)

Notice what happened: the original distribution centered at μ=3 got shifted and rescaled into a new distribution centered exactly at 0, with a standard deviation of exactly 1 — i.e., X ~ SND(μ=0, σ=1).

**Interpreting a z-score:** a positive z-score means the value is *above* the mean; negative means *below*. The magnitude tells you *how far*, in standard-deviation units.

**Another worked example — μ=4, σ=1:**
- xᵢ = 4.25 → z-score = (4.25-4)/1 = **0.25** (a quarter of a standard deviation above the mean)
- xᵢ = 2.5 → z-score = (2.5-4)/1 = **-1.5** (one and a half standard deviations below the mean)

### 10.2 Why Z-Scores (Standardization) Matter in Machine Learning

Real datasets almost never have features on the same scale. Consider a dataset with:

| Age (years) | Weight (kg) | Height (cm) | Salary (INR) |
|---|---|---|---|
| 24 | 70 | 175 | 40,000 |
| 25 | 60 | 160 | 50,000 |
| ... | ... | ... | ... |

Age ranges roughly 20-35, but Salary ranges in the tens of thousands — wildly different scales. Feed this directly into most ML models (clustering algorithms, linear regression, logistic regression) and the salary column will dominate purely because its raw numbers are bigger, not because it's actually more important.

**The fix: standardization** — converting every column to z-scores so everything sits on the same "standard deviations from the mean" scale:

$$z\text{-score} = \frac{x_i - \mu_{age}}{\sigma_{age}} \qquad z\text{-score} = \frac{x_i - \mu_{weight}}{\sigma_{weight}}$$

...and so on for every column. This is a standard preprocessing step before feeding data into distance-based or gradient-based ML models like clustering algorithms, linear regression, and logistic regression.

---

## Chapter 11: Uniform Distribution

> A **uniform distribution** describes a situation where every possible outcome is **equally likely** — there's no "bunching up" anywhere, the probability is spread perfectly flat across the range.

There are two versions, depending on whether the variable is continuous or discrete.

### 11.1 Continuous Uniform Distribution

**Notation:** U(a, b), where a and b are the minimum and maximum bounds (-∞ < a < b < ∞).

**PDF:**
$$f(x) = \begin{cases} \dfrac{1}{b-a} & x \in [a,b] \\ 0 & \text{otherwise} \end{cases}$$

Notice the density is a flat constant value (1/(b-a)) across the entire range — that flatness is exactly what "uniform" means visually. Plot it, and you get a perfect rectangle.

**CDF:**
$$F(x) = \begin{cases} 0 & x < a \\ \dfrac{x-a}{b-a} & x \in [a,b] \\ 1 & x > b \end{cases}$$

**Mean and Variance:**
$$\text{Mean} = \frac{1}{2}(a+b) \qquad \text{Median} = \frac{1}{2}(a+b) \qquad \text{Variance} = \frac{1}{12}(b-a)^2$$

**Worked example.** The number of candies sold daily at a shop is uniformly distributed with a minimum of 10 and a maximum of 40.

**Question 1: probability of daily sales falling between 15 and 30?**

The general trick for any range question on a uniform distribution: (width of your range) × (constant height 1/(b-a)).
$$Pr(15 \leq X \leq 30) = (x_2 - x_1) \times \frac{1}{b-a} = (30-15) \times \frac{1}{30} = 0.5 \;(50\%)$$

**Question 2: probability that sales exceed 20?**
$$Pr(X > 20) = (40-20) \times \frac{1}{30} = 0.66 \;(66\%)$$

### 11.2 Discrete Uniform Distribution

> A **discrete uniform distribution** describes a **finite** number of values that are all equally likely to occur, each with probability 1/n.

**Example — rolling a fair die.** X = {1, 2, 3, 4, 5, 6}. Here, n (the number of possible outcomes) = b - a + 1 = 6 - 1 + 1 = 6, and every outcome has probability 1/6:
$$Pr(1) = Pr(2) = Pr(3) = \cdots = \frac{1}{6}$$

**Notation:** U(a, b)

**Parameters:** a, b where b ≥ a

**PMF:** 1/n (constant for every valid outcome)

**Mean and Median:**
$$\text{Mean} = \frac{a+b}{2} \qquad \text{Median} = \frac{a+b}{2}$$

---

## Chapter 12: Log-Normal Distribution

> A **log-normal distribution** is a continuous probability distribution of a random variable **whose logarithm is normally distributed**. In other words: if X is log-normally distributed, then Y = ln(X) follows a normal distribution. Flip it around: if Y is normally distributed, then X = exp(Y) is log-normally distributed.

$$X \sim \text{Log-Normal}(\mu, \sigma) \implies Y \sim \ln(X) = \text{Normal Distribution}$$

**Visually:** a log-normal distribution is **right-skewed** — it rises sharply, peaks early, then has a long tail stretching out to the right. This is very different from the perfectly symmetric bell of a normal distribution.

**Why the log-transform matters in practice:** if you take a right-skewed dataset and apply the natural log to every value, the resulting transformed data often ends up looking beautifully normal — which is extremely useful, since a lot of statistical tools (like the Empirical Rule, or models that assume normality) only work cleanly on normal data. This "un-skewing via log" is a common preprocessing trick.

$$\text{Log-Normal Distribution} \xrightarrow{\ln(x)} \text{Normal Distribution}$$

You can also verify this transformation worked using a QQ plot on the transformed (logged) data — if it lines up straight, the log-transform successfully normalized it.

**Real-world examples of naturally log-normal data:**
1. **Wealth distribution of the world** — a small number of people (Bill Gates, Adani, Elon Musk) sit far out on the long right tail, while most people cluster at lower wealth levels
2. **Discussion forums** — length of comments (most comments are short, a few are very long)
3. **Length of chess games**
4. **Dwell time on online articles** (jokes, news)
5. **Salaries of employees in a company** — most cluster around a typical range, with a long tail of high earners stretching the distribution rightward

---

## Chapter 13: Power Law Distribution

> In statistics, a **power law** is a functional relationship between two quantities, where a relative change in one quantity produces a *proportional* relative change in the other — regardless of the initial size of those quantities. In short: **one quantity varies as a power of another.**

**Visually:** power-law distributions look like a steep drop from a high peak near zero, followed by a long, thin tail — very different from the normal distribution's symmetric bell.

This shape is exactly what's behind the famous **80-20 rule**: a small fraction of "causes" is responsible for a large fraction of "effects."

**Real-world examples:**
1. **IPL (cricket)** — roughly 20% of teams are responsible for winning 80% of matches
2. **Wealth** — 80% of a country's wealth is typically distributed among just 20% of the population
3. **Oil** — 80% of a nation's total oil reserves are typically held by 20% of nations
4. **Word frequency** — in most languages, a small set of words (like "the," "and," "a") account for the vast majority of word usage
5. **Software defects** — fixing the top 20% of major bugs typically resolves 80% of the upcoming defects in a product

**Connecting back to earlier chapters:** power-law data is right-skewed, just like log-normal data — and applying **ln(x)** to it can pull it toward a more normal-looking shape too. There's also a more general technique called the **Box-Cox transformation**, which is a broader family of transformations (log being one specific case) used to convert non-normal, skewed data into something closer to normal — again checkable via a QQ plot afterward.

$$\text{Power Law (skewed)} \xrightarrow{\text{Box-Cox Transform}} \text{Approximately Normal}$$

---

## Chapter 14: Pareto Distribution

> The **Pareto distribution**, named after the Italian economist Vilfredo Pareto, is a **power-law probability distribution** used to describe social, quality-control, geophysical, actuarial, and many other observable phenomena. It was originally created to describe the distribution of wealth in a society — capturing the pattern that a large portion of wealth is held by a small fraction of the population.

In other words: **Pareto is the specific, formalized power-law distribution behind the 80-20 rule.** Every Pareto distribution is a power-law distribution, though not every power-law distribution is exactly a Pareto distribution — but in practice, people often use the two terms in the same breath.

**This is a non-Gaussian distribution** — it does not have the symmetric bell shape of a normal distribution at all.

**Shape:** the Pareto PDF starts very high near the minimum value and then decays steeply, with the shape controlled by a parameter **α (alpha)**. As α increases, the curve becomes steeper (more concentrated near the low end, thinner tail).

**Just like Power Law and Log-Normal data, Pareto-distributed data is heavily right-skewed** — and the same fix applies: a log transform (or a Box-Cox transformation more generally) can pull a Pareto-shaped dataset toward something closer to a normal distribution, which you can then verify visually with a QQ plot.

$$\text{Pareto (right-skewed)} \xrightarrow{\text{log transform / Box-Cox}} \text{Approximately Normal}$$

**Real-world example — the IT industry:**
1. 80% of an entire project's work tends to get completed by just 20% of the team
2. 80% of upcoming defects in a product can often be resolved by fixing just the top 20% of underlying bugs

---

## Chapter 15: Central Limit Theorem

This is one of the most quietly powerful ideas in all of statistics — it's the reason the Normal distribution shows up *everywhere*, even when the underlying data itself isn't normal at all.

The theorem relies on a concept called a **sampling distribution** — the probability distribution you get when you take a large number of samples from a population and compute some statistic (like the mean) for each one.

> **The Central Limit Theorem (CLT) says: the sampling distribution of the mean will always be approximately normally distributed, as long as the sample size is large enough — regardless of whether the original population itself follows a normal, Poisson, binomial, or any other distribution at all.**

This is the surprising part: it doesn't matter what shape your original population data has. Even if it's wildly skewed (like a power-law or log-normal shape), the *distribution of sample means* taken from it will still end up looking normal.

### 15.1 How This Actually Works

Say you have some population data X ~ N(μ, σ) — or really, any distribution at all, normal or not. Take repeated samples from it:

$$S_1 = \{x_1, x_2, x_4, ... x_n\} \rightarrow \bar{x}_1$$
$$S_2 = \{x_2, x_3, ... x_n\} \rightarrow \bar{x}_2$$
$$S_3 \rightarrow \bar{x}_3 \quad ... \quad S_m \rightarrow \bar{x}_m$$

Each sample S₁, S₂, ..., Sₘ gives you one sample mean (x̄₁, x̄₂, ..., x̄ₘ). Now, instead of looking at the original data, plot the **distribution of all these sample means** together. That new distribution — the **sampling distribution of the mean** — comes out looking Gaussian/Normal, centered around the true population mean.

This holds true **even when the original population is decidedly not normal** (e.g., right-skewed like a log-normal or power-law shape) — take enough samples, average each one, and the *distribution of those averages* still converges toward a bell curve.

### 15.2 The Practical Rule of Thumb

$$n \geq 30 \implies \text{sample size is "large enough" for CLT to kick in reliably}$$

This n ≥ 30 threshold is the commonly used rule of thumb in practice — below that, the sampling distribution of the mean might not yet look convincingly normal, especially if the original population is heavily skewed.

### 15.3 The Formula for the Sampling Distribution

Once CLT applies, the sampling distribution of the mean follows:

$$\bar{X} \sim N\left(\mu, \frac{\sigma}{\sqrt{n}}\right)$$

Where:
- **μ** = the population mean (the sampling distribution centers on the exact same mean as the original population)
- **σ** = the population standard deviation
- **n** = the sample size

Notice the spread shrinks as n grows — the term σ/√n means that as you take bigger samples, your sample means cluster more and more tightly around the true population mean. This is exactly *why* larger samples give you more reliable estimates: individual sample means bounce around less once n is large.

**Why this theorem is such a big deal:** it's the theoretical foundation underneath confidence intervals, hypothesis testing, z-tests, and t-tests. Without CLT, you couldn't justify treating a sample mean as approximately normal — and a huge fraction of classical statistics leans on exactly that assumption.

---

## Chapter 16: Estimates

Once you've collected a sample, the whole point is usually to *estimate* something you don't actually know about the full population. An **estimate** is how you formalize that guess.

> An **estimate** is a specified, observed numerical value used to estimate an unknown population parameter.

There are two types:

### 16.1 Point Estimate

> A **point estimate** is a single numerical value used to estimate an unknown population parameter.

**Example:** the sample mean (x̄) is a point estimate of the population mean (μ). If you calculate a sample mean of 60, that single number 60 is your best single guess at the true (unknown) population mean.

Picture it as two circles — a big one representing the true population mean μ (say, at 65) and a small one representing your point estimate x̄ (say, at 60). The point estimate is your one specific "arrow" aimed at the unknown target — it might be close, it might be a bit off, but it's a single committed number.

**The catch with point estimates:** they give you no sense of how confident you should be, or how much uncertainty is involved. A single number like "60" doesn't tell you whether the true answer is almost certainly close to 60, or could plausibly be anywhere from 40 to 80. That gap is exactly what the next tool addresses.

### 16.2 Interval Estimate

> An **interval estimate** is a *range* of values used to estimate an unknown population parameter — instead of committing to one single number, you give a range you're reasonably confident contains the true value.

**Example:** instead of saying "the population mean is 60," you say "the population mean likely falls somewhere between 55 and 65." That range [55, 65] is your interval estimate, and it's built around your point estimate sitting somewhere inside it.

This range is more formally known as a **confidence interval** — and it directly reflects the uncertainty that a bare point estimate hides. A narrower interval means more precision/confidence in the estimate; a wider interval means more uncertainty.

**Point estimate vs. interval estimate — why both matter:** a point estimate gives you a clean, specific number to work with (easy to communicate, easy to plug into further calculations), while an interval estimate honestly communicates the *uncertainty* around that number — which matters enormously in real decision-making. Reporting "the average customer satisfaction score is 7.2" sounds precise, but "we're 95% confident the true average is between 6.8 and 7.6" tells the full, honest story.

---

## Quick Recap — The Whole Guide in One Table

| Concept | One-line takeaway |
|---|---|
| Probability | Likelihood of an event, always between 0 and 1 |
| Mutually exclusive events | Can't happen at the same time (e.g., Heads and Tails on one toss) |
| Addition Rule (mutually exclusive) | Pr(A or B) = Pr(A) + Pr(B) |
| Non-mutually exclusive events | Can overlap (e.g., King of Hearts is both King and Heart) |
| Addition Rule (non-mutually exclusive) | Pr(A or B) = Pr(A) + Pr(B) − Pr(A and B), subtracting the double-counted overlap |
| Independent events | One event doesn't affect the other (e.g., two separate coin tosses) |
| Multiplication Rule (independent) | Pr(A and B) = Pr(A) × Pr(B) |
| Dependent events | One event changes the odds of the other (e.g., drawing cards without replacement) |
| Conditional probability | Pr(B \| A) — probability of B, given that A already happened |
| Multiplication Rule (dependent) | Pr(A and B) = Pr(A) × Pr(B \| A) |
| PMF | Discrete random variable — exact probability of one specific value |
| PDF | Continuous random variable — probability read as area under the curve |
| CDF | Pr(X ≤ x) — the running cumulative total; PDF is its slope, CDF is the PDF's running area |
| Bernoulli | A single yes/no trial (p = success, q = 1-p) |
| Binomial | Number of successes across n repeated independent Bernoulli trials |
| Poisson | Number of events occurring in a fixed time interval, given rate λ |
| Normal / Gaussian | Symmetric bell curve — mean = median = mode; 68-95-99.7 empirical rule |
| Z-score | (x − μ) / σ — how many standard deviations a value sits from the mean; basis of standardization |
| Uniform | Every outcome equally likely — flat probability across a range (continuous or discrete) |
| Log-Normal | Right-skewed data whose logarithm is normally distributed |
| Power Law | One quantity scales as a power of another — basis of the 80-20 rule |
| Pareto | The classic power-law distribution behind wealth/effort concentration (80-20 rule) |
| Central Limit Theorem | Sample means become approximately normal as sample size grows (n ≥ 30 rule of thumb), regardless of the population's original shape |
| Point Estimate | A single number used to estimate an unknown population parameter (e.g., sample mean) |
| Interval Estimate | A range (confidence interval) used to estimate an unknown population parameter, capturing uncertainty a point estimate hides |

**The pattern that ties the whole guide together:** every tool here exists to turn "I don't know exactly what will happen" into a number you can actually work with — whether that's the probability of a single event, the shape an entire variable's randomness takes, or how confident you can be in a guess made from limited data.
