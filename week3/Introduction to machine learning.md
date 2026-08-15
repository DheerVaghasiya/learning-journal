# Introduction to Machine Learning

**Learning Journal · Machine Learning Fundamentals**

Everything I need to actually understand before touching a single algorithm — what ML even is, how it's split up, the geometry hiding behind almost every model, and the one distinction (instance-based vs model-based) that explains why some algorithms are slow and dumb, while others are fast and generalize.

`Topic 01 · ML Fundamentals` · `Source: Krish Naik Udemy course` · `Level: Absolute beginner`

## In this note

1. [AI vs ML vs DL vs DS — what's actually different](#chapter-one-ai-vs-ml-vs-dl-vs-ds--whats-actually-different)
2. [The three types of Machine Learning techniques](#chapter-two-the-three-types-of-machine-learning-techniques)
3. [Equation of a Line, a Plane, and a Hyperplane](#chapter-three-equation-of-a-line-a-plane-and-a-hyperplane)
4. [Distance of a point from a plane](#chapter-four-distance-of-a-point-from-a-plane)
5. [Instance-based vs Model-based learning](#chapter-five-instance-based-vs-model-based-learning)

---

## Chapter One: AI vs ML vs DL vs DS — what's actually different

These four terms get thrown around like they're interchangeable, but they're not siblings — they're more like nested boxes, one sitting inside the other. Getting this hierarchy straight early saves a lot of confusion later, because "AI" is a goal, "ML" is a strategy to get there, "DL" is one specific technique inside that strategy, and "DS" is the toolbox you use to make sense of the data along the way.

#### Artificial Intelligence (AI)

AI is the *outcome* we want: building an application that can perform its own task without needing constant human intervention. It's the umbrella goal — "make the machine act intelligently" — and it doesn't say anything about *how* you get there. You could technically hardcode a thousand if-else rules and call it AI (old-school AI actually worked like this).

#### Machine Learning (ML)

ML is one way — currently the dominant way — of achieving AI. Instead of hardcoding rules, you feed the machine data and let it find the patterns itself. The name is literal: the machine learns from examples instead of being explicitly programmed for every case.

#### Deep Learning (DL)

DL is a subset of ML that uses **multi-layered neural networks** — structures loosely inspired by how neurons connect in the human brain. It's not a separate philosophy from ML; it's ML pushed through architectures that stack many layers of computation, which is what lets it handle messy, high-dimensional data like images, audio, and language really well.

#### Data Science (DS)

DS is a bit different from the other three — it's less about building intelligent systems and more about the **statistical toolkit**: analyzing, visualizing, predicting, and forecasting from data. A lot of ML work actually leans on data science skills (EDA, statistics, visualization) before a single model gets trained.

> **Fig 1.1** — AI is the outer goal; ML is the dominant strategy inside it; DL is a technique inside ML; DS overlaps both as the analytical toolkit. (Nested circles: AI containing ML, ML containing DL, with DS as a separate circle overlapping ML/DL since both lean on statistics, analysis & visualization.)

**Worked example**
Take Netflix's recommendation system. The **AI** goal is "recommend movies without a human curating them for every user." The **ML** part is a model trained on your watch history — say you watch a lot of action movies — that learns the pattern and recommends more action movies. If that model happens to be a deep neural network processing your entire viewing sequence, that's the **DL** layer. And the dashboards Netflix's analysts use to spot trends in what people are watching? That's **DS**.

---

## Chapter Two: The three types of Machine Learning techniques

Once you're inside "ML," there are three fundamentally different ways a machine can learn, and the difference comes down to one question: **does your data have labels or not, and is there a reward signal instead?**

> **Fig 2.1** — The three branches of ML, split by what kind of feedback the model gets: **Supervised ML** (labeled data), **Unsupervised ML** (unlabeled data), **Reinforcement Learning** (reward-based).

### 1. Supervised Machine Learning

In supervised learning, every row of your dataset already comes with the "answer" attached. You're not asking the model to discover something from scratch — you're giving it examples of inputs paired with correct outputs, and it learns the mapping between them.

Every dataset here splits into two kinds of columns:

- **Independent features** — the inputs. These are what you already know and use to make a prediction.
- **Dependent feature / output feature** — the thing you're trying to predict. This column decides whether the problem is regression or classification:
  - If it's **continuous** (any numeric value, like price) → it's a **Regression** problem.
  - If it's **categorical** (a fixed set of labels, like Pass/Fail) → it's a **Classification** problem.

#### Regression — predicting a number

**Worked example — predicting house price**

| Size of house | # of Rooms | Price |
|---|---|---|
| 5000 | 5 | 450K |
| 6000 | 6 | 500K |

Size and rooms are the **independent features**. Price is the **dependent feature** — and since price can be any continuous number, this is a regression problem.

> **Fig 2.2** — Regression: the output is continuous, so the model fits a best-fit line/curve through the scattered points (Size of house on x-axis, Price on y-axis).

#### Classification — predicting a category

**Worked example — pass or fail**

| No. of study hours | No. of play hours | Pass/Fail |
|---|---|---|
| 7 | 3 | Pass |
| 2 | 6 | Fail |

Here the output is categorical (Pass or Fail), so this is classification. Specifically it's **binary classification** because there are only two possible classes. If there were more than two classes (say Pass / Fail / Distinction), it would be called **multiclass classification**.

### 2. Unsupervised Machine Learning

Here, there's no answer column at all — the data is unlabeled. The model isn't told what the "correct" output is; it's just asked to find structure in the data on its own. The most common thing it does with that freedom is **clustering** — grouping similar data points together.

**Worked example — customer segmentation**
An e-commerce company has customer data with **Salary** and **Spending Score (1–10)**, but no label telling it which "type" of customer each row is. Feed this into an unsupervised algorithm and it naturally separates customers into clusters — say, low salary/high spending, high salary/low spending, and so on. The company can then target the "low salary, high spending" cluster with a discount email, since they're price-sensitive but already engaged shoppers.

> **Fig 2.3** — Unsupervised clustering: no labels given, the algorithm discovers the groups itself. Three natural clusters emerge on a Salary (x-axis) vs Spending (y-axis) plot: low salary/high spending, mid/mid, and high salary/low spending.

### 3. Reinforcement Learning

The third branch works completely differently from the other two — there's no fixed dataset of labeled examples at all. Instead, an **agent** interacts with an **environment**, takes actions, and gets **rewards or penalties** based on how good those actions were. Over time it learns a strategy (a "policy") that maximizes reward. Think of training a dog: no one labels every situation, but good behavior gets a treat and bad behavior doesn't — repeated enough times, the dog learns the pattern.

### Common algorithms in each bucket

**Supervised ML**
Linear Regression · Ridge & Lasso · ElasticNet · Logistic Regression (classification) · Decision Tree · Random Forest · AdaBoost · XGBoost

The last four (Decision Tree onward) can each be used for *both* classification and regression.

**Unsupervised ML**
K-Means · Hierarchical Mean (Clustering) · DBSCAN Clustering

---

## Chapter Three: Equation of a Line, a Plane, and a Hyperplane

This feels like pure math at first, but it's actually the backbone of a huge chunk of ML — logistic regression, SVMs, and neural network layers are all, underneath, drawing a line, a plane, or a "hyperplane" to separate or fit data. So it's worth building this up slowly from something you already know.

### The 2D line you already know

You've seen this since school:

```
slope-intercept form
y = mx + c
```

where **m** is the slope (how steep the line is) and **c** is the y-intercept (where the line crosses the y-axis, i.e. the value of y when x = 0). In ML notation you'll often see the exact same line written as `y = β₀ + β₁x` — β₀ is just c, and β₁ is just m. Same line, statistics dialect.

> **Fig 3.1** — y = mx + c: c is where the line starts (the intercept, at x=0), m is how fast y climbs for every step in x (rise / run = m).

### Rewriting it as a general equation

The slope form is convenient for plotting, but ML prefers a more symmetric form because it generalizes cleanly to higher dimensions. Start from the general line equation and rearrange back to slope form to see they're the same thing:

```
ax + by + c = 0
⇓ (solve for y)
by = −ax − c
y = (−a/b)·x − (c/b)
```

Compare that to `y = mx + c` — you can see `m = −a/b` and the intercept term is `−c/b`. It's the same line, just rearranged. This "everything equals zero" form (`ax + by + c = 0`) is the one that scales up nicely.

### Moving to a plane in 3D

Add a third axis (x₃) and a line becomes a flat **plane**. The equation just gets one more term:

```
w₁x₁ + w₂x₂ + w₃x₃ + b = 0
```

Notice the variable names changed from (a, b, c) and (x, y) to **weights** (w₁, w₂, w₃) and **features** (x₁, x₂, x₃), with **b** as the bias/intercept. This is deliberate — it's the exact same idea, but now written the way you'll see it in every ML paper and every line of model code.

> **Fig 3.2** — A plane in 3D space (axes x₁, x₂, x₃): one weight per axis, plus a bias term. `w₁x₁ + w₂x₂ + w₃x₃ + b = 0`

### Generalizing to n dimensions — the Hyperplane

Real datasets don't stop at 3 features — you might have 50, or 500. You obviously can't draw a 500-dimensional picture, but the equation keeps extending exactly the same way, one more `wᵢxᵢ` term per feature:

```
w₁x₁ + w₂x₂ + w₃x₃ + … + wₙxₙ + b = 0
```

Written in **vector form**, this collapses beautifully into one compact expression that you'll see everywhere in ML:

```
the hyperplane equation
wᵀx + b = 0
```

where:

```
w = [w₁, w₂, w₃, …, wₙ]ᵀ   (the weight vector)
x = [x₁, x₂, x₃, …, xₙ]ᵀ   (the feature vector)
```

A "hyperplane" is just the general name for this flat surface at any dimension: in 2D it's a line, in 3D it's a plane, and from 4D onward we just call it a hyperplane because we run out of names. **This single equation, wᵀx + b = 0, is the mathematical object that a huge number of ML algorithms are trying to find** — the right w and b that best separate or fit your data.

### A special case: passing through the origin

If the bias b = 0, the hyperplane is forced to pass through the origin (0,0,...,0), and the equation simplifies to:

```
wᵀx = 0
```

### Why w is always perpendicular to the hyperplane

This is one of those facts that seems random until you see the dot product behind it. Recall that the dot product of two vectors can be written two ways:

```
w · x = wᵀx = ‖w‖ ‖x‖ cos θ
```

On the hyperplane we just said `wᵀx = 0`. Since ‖w‖ and ‖x‖ are just lengths (not zero for any real point), the only way the whole right-hand side can equal zero is if `cos θ = 0` — which happens exactly when **θ = 90°**. In other words, for every single point x that lies on the hyperplane, the vector w makes a 90° angle with it. That's precisely the definition of "perpendicular." So the weight vector w is always the **normal vector** to the hyperplane — it points straight out of the surface.

> **Fig 3.3** — The weight vector w always sticks straight out of the hyperplane, at exactly 90° (w ⊥ Π), where the plane Π is defined by wᵀx = 0, anchored at the origin.

---

## Chapter Four: Distance of a point from a plane

Now that we know a hyperplane is `wᵀx + b = 0` and that w points straight out of it, the next natural question is: given some point that's *not* on the plane, how far away is it? This matters a lot in ML — it's exactly how algorithms like SVM decide which side of a decision boundary a point falls on, and how confidently.

### Building the intuition

Picture the hyperplane as a flat wall, and w as an arrow poking straight out of it. Any point in space can be reached by walking along that same direction w, starting from the plane. The "distance from the plane" is just: how far do you have to walk along the direction of w to land exactly on your point?

### The formula

For a hyperplane `wᵀx + b = 0` and a point x₀ that may or may not lie on it, the perpendicular distance from x₀ to the plane is:

```
distance of a point from a plane
d = |wᵀx₀ + b| / ‖w‖
```

Breaking that down piece by piece:

- **wᵀx₀ + b** — plug the point's coordinates straight into the plane's equation. If the point is actually on the plane, this equals exactly 0 (that's the definition of the plane). The further off the plane the point is, the bigger this value gets.
- **‖w‖** — the length (magnitude) of the weight vector. Dividing by it "normalizes" the raw value above into an actual distance in real units, instead of a value that depends on how large your w happened to be.
- **| · |** — the absolute value, because a point can sit on either side of the plane (giving a positive or negative result from wᵀx₀ + b), but distance itself is never negative.

**Where this shows up later**
Support Vector Machines are literally built on this formula — they search for the w and b that *maximize* this distance (the "margin") between the hyperplane and the nearest points of each class. Understanding this formula now means SVM will feel like plugging into something familiar instead of learning something brand new.

> **Fig 4.1** — Walking from the plane along the direction of w until you reach x₀ — that walk's length is d = |wᵀx₀ + b| / ‖w‖.

---

## Chapter Five: Instance-based vs Model-based learning

Every supervised ML algorithm fits into one of two philosophies about *how* it learns from training data, and this distinction explains a lot of practical differences you'll run into later — why some models are instant to train but slow to predict, and others are slow to train but instant to predict.

### The core split: memorizing vs generalizing

Zoom out for a second: at the highest level, "learning" itself splits into **memorizing** and **generalizing**.

- **Memorizing** → the strategy behind **Instance-based learning**.
- **Generalizing** → the strategy behind **Model-based learning**.

### Instance-based learning

The algorithm doesn't build any abstract "model." It just keeps the entire training dataset around, and when a new query comes in, it looks directly at the stored data to answer. Nothing is generalized in advance — the pattern discovery is postponed until the moment a new scoring query actually arrives.

```
DATA → Output   (no model in between)
```

The classic example is **KNN (K-Nearest Neighbours)**: to classify a new point, it just looks at the K closest points already in the training data and takes a majority vote among them. It's often called "domain expert" style learning — like memorizing the answer key religiously rather than actually understanding the subject, then looking up the closest matching question when the test comes.

> **Fig 5.1** — Instance-based (KNN): a new point is classified by checking who its nearest neighbours are, using the raw training data directly. A query point (Test) is surrounded by a circle; the K nearest × marks inside the circle are used to vote on the class (Study hours vs Play hours plot).

### Model-based learning

This is the more familiar "usual" ML workflow. The algorithm looks at the training data, discovers the underlying **pattern**, and compresses that pattern into a **generalized model** — a fixed mathematical form (like a set of weights) — before it ever sees a new query. Once that generalized model exists, the raw training data isn't needed anymore for future predictions.

```
DATA → Pattern → Generalization Method → Generalized Model
```

> **Fig 5.2** — Model-based learning: the pattern gets baked into one clean decision boundary (the model) that generalizes to any future point instantly — separating a PASS region from a FAIL region.

### Side-by-side comparison

| Usual / Conventional ML (Model-based) | Instance-based Learning |
|---|---|
| Prepare the data for model training | Prepare the data for model training — no difference here |
| Trains a model from the training data to estimate parameters, i.e. discovers patterns | Does not train a model. Pattern discovery is postponed until a scoring query is actually received |
| Stores the model in a suitable form | There is no model to store |
| Generalizes the rules into a model, even before any scoring instance is seen | No generalization happens beforehand — it generalizes individually for each query, only when seen |
| Predicts an unseen instance using the model | Predicts an unseen instance using the training data directly |
| Can discard the input/training data after training the model | Must keep the input/training data around — each query needs part or all of it |
| Requires a known model form | May not have any explicit model form |
| Storing the model generally needs less storage | Storing all the training data generally needs more storage |
| Scoring a new instance is generally fast | Scoring a new instance can be slow, since it re-checks the training data every time |

**One-line way to remember it**
Model-based learning does the hard work *once*, upfront, and then answers new questions instantly. Instance-based learning skips the hard work upfront, and instead redoes a little bit of it every single time a new question shows up.

---
