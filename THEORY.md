# Model, mathematics, and verification

The simulator compares the Deep Sleep and REM minutes and number of awakenings under a fitted baseline and an intervention that doubles the probability of moving from sleep to Wake. Each epoch represents 30 seconds.

## Four-state Markov chain

Let $X_t$ denote the stage at epoch $t$, and let $S=(W,L,D,R)$ denote Wake, Light, Deep, and REM in the order used by the matrix. The chain is first-order and time-homogeneous. The current stage determines the next-stage distribution, and the same row applies throughout the night.

$$
P_{ij} = \Pr(X_{t+1}=j \mid X_t=i),\qquad P_{ij}\ge 0,\qquad \sum_{j\in S}P_{ij}=1.
$$

For source stage $i$ and destination $j$, $C_{ij}$ counts observed adjacent, valid epochs within training recordings. The empirical estimate is $\widehat P_{ij}=C_{ij}/n_i$, where $n_i=\sum_j C_{ij}$ is the number of outgoing training transitions from $i$. An empty training row triggers an error. The following matrix shows probabilities rounded to six decimals. The implementations use the unrounded values.

| Current stage | Wake | Light | Deep | REM | Outgoing count |
|---|---:|---:|---:|---:|---:|
| Wake | 0.935719 | 0.062624 | 0.000331 | 0.001325 | 3,018 |
| Light | 0.015582 | 0.928768 | 0.037630 | 0.018020 | 9,434 |
| Deep | 0.003815 | 0.144977 | 0.850784 | 0.000424 | 2,359 |
| REM | 0.011268 | 0.042567 | 0.000000 | 0.946166 | 3,195 |

The zero estimate for REM to Deep represents zero observed training transitions in that cell. The initial distribution $\pi=(1,0,0,0)$ comes from one starting epoch per training window. All 18 starts were Wake. Epoch zero represents the beginning of the preprocessed window, which can precede sleep onset. See [data preparation](data/README.md).

## Awakening multiplier

For multiplier $m$ and a sleep row $i\in\{L,D,R\}$, let $p_i=\widehat P_{iW}$ be the fitted probability of moving to Wake. The intervention sets

$$
P^{(m)}_{iW}=mp_i,\qquad
P^{(m)}_{ij}=\widehat P_{ij}\frac{1-mp_i}{1-p_i}\quad(j\ne W).
$$

The factor redistributes the remaining mass in proportion to the original non-Wake probabilities. The Wake row stays fixed, and $m=1$ reproduces the baseline. The valid range for this fit is $1\le m\le 1/\max_i p_i=9434/147\approx64.18$. The input must be finite. The implementations guard against floating-point overshoot of one at the upper limit. An experiment with $m=2$ doubles the conditional sleep-to-Wake probabilities. The resulting number of awakenings also depends on which stages the chain visits.

## Sampling and measured outcomes

The notebook has a readable single-night simulator and NumPy implementations. In the batched simulator, the first uniform draw samples $\pi$. Each subsequent draw samples the current stage’s transition row. Given a draw $u$ in $[0,1)$, the selected stage is the first whose cumulative probability exceeds $u$. The JavaScript demo uses the same cumulative sampling rule. With eight hours at 30 seconds per epoch, a path has $T=960$ epochs.

Writing $\mathbf 1(\cdot)$ for an indicator, stage $j$ contributes $Y_j=0.5\sum_{t=0}^{T-1}\mathbf 1(X_t=j)$ minutes. The awakening count is $A=\sum_{t=1}^{T-1}\mathbf 1(X_{t-1}\in\{L,D,R\},X_t=W)$. Initial Wake and Wake-to-Wake transitions do not contribute.

## Paired Monte Carlo and confidence intervals

The 100-pair pilot precedes the 10,000-pair final experiment. The final simulation spawns one NumPy child seed per pair from master seed 4042 and processes pairs in batches of 500. The two arms share the same initial distribution, length, and uniform draws within each pair. Distinct pairs have separate child streams. Both arms use the same sampling code and differ only in the transition matrix.

For each outcome, $d_r$ is the multiplier-2 value minus the baseline value in pair $r$. The estimate is the mean paired difference $\bar d$. With $R$ pairs and sample standard deviation $s_d$ calculated with $R-1$ in its denominator, the 95% interval is

$$
\bar d \ \pm\ t_{0.975,R-1}\frac{s_d}{\sqrt R}.
$$

Here $t_{0.975,R-1}$ is the 97.5th percentile of Student’s t distribution with $R-1$ degrees of freedom. The final experiment uses 9,999 degrees of freedom. The intervals quantify Monte Carlo uncertainty for the fitted model. They do not incorporate uncertainty in the recordings, fitted probabilities, or assumptions. See the [final results](README.md#results).

## Verification and held-out validation

| Check | Evidence |
|---|---|
| Data and fitting | Stage mapping, exclusion of missing and non-consecutive pairs, participant separation, reproducible split, probability bounds and normalisation, and one training start per recording. |
| Sampling | Valid stage labels and 960 epochs per eight-hour night. Stage durations sum to 480 minutes. A 20,000-draw Light-row check uses a one-percentage-point tolerance. |
| Intervention | Identity at multiplier 1, doubled sleep-to-Wake entries at multiplier 2, unchanged Wake row and baseline matrix, valid row sums, and invalid-input checks. |
| Reproducibility and inference | Same-seed exact replay for stored final outcomes, documented batch-size checks, and paired intervals checked against SciPy. |
| Browser parity | The [fixed-draw test](USER_GUIDE.md#check-against-the-notebook) matched all 960 stages and all three metrics at each tested multiplier. |

The six held-out recordings from three separate participants supplied 6,901 transitions. The largest absolute held-out minus training difference was 2.21 percentage points for REM persistence. Sleep-to-Wake differences were +0.27, +0.33, and +0.62 percentage points for Light, Deep, and REM respectively. These checks compare one-step conditional frequencies. Validation of complete-night dynamics and population-level inference remain open. See the [references](REFERENCES.md) for the dataset, mathematical background, and related sleep models.
