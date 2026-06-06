# Predicting Recipe Popularity on Food.com

### Why a recipe's rating reflects attention, not the recipe itself

**By Aryan Agarwal** — a project for DSC 80 at UC San Diego.

We assume a recipe earns a good rating because it is a good recipe. Using
83,782 recipes and their reviews from [food.com](https://food.com), this
project asks whether that is true — or whether a recipe's rating is really a
mirror of how much **attention** (how many reviews) it receives.

---

## Introduction

This project uses recipes and reviews scraped from food.com (the subset posted
since 2008). The recipes table has **83,782 rows**, one per recipe. After
merging in the reviews, the columns central to our question are the recipe's own
attributes (`n_steps`, `minutes`, `n_ingredients`, and the parsed `nutrition`
fields), its average rating (`recipe_avg_rating`), and our measure of attention,
`n_reviews` — the number of ratings a recipe received.

**Central question: do a recipe's own attributes predict how well it is rated,
or is its rating really driven by how much attention it gets?**

## Data Cleaning and Exploratory Data Analysis

We left-merge recipes with their reviews, treat ratings of 0 as missing (the
star scale starts at 1, so a 0 marks an absent rating), and attach each recipe's
average rating and review count. We parse the `nutrition` string into seven
numeric columns and clip the wildly skewed `minutes` column (its maximum is
about two years) at the 99th percentile for plotting.

Two findings drive the whole project. First, **every recipe attribute has
near-zero correlation with rating** — steps, time, ingredients, and all nutrition
fields sit between -0.01 and +0.01. The only feature that moves rating is the
number of reviews.

<div>
<iframe src="assets/fig_corr.html" width="800" height="450" frameborder="0" style="border:none;max-width:100%"></iframe>
</div>

Second, **rating rises steadily with the number of reviews**: recipes with one
review average 4.57, while recipes with 11+ reviews average 4.74.

<div>
<iframe src="assets/fig_rating_by_reviews.html" width="800" height="450" frameborder="0" style="border:none;max-width:100%"></iframe>
</div>

The average rating itself is heavily skewed toward 5 stars, which is why we later
frame prediction as classification rather than regression.

<div>
<iframe src="assets/fig_rating_dist.html" width="800" height="450" frameborder="0" style="border:none;max-width:100%"></iframe>
</div>

This grouped table summarizes the core relationship — average rating climbs with
each review-count bin:

| Number of reviews | Mean rating | Recipes |
| --- | --- | --- |
| 1 | 4.571 | 40,834 |
| 2 | 4.637 | 16,572 |
| 3-5 | 4.694 | 15,572 |
| 6-10 | 4.738 | 5,761 |
| 11+ | 4.743 | 2,434 |

## Assessment of Missingness

**NMAR.** We believe the individual `rating` column is plausibly **NMAR** (Not
Missing At Random). Whether a user leaves a rating likely depends on the rating
they *would* have given — people with strong opinions return to rate, while
indifferent users silently move on. Knowing each user's general engagement (how
often they rate anything) could explain the missingness and make it MAR instead.

**Dependency tests.** Analyzing the missingness of `description` (70 recipes
have none), a permutation test shows its missingness **depends on**
`n_ingredients` (observed difference about 1.47 ingredients, p about 0.002) —
recipes without a description tend to have fewer ingredients — but **does not
depend on** `minutes` (p about 0.58).

<div>
<iframe src="assets/fig_missing.html" width="800" height="450" frameborder="0" style="border:none;max-width:100%"></iframe>
</div>

## Hypothesis Testing

We split recipes into high attention (reviews above the median) and low
attention (at or below), and tested whether attention is associated with a
higher rating.

- **Null hypothesis:** a recipe's average rating does not depend on its
  attention group; high- and low-attention recipes share the same rating
  distribution, and any difference is due to chance.
- **Alternative hypothesis:** high-attention recipes have a higher average
  rating than low-attention recipes.

Using a permutation test (difference in mean rating as the test statistic, 5,000
permutations), high-attention recipes averaged **4.68** vs **4.57** for
low-attention recipes — an observed difference of about **+0.108** with a
p-value of about **0.000**. At the 0.05 level we reject the null hypothesis.
Attention is strongly associated with higher ratings, and the gap dwarfs
anything the recipe's own attributes produced. (This is an association, not
proof of causation.)

<div>
<iframe src="assets/fig_perm_test.html" width="800" height="450" frameborder="0" style="border:none;max-width:100%"></iframe>
</div>

## Framing a Prediction Problem

Because recipe attributes barely predict *rating*, we instead predict
**attention**. We frame a binary classification problem: predict whether a
recipe is "popular" (review count above the median) using only attributes known
at the moment it is published. The classes are roughly balanced (about 48%
popular), so we report both accuracy and F1-score.

## Baseline Model

A logistic-regression baseline using two scaled features (`n_steps`,
`n_ingredients`) reaches only about **0.52 accuracy** and **0.29 F1** — barely
above the base rate. That weakness is itself the point: just two attributes
carry almost no signal about whether a recipe will draw attention.

## Final Model

The final model adds three engineered features — `log_minutes` (log of cook time,
to tame extreme outliers), the nutrition fields `calories`, `sugar`, and
`protein` (a recipe's richness may affect how many people try it), and
`desc_len` (a longer description signals a more carefully presented recipe) —
and switches to a class-balanced `RandomForestClassifier`. Tuning `max_depth`
over [10, 20, None] with `GridSearchCV`-style search, the best depth was **10**.

The final model reaches about **0.54 accuracy** and **F1 about 0.56**, up from
the baseline's **0.29** — close to a 2x gain on the metric we care about. The
improvement is explainable: the engineered features add real signal and the
random forest captures interactions a linear model cannot.

## Fairness Analysis

We asked whether the model predicts popularity equally well for **quick recipes**
(cook time at or below the 35-minute median) versus **slow recipes** (above it),
comparing **precision** across the two groups.

- **Null hypothesis:** the model is fair; precision for quick and slow recipes is
  the same, and any difference is due to chance.
- **Alternative hypothesis:** the model is unfair; precision differs between the
  groups.

Precision was **0.529 for quick recipes** vs **0.502 for slow recipes** — an
observed difference of about **+0.027**, with a two-sided permutation p-value of
about **0.006**. At the 0.05 level we reject the null hypothesis: the model is
significantly better at predicting popularity for quick recipes than slow ones,
likely because quick recipes are far more common in the data. The gap is small
but real.

<div>
<iframe src="assets/fig_fairness.html" width="800" height="450" frameborder="0" style="border:none;max-width:100%"></iframe>
</div>
