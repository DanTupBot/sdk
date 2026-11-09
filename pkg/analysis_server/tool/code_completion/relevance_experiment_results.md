# Code Completion Relevance Tables & Scores Experiment (#48771 / #64611)

This document summarizes the experiment comparing three completion ranking configurations using `pkg/analysis_server/tool/code_completion/completion_metrics.dart`:

1. **`shipping`**: Default behavior (uses all `FeatureComputer` features + generated `elementKindRelevance` and `keywordRelevance` tables in `relevance_tables.g.dart`).
2. **`no tables`**: Disables `elementKindRelevance` and `keywordRelevance` lookups (`useRelevanceTables = false`), while keeping all other relevance features (`contextType`, `localVariableDistance`, `hasDeprecated`, `isConstant`, `isNotImported`, `isNoSuchMethod`, `startsWithDollar`, `superMatches`).
3. **`no relevance`**: Disables `RelevanceComputer.computeRelevance` (`useRelevanceScores = false`, returning `0` for all candidates), so candidates are ranked solely by prefix `matcherScore` (and discovery order on ties).

---

## Executive Summary (48,384 Completions across 4 Packages)

### Part 1: `--prefix-length=0` (Default — 0 characters typed)

#### Overall Inverse MRR (Average Rank — **lower is better**)

| Package / Directory | Files | Completions | `shipping` | `no tables` (delta) | `no relevance` (delta) |
| :--- | ---: | ---: | ---: | ---: | ---: |
| `pkg/dtd/lib` | 10 | 655 | **2.668** | 2.916 (+0.248) | 5.442 (+2.774) |
| `pkg/dds/lib` | 32 | 4,342 | **2.566** | 2.597 (+0.031) | 5.935 (+3.370) |
| `pkg/analysis_server/lib/src/lsp` | 113 | 18,065 | **2.253** | 2.369 (+0.117) | 6.327 (+4.075) |
| `pkg/analyzer/lib/src/dart/resolver` | 42 | 25,322 | **2.186** | 2.241 (+0.054) | 6.304 (+4.117) |

#### Average Characters Typed Before Target Reaches #1 (`chars_before_top`) and Top 5 (`chars_before_top_five`)

| Package / Directory | Metric | `shipping` | `no tables` | `no relevance` |
| :--- | :--- | ---: | ---: | ---: |
| `pkg/dtd/lib` (655) | `chars_before_top`<br>`chars_before_top_five` | 3.11<br>2.01 | 2.92<br>2.01 | 2.36<br>0.78 |
| `pkg/dds/lib` (4,342) | `chars_before_top`<br>`chars_before_top_five` | **2.29**<br>**0.84** | 2.35<br>0.91 | 2.51<br>1.13 |
| `pkg/analysis_server/lib/src/lsp` (18,065) | `chars_before_top`<br>`chars_before_top_five` | **2.36**<br>0.94 | 2.54<br>**0.89** | 2.86<br>1.36 |
| `pkg/analyzer/lib/src/dart/resolver` (25,322) | `chars_before_top`<br>`chars_before_top_five` | **2.19**<br>**0.66** | 2.24<br>0.70 | 2.81<br>1.12 |

---

### Part 2: `--prefix-length=2` (2 characters typed, `FuzzyMatcher` active)

*(Note: Required fixing a bug in `completion_metrics.dart` where `options.prefixLength` was added to `columnNumber` and overlay content, but not to `request.offset`.)*

#### Overall Inverse MRR (Average Rank — **lower is better**)

| Package / Directory | Files | Completions | `shipping` | `no tables` (delta) | `no relevance` (delta) |
| :--- | ---: | ---: | ---: | ---: | ---: |
| `pkg/dtd/lib` | 10 | 655 | **1.228** | 1.229 (+0.000) | 1.391 (+0.163) |
| `pkg/dds/lib` | 32 | 4,342 | 1.249 | **1.242 (-0.007)** | 1.721 (+0.472) |
| `pkg/analysis_server/lib/src/lsp` | 113 | 18,065 | 1.237 | **1.231 (-0.006)** | 2.174 (+0.937) |
| `pkg/analyzer/lib/src/dart/resolver` | 42 | 25,322 | **1.225** | 1.240 (+0.015) | 2.153 (+0.929) |

---

## Key Takeaways

1. **Relevance scores as a whole (`shipping` / `no tables` vs `no relevance`) have a major positive impact, even with a 2-character prefix:**
   - At `prefix-length=0`, removing relevance scores degrades overall Inverse MRR from **~2.2–2.6** to **~5.9–6.3** (and Top-5 Inverse MRR from **~2.3–2.7** to **~9.2–10.6**).
   - Even at `prefix-length=2` (where `FuzzyMatcher` filters and ranks by `matcherScore` first), removing relevance scores degrades overall Inverse MRR from **~1.23** to **~1.72–2.17**, and hurts local variables (`1.06` -> `3.77–4.31`) and parameters (`1.03` -> `5.08–5.63`) significantly because all candidates starting with the same 2 characters tie on `matcherScore`.
   - Non-table features (`contextType`, `localVariableDistance`, `isNotImported`, `hasDeprecated`, `isConstant`, `superMatches`) account for **95%–100%** of the benefit of relevance scoring.

2. **The generated relevance tables (`shipping` vs `no tables`) have almost zero net impact:**
   - At `prefix-length=2`, **`no tables` actually slightly beats `shipping`** on `pkg/dds/lib` (`1.242` vs `1.249`) and `pkg/analysis_server/lib/src/lsp` (`1.231` vs `1.237`), ties on `pkg/dtd/lib` (`1.229` vs `1.228`), and is within `0.015` on `pkg/analyzer/lib/src/dart/resolver` (`1.240` vs `1.225`).
   - At `prefix-length=0`, `shipping` is only marginally ahead of `no tables` overall (`+0.03` to `+0.12` inverse MRR), and much of that difference is an artifact of `inheritanceDistance` being coupled to `elementKindRelevance` (see Caveat #1 below).
   - **Where the relevance tables HURT (worse in `shipping` than `no tables` across both `prefix-length=0` and `prefix-length=2`):**
     - **Local variables** (`prefix=0`: `1.664` vs `1.464` in `lsp`, `1.568` vs `1.372` in `resolver`; `prefix=2`: `1.081` vs `1.066` in `lsp`, `1.063` vs `1.048` in `resolver`).
     - **Parameters** (`prefix=0`: `1.722` vs `1.640` in `lsp`; `prefix=2`: `1.056` vs `1.053` in `lsp`).
     - **Local functions & Import prefixes** (`prefix=0` `local function`: `144.9` vs `9.8` in `lsp`, `768.5` vs `16.4` in `resolver`; `prefix=0` `prefix`: `491.0` vs `302.2` in `lsp`, `788.2` vs `326.6` in `resolver`): Because import prefixes and local functions were rare in the training corpus, the relevance tables assign them near-zero probability (`elementKindFeature ≈ 0.001`), pushing them far down the list.

---

## Important Quirks & Caveats in `completion_metrics.dart` and `FeatureComputer`

1. **Disabling `elementKindRelevance` also disables `inheritanceDistance`!**
   - In `pkg/analysis_server/lib/src/services/completion/dart/feature_computer.dart` (`weightedAverage`) and `relevance_computer.dart` (`RelevanceComputer.computeScore`):
     - `inheritanceDistance` is passed to `computeScore(...)`, which passes it to `listener?.computedFeatures(...)`, **but does NOT pass `inheritanceDistance` to `weightedAverage(...)`!**
     - Instead, `inheritanceDistance` is only used as the `distance` argument to `featureComputer.elementKindFeature(element, location, distance: inheritanceDistance)` (`range.conditionalProbability(distance)`).
     - Consequently, in the `no tables` configuration, returning `0.0` from `elementKindFeature` *also* zeroes out `inheritanceDistance`, which explains why `instance member` is slightly worse in `no tables` (`2.52` vs `3.01` at `prefix=0`). If `inheritanceDistance` is preserved as a standalone feature when tables are removed, `no tables` should perform even better on instance members.
2. **Tie-breaking bias in `Counts for 'relevance rank comparison'`:**
   - `computeSuggestionsAndMetrics` in `completion_metrics.dart` uses strict `<` (`if (bestRank < 0 || rank < bestRank)`), so whenever `shipping` and `no tables` tie at the exact same rank (e.g. rank 1), it credits `shipping` (whichever metric is first in `targetMetrics`).
3. **Why `no relevance` preserves scope order on ties:**
   - `SuggestionCollector.finalize` sorts by `matcherScore` and then `relevanceScore`. When all candidates have `relevanceScore = 0` (`no relevance`), Dart's stable `.sort()` keeps candidates in the order `InScopeCompletionPass` and `NotImportedCompletionPass` discovered them (innermost local scope -> parameters -> enclosing class -> imported libraries -> not-imported libraries).
4. **Why `_filterSuggestions` in `_computeCharsBeforeTop` is an approximation:**
   - `_filterSuggestions` in `completion_metrics.dart` filters the suggestion list using `suggestion.completion.startsWith(prefix)` rather than re-running `FuzzyMatcher` for each additional character typed.
5. **Bug fixed for `--prefix-length > 0` in `completion_metrics.dart`:**
   - Previously, `ExpectedCompletionsVisitor` added `_caretOffset` (`options.prefixLength`) to `columnNumber` and `getOverlayContent` added `prefixLength` when slicing the file for overlays, but `handleExpectedCompletion` passed the unshifted `expectedCompletion.offset` (start of token) to `DartCompletionRequest.forResolvedUnit`. That meant `--prefix-length` with `--overlay=none` was still requesting completion at offset 0 of the token. We fixed `handleExpectedCompletion` to add `options.prefixLength` when `syntacticEntity.length >= options.prefixLength`.

---

## Raw Outputs (`--prefix-length=0`)

### 1. `pkg/dtd/lib` (`--prefix-length=0`, 10 files, 655 completions, 8.5s)
```text
Analyzing root: "/home/ubuntu/projects/dart-sdk/sdk/pkg/dtd/lib"
Computing completions at root: /home/ubuntu/projects/dart-sdk/sdk/pkg/dtd/lib
Resolving 10 files...
Analyzing completion suggestions...

Metrics computed in 0:00:08.516000

# Comparison of experiments

## Comparison of inverse mean reciprocal ranks

A lower value is better, so a negative delta is good.

                        shipping  |  no tables    delta  |  no relevance   delta
all completions            2.668  |      2.916    0.248  |         5.442   2.774
successful completions     2.668  |      2.916    0.248  |         5.442   2.774
                                                                                
class                     17.193  |     20.062    2.868  |        20.069   2.875
extension                 56.382  |     44.141  -12.240  |       118.650  62.268
instance member            3.321  |      3.735    0.414  |         3.972   0.650
local variable             1.299  |      1.064   -0.234  |         6.240   4.941
parameter                  1.339  |      1.284   -0.054  |         6.419   5.080
static member              3.100  |      3.836    0.735  |         4.091   0.991
top level member           1.347  |     16.507   15.160  |        32.485  31.138

## Counts for 'relevance rank comparison' (total = 655)

              count  percent
shipping      543    82.90%
no tables     94     14.35%
no relevance  18     2.75%

## Comparison of other metrics

                                       shipping        no tables     no relevance
ms per completion             0, 4.279389, 2019  0, 0.954198, 22   0, 0.407634, 3
chars_before_top                0, 3.112977, 32  0, 2.916031, 32  0, 2.360305, 18
chars_before_top_five           0, 2.013740, 32  0, 2.013740, 32   0, 0.784733, 3
insertion_length_theoretical    0, 7.103817, 30  0, 7.300763, 30  0, 7.856489, 30

shipping: [0] 652 [10] 2 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 1
no tables: [0] 654 [10] 0 [20] 1 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0
no relevance: [0] 655 [10] 0 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0

## Comparison of completion counts

                   shipping      no tables   no relevance
total                   655            655            655
successful    655 (100.00%)  655 (100.00%)  655 (100.00%)
unsuccessful     0 ( 0.00%)     0 ( 0.00%)     0 ( 0.00%)

# Completion metrics for shipping

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.374780        2.668  0.349440          2.862    655
successful completions  0.374780        2.668  0.349440          2.862    655
                                                                             
class                   0.058162       17.193  0.000000       Infinity     47
extension               0.017736       56.382  0.000000       Infinity     65
instance member         0.301070        3.321  0.274221          3.647    214
local variable          0.770115        1.299  0.764368          1.308     29
parameter               0.747021        1.339  0.742997          1.346    119
static member           0.322543        3.100  0.285992          3.497    163
top level member        0.742340        1.347  0.722222          1.385     18


# Completion metrics for no tables

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.342899        2.916  0.311120          3.214    655
successful completions  0.342899        2.916  0.311120          3.214    655
                                                                             
class                   0.049847       20.062  0.000000       Infinity     47
extension               0.022654       44.141  0.000000       Infinity     65
instance member         0.267702        3.735  0.235826          4.240    214
local variable          0.939655        1.064  0.939655          1.064     29
parameter               0.778645        1.284  0.772689          1.294    119
static member           0.260712        3.836  0.209305          4.778    163
top level member        0.060579       16.507  0.000000       Infinity     18


# Completion metrics for no relevance

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.183746        5.442  0.130687          7.652    655
successful completions  0.183746        5.442  0.130687          7.652    655
                                                                             
class                   0.049829       20.069  0.000000       Infinity     47
extension               0.008428      118.650  0.000000       Infinity     65
instance member         0.251786        3.972  0.214252          4.667    214
local variable          0.160259        6.240  0.052299         19.121     29
parameter               0.155790        6.419  0.070168         14.251    119
static member           0.244422        4.091  0.183333          5.455    163
top level member        0.030783       32.485  0.000000       Infinity     18
```

### 2. `pkg/dds/lib` (`--prefix-length=0`, 32 files, 4,342 completions, 47.5s)
```text
Analyzing root: "/home/ubuntu/projects/dart-sdk/sdk/pkg/dds/lib"
Computing completions at root: /home/ubuntu/projects/dart-sdk/sdk/pkg/dds/lib
Resolving 32 files...
Analyzing completion suggestions...

Metrics computed in 0:00:47.452000

# Comparison of experiments

## Comparison of inverse mean reciprocal ranks

A lower value is better, so a negative delta is good.

                        shipping  |  no tables     delta  |  no relevance     delta
all completions            2.566  |      2.597     0.031  |         5.935     3.370
successful completions     2.538  |      2.569     0.031  |         5.871     3.333
                                                                                   
class                     28.483  |     25.806    -2.677  |        69.388    40.905
enum                       1.385  |      2.455     1.070  |        12.103    10.719
extension                396.125  |    352.542   -43.584  |       354.309   -41.816
instance member            3.088  |      3.845     0.757  |         4.453     1.365
local function            62.811  |     12.872   -49.939  |        10.266   -52.545
local variable             1.635  |      1.395    -0.239  |         7.167     5.532
parameter                  1.694  |      1.666    -0.027  |         7.725     6.032
prefix                   504.392  |     60.976  -443.416  |        17.626  -486.766
static member              3.614  |      3.694     0.080  |         3.989     0.375
top level member          10.126  |     20.623    10.497  |        24.008    13.882
unknown                    3.429  |      1.333    -2.095  |         1.333    -2.095

## Counts for 'relevance rank comparison' (total = 4342)

              count  percent
shipping      3327   76.62%
no tables     746    17.18%
no relevance  269    6.20%

## Comparison of other metrics

                                       shipping        no tables     no relevance
ms per completion             0, 3.119674, 1946  0, 2.378114, 24  0, 1.064028, 10
chars_before_top                0, 2.290803, 31  0, 2.347381, 31  0, 2.506868, 23
chars_before_top_five           0, 0.836088, 31  0, 0.908265, 31  0, 1.131083, 23
insertion_length_theoretical    0, 7.597672, 46  0, 7.541094, 46  0, 7.381607, 46

shipping: [0] 4281 [10] 13 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 1
no tables: [0] 4291 [10] 3 [20] 1 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0
no relevance: [0] 4293 [10] 2 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0

## Comparison of completion counts

                   shipping      no tables   no relevance
total                  4342           4342           4342
successful    4295 (98.92%)  4295 (98.92%)  4295 (98.92%)
unsuccessful    47 ( 1.08%)    47 ( 1.08%)    47 ( 1.08%)

# Completion metrics for shipping

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.389757        2.566  0.365442          2.736   4342
successful completions  0.394022        2.538  0.369441          2.707   4295
                                                                             
class                   0.035109       28.483  0.013619         73.429    257
enum                    0.722222        1.385  0.722222          1.385      9
extension               0.002524      396.125  0.000000       Infinity      4
instance member         0.323839        3.088  0.291899          3.426   1718
local function          0.015921       62.811  0.000000       Infinity     25
local variable          0.611666        1.635  0.598292          1.671    878
parameter               0.590358        1.694  0.577800          1.731    756
prefix                  0.001983      504.392  0.000000       Infinity     40
static member           0.276735        3.614  0.233801          4.277    428
top level member        0.098754       10.126  0.070225         14.240    178
unknown                 0.291667        3.429  0.291667          3.429      2


# Completion metrics for no tables

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.385065        2.597  0.358372          2.790   4342
successful completions  0.389278        2.569  0.362293          2.760   4295
                                                                             
class                   0.038751       25.806  0.022049         45.353    257
enum                    0.407407        2.455  0.407407          2.455      9
extension               0.002837      352.542  0.000000       Infinity      4
instance member         0.260093        3.845  0.220809          4.529   1718
local function          0.077691       12.872  0.010000        100.000     25
local variable          0.716632        1.395  0.709434          1.410    878
parameter               0.600095        1.666  0.588360          1.700    756
prefix                  0.016400       60.976  0.000000       Infinity     40
static member           0.270721        3.694  0.221729          4.510    428
top level member        0.048489       20.623  0.017041         58.681    178
unknown                 0.750000        1.333  0.750000          1.333      2


# Completion metrics for no relevance

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.168486        5.935  0.108487          9.218   4342
successful completions  0.170330        5.871  0.109674          9.118   4295
                                                                             
class                   0.014412       69.388  0.000000       Infinity    257
enum                    0.082621       12.103  0.000000       Infinity      9
extension               0.002822      354.309  0.000000       Infinity      4
instance member         0.224583        4.453  0.182586          5.477   1718
local function          0.097410       10.266  0.018000         55.556     25
local variable          0.139524        7.167  0.038781         25.786    878
parameter               0.129442        7.725  0.041997         23.811    756
prefix                  0.056734       17.626  0.000000       Infinity     40
static member           0.250688        3.989  0.203544          4.913    428
top level member        0.041653       24.008  0.014045         71.200    178
unknown                 0.750000        1.333  0.750000          1.333      2
```

### 3. `pkg/analysis_server/lib/src/lsp` (`--prefix-length=0`, 113 files, 18,065 completions, 21m 05s)
```text
Analyzing root: "/home/ubuntu/projects/dart-sdk/sdk/pkg/analysis_server/lib/src/lsp"
Computing completions at root: /home/ubuntu/projects/dart-sdk/sdk/pkg/analysis_server/lib/src/lsp
Resolving 113 files...
Analyzing completion suggestions...

Metrics computed in 0:21:05.144000

# Comparison of experiments

## Comparison of inverse mean reciprocal ranks

A lower value is better, so a negative delta is good.

                        shipping  |  no tables      delta  |  no relevance      delta
all completions            2.253  |      2.369      0.117  |         6.327      4.075
successful completions     2.235  |      2.350      0.116  |         6.277      4.042
                                                                                     
class                      5.691  |      4.651     -1.040  |       284.116    278.424
enum                       1.741  |      3.342      1.601  |        88.512     86.771
instance member            2.524  |      3.013      0.489  |         4.535      2.011
local function           144.891  |      9.839   -135.052  |         5.600   -139.291
local variable             1.664  |      1.464     -0.200  |         7.675      6.010
mixin                   3972.000  |   2377.000  -1595.000  |      2377.000  -1595.000
parameter                  1.722  |      1.640     -0.082  |         7.731      6.009
prefix                   490.991  |    302.211   -188.780  |         9.733   -481.258
static member              3.698  |      3.575     -0.123  |         3.704      0.006
top level member           2.935  |     27.841     24.906  |       144.384    141.449
unknown                    6.013  |      2.413     -3.601  |         2.624     -3.390

## Counts for 'relevance rank comparison' (total = 18065)

              count  percent
shipping      13656  75.59%
no tables     3447   19.08%
no relevance  962    5.33%

## Comparison of other metrics

                                        shipping          no tables      no relevance
ms per completion             0, 21.868876, 3278  0, 21.321170, 101  0, 12.110200, 66
chars_before_top                 0, 2.355931, 32    0, 2.541625, 35   0, 2.861957, 35
chars_before_top_five            0, 0.944705, 28    0, 0.891865, 28   0, 1.358052, 27
insertion_length_theoretical     0, 7.748131, 41    0, 7.562437, 42   0, 7.242105, 41

shipping: [0] 6137 [10] 945 [20] 4192 [30] 5588 [40] 439 [50] 250 [60] 164 [70] 126 [80] 58 [90] 18 [100] 5
no tables: [0] 6212 [10] 895 [20] 3766 [30] 6030 [40] 457 [50] 258 [60] 136 [70] 112 [80] 49 [90] 6 [100] 1
no relevance: [0] 6262 [10] 8696 [20] 2706 [30] 188 [40] 53 [50] 15 [60] 2 [70] 0 [80] 0 [90] 0 [100] 0

## Comparison of completion counts

                    shipping       no tables    no relevance
total                  18065           18065           18065
successful    17922 (99.21%)  17922 (99.21%)  17922 (99.21%)
unsuccessful    143 ( 0.79%)    143 ( 0.79%)    143 ( 0.79%)

# Completion metrics for shipping

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.443943        2.253  0.425283          2.351  18065
successful completions  0.447485        2.235  0.428676          2.333  17922
                                                                             
class                   0.175705        5.691  0.161562          6.190    813
enum                    0.574473        1.741  0.562417          1.778    251
instance member         0.396274        2.524  0.370603          2.698   6813
local function          0.006902      144.891  0.000000       Infinity     59
local variable          0.600939        1.664  0.587368          1.703   3776
mixin                   0.000252     3972.000  0.000000       Infinity      1
parameter               0.580643        1.722  0.568944          1.758   3440
prefix                  0.002037      490.991  0.000000       Infinity    208
static member           0.270385        3.698  0.234047          4.273   1167
top level member        0.340745        2.935  0.333253          3.001   1251
unknown                 0.166297        6.013  0.141259          7.079    143


# Completion metrics for no tables

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.422099        2.369  0.400799          2.495  18065
successful completions  0.425467        2.350  0.403997          2.475  17922
                                                                             
class                   0.214993        4.651  0.206376          4.846    813
enum                    0.299245        3.342  0.285193          3.506    251
instance member         0.331901        3.013  0.300976          3.323   6813
local function          0.101635        9.839  0.025424         39.333     59
local variable          0.683122        1.464  0.671743          1.489   3776
mixin                   0.000421     2377.000  0.000000       Infinity      1
parameter               0.609681        1.640  0.594845          1.681   3440
prefix                  0.003309      302.211  0.000000       Infinity    208
static member           0.279707        3.575  0.242288          4.127   1167
top level member        0.035918       27.841  0.023421         42.696   1251
unknown                 0.414489        2.413  0.379021          2.638    143


# Completion metrics for no relevance

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.158051        6.327  0.100029          9.997  18065
successful completions  0.159312        6.277  0.100827          9.918  17922
                                                                             
class                   0.003520      284.116  0.000000       Infinity    813
enum                    0.011298       88.512  0.001328        753.000    251
instance member         0.220522        4.535  0.182609          5.476   6813
local function          0.178575        5.600  0.089831         11.132     59
local variable          0.130301        7.675  0.029480         33.921   3776
mixin                   0.000421     2377.000  0.000000       Infinity      1
parameter               0.129346        7.731  0.034835         28.707   3440
prefix                  0.102741        9.733  0.037500         26.667    208
static member           0.269958        3.704  0.230720          4.334   1167
top level member        0.006926      144.384  0.000320       3127.500   1251
unknown                 0.381126        2.624  0.340326          2.938    143
```

### 4. `pkg/analyzer/lib/src/dart/resolver` (`--prefix-length=0`, 42 files, 25,322 completions, 10m 58s)
```text
Analyzing root: "/home/ubuntu/projects/dart-sdk/sdk/pkg/analyzer/lib/src/dart/resolver"
Computing completions at root: /home/ubuntu/projects/dart-sdk/sdk/pkg/analyzer/lib/src/dart/resolver
Resolving 42 files...
Analyzing completion suggestions...

Metrics computed in 0:10:57.775000

# Comparison of experiments

## Comparison of inverse mean reciprocal ranks

A lower value is better, so a negative delta is good.

                        shipping  |  no tables      delta  |  no relevance      delta
all completions            2.186  |      2.241      0.054  |         6.304      4.117
successful completions     2.173  |      2.227      0.054  |         6.265      4.092
                                                                                     
class                     11.445  |     10.842     -0.602  |       193.221    181.776
enum                       6.436  |      6.658      0.222  |        60.869     54.434
extension               3382.168  |   1236.272  -2145.896  |      1236.272  -2145.896
instance member            3.013  |      3.584      0.572  |         5.623      2.611
local function           768.498  |     16.412   -752.086  |         6.924   -761.574
local variable             1.568  |      1.372     -0.196  |         7.251      5.683
parameter                  1.522  |      1.534      0.012  |         7.698      6.176
prefix                   788.166  |    326.647   -461.518  |        29.499   -758.667
static member              1.714  |      1.609     -0.105  |         1.626     -0.088
top level member           5.136  |     97.172     92.036  |       371.120    365.984
unknown                    3.386  |      1.253     -2.133  |         1.423     -1.962

## Counts for 'relevance rank comparison' (total = 25322)

              count  percent
shipping      19959  78.82%
no tables     4285   16.92%
no relevance  1078   4.26%

## Comparison of other metrics

                                       shipping        no tables     no relevance
ms per completion             0, 8.449420, 1575  0, 7.995272, 55  0, 4.148800, 26
chars_before_top                0, 2.185513, 42  0, 2.244636, 42  0, 2.811268, 42
chars_before_top_five           0, 0.657621, 27  0, 0.699539, 27  0, 1.123053, 27
insertion_length_theoretical    0, 8.429434, 44  0, 8.370312, 44  0, 7.803679, 44

shipping: [0] 12136 [10] 12937 [20] 80 [30] 7 [40] 2 [50] 2 [60] 3 [70] 0 [80] 0 [90] 0 [100] 1
no tables: [0] 13087 [10] 12012 [20] 55 [30] 6 [40] 3 [50] 5 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0
no relevance: [0] 25024 [10] 139 [20] 5 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0

## Comparison of completion counts

                    shipping       no tables    no relevance
total                  25322           25322           25322
successful    25168 (99.39%)  25168 (99.39%)  25168 (99.39%)
unsuccessful    154 ( 0.61%)    154 ( 0.61%)    154 ( 0.61%)

# Completion metrics for shipping

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.457361        2.186  0.436437          2.291  25322
successful completions  0.460160        2.173  0.439107          2.277  25168
                                                                             
class                   0.087377       11.445  0.067116         14.900    334
enum                    0.155386        6.436  0.108190          9.243    175
extension               0.000296     3382.168  0.000000       Infinity      2
instance member         0.331909        3.013  0.300778          3.325  11776
local function          0.001301      768.498  0.000000       Infinity     24
local variable          0.637883        1.568  0.624520          1.601   5418
parameter               0.657045        1.522  0.647541          1.544   5416
prefix                  0.001269      788.166  0.000000       Infinity    263
static member           0.583490        1.714  0.572222          1.748    645
top level member        0.194698        5.136  0.179006          5.586   1036
unknown                 0.295359        3.386  0.289451          3.455     79


# Completion metrics for no tables

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.446276        2.241  0.424213          2.357  25322
successful completions  0.449007        2.227  0.426809          2.343  25168
                                                                             
class                   0.092231       10.842  0.065319         15.309    334
enum                    0.150195        6.658  0.129905          7.698    175
extension               0.000809     1236.272  0.000000       Infinity      2
instance member         0.278986        3.584  0.244640          4.088  11776
local function          0.060929       16.412  0.013889         72.000     24
local variable          0.729089        1.372  0.720103          1.389   5418
parameter               0.651985        1.534  0.637497          1.569   5416
prefix                  0.003061      326.647  0.000000       Infinity    263
static member           0.621472        1.609  0.607494          1.646    645
top level member        0.010291       97.172  0.007111        140.633   1036
unknown                 0.798101        1.253  0.794304          1.259     79


# Completion metrics for no relevance

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.158634        6.304  0.094309         10.603  25322
successful completions  0.159605        6.265  0.094886         10.539  25168
                                                                             
class                   0.005175      193.221  0.001497        668.000    334
enum                    0.016429       60.869  0.007619        131.250    175
extension               0.000809     1236.272  0.000000       Infinity      2
instance member         0.177828        5.623  0.135093          7.402  11776
local function          0.144427        6.924  0.058333         17.143     24
local variable          0.137916        7.251  0.037219         26.868   5418
parameter               0.129897        7.698  0.027597         36.236   5416
prefix                  0.033900       29.499  0.000000       Infinity    263
static member           0.615189        1.626  0.600620          1.665    645
top level member        0.002695      371.120  0.000000       Infinity   1036
unknown                 0.702532        1.423  0.702532          1.423     79
```

---

## Raw Outputs (`--prefix-length=2`)

### 1. `pkg/dtd/lib` (`--prefix-length=2`, 10 files, 655 completions, 7.1s)
```text
Analyzing root: "/home/ubuntu/projects/dart-sdk/sdk/pkg/dtd/lib"
Computing completions at root: /home/ubuntu/projects/dart-sdk/sdk/pkg/dtd/lib
Resolving 10 files...
Analyzing completion suggestions...

Metrics computed in 0:00:07.131000

# Comparison of experiments

## Comparison of inverse mean reciprocal ranks

A lower value is better, so a negative delta is good.

                        shipping  |  no tables   delta  |  no relevance   delta
all completions            1.228  |      1.229   0.000  |         1.391   0.163
successful completions     1.129  |      1.129   0.000  |         1.279   0.150
                                                                               
class                      1.232  |      1.234   0.003  |         1.234   0.003
extension                  1.337  |      1.337   0.000  |         1.975   0.637
instance member            1.083  |      1.074  -0.010  |         1.110   0.026
local variable             1.087  |      1.048  -0.039  |         1.655   0.567
parameter                  1.087  |      1.082  -0.005  |         1.797   0.711
static member              1.169  |      1.212   0.043  |         1.137  -0.032
top level member           1.137  |      1.080  -0.057  |         1.096  -0.040

## Counts for 'relevance rank comparison' (total = 655)

              count  percent
shipping      571    87.18%
no relevance  62     9.47%
no tables     22     3.36%

## Comparison of other metrics

                                       shipping        no tables     no relevance
ms per completion             0, 4.132890, 2058   0, 0.521595, 4   0, 0.428571, 4
chars_before_top                0, 0.642857, 18  0, 0.616279, 18  0, 1.237542, 18
chars_before_top_five            0, 0.009967, 3   0, 0.023256, 3   0, 0.171096, 3
insertion_length_theoretical    0, 9.255814, 32  0, 9.282392, 32  0, 8.661130, 31

shipping: [0] 600 [10] 1 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 1
no tables: [0] 602 [10] 0 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0
no relevance: [0] 602 [10] 0 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0

## Comparison of completion counts

                  shipping     no tables  no relevance
total                  655           655           655
successful    602 (91.91%)  602 (91.91%)  602 (91.91%)
unsuccessful   53 ( 8.09%)   53 ( 8.09%)   53 ( 8.09%)

# Completion metrics for shipping

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.814233        1.228  0.813639          1.229    655
successful completions  0.885918        1.129  0.885271          1.130    602
                                                                             
class                   0.811957        1.232  0.811957          1.232     46
extension               0.747807        1.337  0.747807          1.337     38
instance member         0.923084        1.083  0.921340          1.085    214
local variable          0.919540        1.087  0.919540          1.087     29
parameter               0.920304        1.087  0.920168          1.087    119
static member           0.855072        1.169  0.855072          1.169    138
top level member        0.879630        1.137  0.879630          1.137     18


# Completion metrics for no tables

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.813988        1.229  0.813359          1.229    655
successful completions  0.885651        1.129  0.884967          1.130    602
                                                                             
class                   0.810145        1.234  0.810145          1.234     46
extension               0.747807        1.337  0.747807          1.337     38
instance member         0.931365        1.074  0.929439          1.076    214
local variable          0.954023        1.048  0.954023          1.048     29
parameter               0.924370        1.082  0.924370          1.082    119
static member           0.824879        1.212  0.824879          1.212    138
top level member        0.925926        1.080  0.925926          1.080     18


# Completion metrics for no relevance

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.718841        1.391  0.710204          1.408    655
successful completions  0.782127        1.279  0.772730          1.294    602
                                                                             
class                   0.810145        1.234  0.810145          1.234     46
extension               0.506410        1.975  0.493421          2.027     38
instance member         0.901142        1.110  0.897586          1.114    214
local variable          0.604367        1.655  0.586207          1.706     29
parameter               0.556382        1.797  0.525210          1.904    119
static member           0.879227        1.137  0.879227          1.137    138
top level member        0.912037        1.096  0.902778          1.108     18
```

### 2. `pkg/dds/lib` (`--prefix-length=2`, 32 files, 4,342 completions, 32.8s)
```text
Analyzing root: "/home/ubuntu/projects/dart-sdk/sdk/pkg/dds/lib"
Computing completions at root: /home/ubuntu/projects/dart-sdk/sdk/pkg/dds/lib
Resolving 32 files...
Analyzing completion suggestions...

Metrics computed in 0:00:32.763000

# Comparison of experiments

## Comparison of inverse mean reciprocal ranks

A lower value is better, so a negative delta is good.

                        shipping  |  no tables   delta  |  no relevance   delta
all completions            1.249  |      1.242  -0.007  |         1.721   0.472
successful completions     1.231  |      1.224  -0.007  |         1.696   0.465
                                                                               
class                      2.509  |      2.825   0.317  |         3.309   0.801
enum                       1.000  |      1.000   0.000  |         1.714   0.714
extension                 22.000  |     22.000   0.000  |        22.000   0.000
instance member            1.210  |      1.216   0.006  |         1.243   0.034
local function             2.422  |      1.447  -0.975  |         2.359  -0.062
local variable             1.088  |      1.063  -0.025  |         2.467   1.380
parameter                  1.073  |      1.087   0.014  |         2.460   1.387
prefix                     4.253  |      2.221  -2.031  |         5.625   1.373
static member              1.313  |      1.305  -0.009  |         1.479   0.165
top level member           1.818  |      1.631  -0.187  |         1.865   0.047
unknown                    1.000  |      1.000   0.000  |         1.000   0.000

## Counts for 'relevance rank comparison' (total = 4342)

              count  percent
shipping      3952   91.02%
no tables     212    4.88%
no relevance  178    4.10%

## Comparison of other metrics

                                       shipping        no tables     no relevance
ms per completion             0, 2.108670, 1922  0, 1.549661, 12   0, 1.245151, 7
chars_before_top                0, 1.578874, 23  0, 1.618836, 23  0, 2.333489, 23
chars_before_top_five           0, 0.194672, 23  0, 0.352886, 23  0, 0.766768, 23
insertion_length_theoretical    0, 8.281374, 48  0, 8.241412, 48  0, 7.526759, 48

shipping: [0] 4273 [10] 5 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 1
no tables: [0] 4275 [10] 4 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0
no relevance: [0] 4279 [10] 0 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0

## Comparison of completion counts

                   shipping      no tables   no relevance
total                  4342           4342           4342
successful    4279 (98.55%)  4279 (98.55%)  4279 (98.55%)
unsuccessful    63 ( 1.45%)    63 ( 1.45%)    63 ( 1.45%)

# Completion metrics for shipping

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.800769        1.249  0.794998          1.258   4342
successful completions  0.812559        1.231  0.806703          1.240   4279
                                                                             
class                   0.398605        2.509  0.364902          2.740    255
enum                    1.000000        1.000  1.000000          1.000      6
extension               0.045455       22.000  0.000000       Infinity      4
instance member         0.826692        1.210  0.823525          1.214   1718
local function          0.412963        2.422  0.402000          2.488     25
local variable          0.919503        1.088  0.918700          1.088    877
parameter               0.932146        1.073  0.931325          1.074    755
prefix                  0.235149        4.253  0.200000          5.000     40
static member           0.761375        1.313  0.752307          1.329    419
top level member        0.550199        1.818  0.527528          1.896    178
unknown                 1.000000        1.000  1.000000          1.000      2


# Completion metrics for no tables

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.805354        1.242  0.799651          1.251   4342
successful completions  0.817212        1.224  0.811424          1.232   4279
                                                                             
class                   0.353941        2.825  0.314641          3.178    255
enum                    1.000000        1.000  1.000000          1.000      6
extension               0.045455       22.000  0.000000       Infinity      4
instance member         0.822389        1.216  0.819451          1.220   1718
local function          0.691282        1.447  0.683333          1.463     25
local variable          0.940904        1.063  0.940707          1.063    877
parameter               0.920217        1.087  0.918874          1.088    755
prefix                  0.450159        2.221  0.425000          2.353     40
static member           0.766485        1.305  0.757478          1.320    419
top level member        0.613289        1.631  0.594476          1.682    178
unknown                 1.000000        1.000  1.000000          1.000      2


# Completion metrics for no relevance

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.581121        1.721  0.565269          1.769   4342
successful completions  0.589677        1.696  0.573592          1.743   4279
                                                                             
class                   0.302174        3.309  0.258954          3.862    255
enum                    0.583333        1.714  0.583333          1.714      6
extension               0.045455       22.000  0.000000       Infinity      4
instance member         0.804183        1.243  0.801261          1.248   1718
local function          0.423887        2.359  0.410000          2.439     25
local variable          0.405298        2.467  0.381452          2.622    877
parameter               0.406539        2.460  0.380508          2.628    755
prefix                  0.177764        5.625  0.137500          7.273     40
static member           0.676304        1.479  0.662609          1.509    419
top level member        0.536196        1.865  0.511798          1.954    178
unknown                 1.000000        1.000  1.000000          1.000      2
```

### 3. `pkg/analysis_server/lib/src/lsp` (`--prefix-length=2`, 113 files, 18,065 completions, 14m 48s)
```text
Analyzing root: "/home/ubuntu/projects/dart-sdk/sdk/pkg/analysis_server/lib/src/lsp"
Computing completions at root: /home/ubuntu/projects/dart-sdk/sdk/pkg/analysis_server/lib/src/lsp
Resolving 113 files...
Analyzing completion suggestions...

Metrics computed in 0:14:48.062000

# Comparison of experiments

## Comparison of inverse mean reciprocal ranks

A lower value is better, so a negative delta is good.

                        shipping  |  no tables   delta  |  no relevance   delta
all completions            1.237  |      1.231  -0.006  |         2.174   0.937
successful completions     1.214  |      1.208  -0.006  |         2.134   0.920
                                                                               
class                      1.825  |      1.870   0.045  |         5.905   4.080
enum                       1.907  |      2.025   0.118  |         3.370   1.463
instance member            1.236  |      1.270   0.034  |         1.415   0.179
local function             2.588  |      1.504  -1.084  |         3.560   0.972
local variable             1.081  |      1.066  -0.014  |         4.309   3.228
mixin                     16.000  |     16.000   0.000  |        16.000   0.000
parameter                  1.056  |      1.053  -0.003  |         5.630   4.574
prefix                     5.993  |      5.065  -0.928  |         1.637  -4.356
static member              1.306  |      1.313   0.007  |         1.347   0.041
top level member           1.575  |      1.311  -0.264  |         1.723   0.148
unknown                    1.021  |      1.021  -0.000  |         1.021   0.000

## Counts for 'relevance rank comparison' (total = 18065)

              count  percent
shipping      16376  90.65%
no tables     1048   5.80%
no relevance  641    3.55%

## Comparison of other metrics

                                        shipping         no tables      no relevance
ms per completion             0, 16.304581, 3004  0, 15.912163, 75  0, 14.218323, 67
chars_before_top                 0, 1.621065, 29   0, 1.718211, 29   0, 3.179623, 29
chars_before_top_five            0, 0.382207, 20   0, 0.401896, 20   0, 1.305089, 20
insertion_length_theoretical     0, 8.452950, 43   0, 8.355805, 43   0, 6.894392, 42

shipping: [0] 6371 [10] 3448 [20] 5639 [30] 1766 [40] 461 [50] 28 [60] 6 [70] 5 [80] 0 [90] 1 [100] 1
no tables: [0] 6367 [10] 3537 [20] 5542 [30] 1807 [40] 431 [50] 31 [60] 4 [70] 7 [80] 0 [90] 0 [100] 0
no relevance: [0] 6376 [10] 4624 [20] 5480 [30] 1147 [40] 85 [50] 6 [60] 8 [70] 0 [80] 0 [90] 0 [100] 0

## Comparison of completion counts

                    shipping       no tables    no relevance
total                  18065           18065           18065
successful    17726 (98.12%)  17726 (98.12%)  17726 (98.12%)
unsuccessful    339 ( 1.88%)    339 ( 1.88%)    339 ( 1.88%)

# Completion metrics for shipping

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.808462        1.237  0.802592          1.246  18065
successful completions  0.823923        1.214  0.817942          1.223  17726
                                                                             
class                   0.547971        1.825  0.537030          1.862    798
enum                    0.524351        1.907  0.468778          2.133    150
instance member         0.809185        1.236  0.802625          1.246   6755
local function          0.386340        2.588  0.355932          2.810     59
local variable          0.925309        1.081  0.924743          1.081   3768
mixin                   0.062500       16.000  0.000000       Infinity      1
parameter               0.946883        1.056  0.945791          1.057   3437
prefix                  0.166865        5.993  0.156000          6.410    200
static member           0.765614        1.306  0.756927          1.321   1167
top level member        0.634750        1.575  0.615051          1.626   1248
unknown                 0.979094        1.021  0.979021          1.021    143


# Completion metrics for no tables

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.812453        1.231  0.807064          1.239  18065
successful completions  0.827991        1.208  0.822499          1.216  17726
                                                                             
class                   0.534798        1.870  0.524749          1.906    798
enum                    0.493707        2.025  0.433333          2.308    150
instance member         0.787245        1.270  0.780972          1.280   6755
local function          0.664783        1.504  0.659887          1.515     59
local variable          0.937845        1.066  0.937279          1.067   3768
mixin                   0.062500       16.000  0.000000       Infinity      1
parameter               0.950021        1.053  0.948279          1.055   3437
prefix                  0.197440        5.065  0.186833          5.352    200
static member           0.761386        1.313  0.752399          1.329   1167
top level member        0.762776        1.311  0.749319          1.335   1248
unknown                 0.979233        1.021  0.979021          1.021    143


# Completion metrics for no relevance

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.459906        2.174  0.445174          2.246  18065
successful completions  0.468701        2.134  0.453688          2.204  17726
                                                                             
class                   0.169359        5.905  0.137615          7.267    798
enum                    0.296712        3.370  0.221667          4.511    150
instance member         0.706778        1.415  0.697893          1.433   6755
local function          0.280907        3.560  0.271751          3.680     59
local variable          0.232090        4.309  0.215862          4.633   3768
mixin                   0.062500       16.000  0.000000       Infinity      1
parameter               0.177618        5.630  0.155223          6.442   3437
prefix                  0.610947        1.637  0.606833          1.648    200
static member           0.742309        1.347  0.732976          1.364   1167
top level member        0.580260        1.723  0.564971          1.770   1248
unknown                 0.979058        1.021  0.979021          1.021    143
```

### 4. `pkg/analyzer/lib/src/dart/resolver` (`--prefix-length=2`, 42 files, 25,322 completions, 7m 31s)
```text
Analyzing root: "/home/ubuntu/projects/dart-sdk/sdk/pkg/analyzer/lib/src/dart/resolver"
Computing completions at root: /home/ubuntu/projects/dart-sdk/sdk/pkg/analyzer/lib/src/dart/resolver
Resolving 42 files...
Analyzing completion suggestions...

Metrics computed in 0:07:30.603000

# Comparison of experiments

## Comparison of inverse mean reciprocal ranks

A lower value is better, so a negative delta is good.

                        shipping  |  no tables    delta  |  no relevance    delta
all completions            1.225  |      1.240    0.015  |         2.153    0.929
successful completions     1.217  |      1.232    0.015  |         2.140    0.923
                                                                                 
class                      2.017  |      1.938   -0.080  |         4.682    2.665
enum                       1.327  |      2.172    0.845  |         6.562    5.235
extension                140.456  |     69.468  -70.988  |        69.468  -70.988
instance member            1.300  |      1.346    0.046  |         1.442    0.142
local function             1.894  |      1.659   -0.236  |         1.592   -0.302
local variable             1.063  |      1.048   -0.016  |         3.766    2.703
parameter                  1.026  |      1.023   -0.003  |         5.076    4.050
prefix                    34.329  |     20.844  -13.485  |         3.999  -30.330
static member              1.140  |      1.136   -0.004  |         1.147    0.007
top level member           3.629  |      3.663    0.034  |         3.788    0.159
unknown                    1.013  |      1.013    0.000  |         1.039    0.027

## Counts for 'relevance rank comparison' (total = 25322)

              count  percent
shipping      24042  94.95%
no tables     743    2.93%
no relevance  537    2.12%

## Comparison of other metrics

                                       shipping        no tables     no relevance
ms per completion             0, 5.555313, 1582  0, 5.343320, 21  0, 4.698959, 16
chars_before_top                0, 1.692442, 42  0, 1.753040, 42  0, 2.753437, 42
chars_before_top_five           0, 0.275411, 27  0, 0.307160, 27  0, 0.816181, 29
insertion_length_theoretical    0, 8.922554, 45  0, 8.861957, 45  0, 7.861559, 45

shipping: [0] 21083 [10] 4080 [20] 1 [30] 1 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 1
no tables: [0] 21507 [10] 3658 [20] 1 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0
no relevance: [0] 22975 [10] 2191 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0

## Comparison of completion counts

                    shipping       no tables    no relevance
total                  25322           25322           25322
successful    25166 (99.38%)  25166 (99.38%)  25166 (99.38%)
unsuccessful    156 ( 0.62%)    156 ( 0.62%)    156 ( 0.62%)

# Completion metrics for shipping

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.816572        1.225  0.811670          1.232  25322
successful completions  0.821634        1.217  0.816701          1.224  25166
                                                                             
class                   0.495680        2.017  0.468313          2.135    334
enum                    0.753390        1.327  0.750476          1.332    175
extension               0.007120      140.456  0.000000       Infinity      2
instance member         0.769246        1.300  0.761124          1.314  11776
local function          0.527848        1.894  0.516667          1.935     24
local variable          0.940297        1.063  0.939948          1.064   5417
parameter               0.974718        1.026  0.974561          1.026   5415
prefix                  0.029130       34.329  0.005070        197.250    263
static member           0.877056        1.140  0.876382          1.141    645
top level member        0.275543        3.629  0.266795          3.748   1036
unknown                 0.987342        1.013  0.987342          1.013     79


# Completion metrics for no tables

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.806390        1.240  0.801659          1.247  25322
successful completions  0.811389        1.232  0.806628          1.240  25166
                                                                             
class                   0.516087        1.938  0.498453          2.006    334
enum                    0.460430        2.172  0.454190          2.202    175
extension               0.014395       69.468  0.000000       Infinity      2
instance member         0.742888        1.346  0.734785          1.361  11776
local function          0.602944        1.659  0.597222          1.674     24
local variable          0.954370        1.048  0.954194          1.048   5417
parameter               0.977416        1.023  0.977408          1.023   5415
prefix                  0.047974       20.844  0.024081         41.526    263
static member           0.880276        1.136  0.879742          1.137    645
top level member        0.272971        3.663  0.263674          3.793   1036
unknown                 0.987342        1.013  0.987342          1.013     79


# Completion metrics for no relevance

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.464414        2.153  0.450804          2.218  25322
successful completions  0.467293        2.140  0.453598          2.205  25166
                                                                             
class                   0.213584        4.682  0.192715          5.189    334
enum                    0.152390        6.562  0.127143          7.865    175
extension               0.014395       69.468  0.000000       Infinity      2
instance member         0.693310        1.442  0.684132          1.462  11776
local function          0.627944        1.592  0.625000          1.600     24
local variable          0.265515        3.766  0.244785          4.085   5417
parameter               0.196991        5.076  0.178538          5.601   5415
prefix                  0.250069        3.999  0.236882          4.222    263
static member           0.871464        1.147  0.870594          1.149    645
top level member        0.264004        3.788  0.255486          3.914   1036
unknown                 0.962025        1.039  0.962025          1.039     79
```


---

# Part 2: Follow-up Experiment — Preserving `inheritanceDistance` Without Tables (`#48771`)

Following Brian Wilkerson's observation on [#48771](https://github.com/dart-lang/sdk/issues/48771#issuecomment-6749231693) that disabling `elementKindFeature` by returning `0.0` also inadvertently zeroes out `inheritanceDistance` (which is passed as a `distance` parameter into `elementKindFeature` and interpolated via `range.conditionalProbability(distance)`), we added two additional configurations to `completion_metrics.dart`:

1. **`no tables (uniform)` (Brian's workaround):**
   * When `!useRelevanceTables && useUniformRelevanceRanges`:
     * `elementKindFeature` uses `const ProbabilityRange(lower: 0.0, upper: 1.0)` for all element kinds at all completion locations (returning `0.5` when `distance == null`, or `0.5 + 0.5 * distance` when `distance != null`).
     * `keywordFeature` returns `0.5` for all keywords.
   * This gives the relevance tables **zero net effect** on element kind / keyword rank while preserving `distance` (`inheritanceDistance` and `localVariableDistance`) scaling within `[0.5, 1.0]`.
2. **`no tables (distance)`:**
   * When `!useRelevanceTables && preserveDistanceWithoutTables`:
     * `elementKindFeature` returns `distance ?? 0.0`, and `keywordFeature` returns `0.0`.
3. **Fixed `--print-missing-information` tracking in `RelevanceComputer`:**
   * `_computeElementKind` previously checked `if (elementKind < 0.0)` before calling `listener?.missingCompletionLocationAt` / `listener?.missingElementKindTableFor`, which had been unreachable since commit `216b9d65d82` (Feb 2021) changed `elementKindFeature` to return `0.0` instead of `-1.0`. Restoring this check revealed all AST locations and completion location tables missing from `relevance_tables.g.dart`.

---

## 1. Summary of Results (`no tables (uniform)` & `no tables (distance)`)

### A. With `prefix-length = 2` (Realistic 2-Character Prefix)

With a 2-character prefix, **`no tables (uniform)` outperforms `shipping` across all four target directories** (both overall and specifically on `instance member`, `local variable`, `local function`, `static member`, and `prefix`):

| Target Directory | Completions | `shipping` | `no tables` (`0.0`) | `no tables (uniform)` | `no tables (distance)` | `no relevance` |
| :--- | ---: | ---: | ---: | ---: | ---: | ---: |
| `pkg/dtd/lib` | 655 | 1.228 | 1.229 (`+0.000`) | **1.202 (`-0.026`)** | 1.203 (`-0.025`) | 1.391 (`+0.163`) |
| `pkg/dds/lib` | 4,342 | 1.249 | 1.242 (`-0.007`) | **1.238 (`-0.010`)** | 1.260 (`+0.011`) | 1.721 (`+0.472`) |
| `pkg/analysis_server/lib/src/lsp` | 18,065 | 1.237 | 1.231 (`-0.006`) | **1.199 (`-0.037`)** | 1.207 (`-0.030`) | 2.174 (`+0.938`) |
| `pkg/analyzer/lib/src/dart/resolver` | 25,322 | 1.225 | 1.240 (`+0.015`) | **1.221 (`-0.003`)** | 1.227 (`+0.002`) | 2.153 (`+0.929`) |

#### `instance member` Inverse MRR at `prefix-length = 2` (lower is better):

| Target Directory | Instance Member Count | `shipping` | `no tables` (`0.0`) | `no tables (uniform)` | `no tables (distance)` |
| :--- | ---: | ---: | ---: | ---: | ---: |
| `pkg/dtd/lib` | 214 | 1.083 | 1.074 (`-0.010`) | **1.049 (`-0.035`)** | **1.049 (`-0.035`)** |
| `pkg/dds/lib` | 1,718 | 1.210 | 1.216 (`+0.006`) | **1.202 (`-0.008`)** | **1.199 (`-0.011`)** |
| `pkg/analysis_server/lib/src/lsp` | 6,755 | 1.236 | 1.270 (`+0.035`) | **1.172 (`-0.064`)** | **1.170 (`-0.065`)** |
| `pkg/analyzer/lib/src/dart/resolver` | 11,776 | 1.300 | 1.346 (`+0.046`) | **1.293 (`-0.006`)** | **1.293 (`-0.007`)** |

* In the original `no tables` (`0.0`) experiment, `instance member` had regressed slightly in `dds`, `lsp`, and `resolver` because `inheritanceDistance` was lost.
* Once `inheritanceDistance` is preserved via `no tables (uniform)`, **`instance member` ranking is better than `shipping` in all four packages**—because `inheritanceDistance` is now able to rank subclass methods above `Object` getters (`hashCode`, `runtimeType`) and extension getters ([Dart-Code#5719](https://github.com/Dart-Code/Dart-Code/issues/5719)), rather than being trapped inside the `METHOD` bucket `[0.004, 0.025]` below the `FIELD` bucket `[0.025, 0.785]`.

---

### B. With `prefix-length = 0` (Empty Prefix)

| Target Directory | Completions | `shipping` | `no tables` (`0.0`) | `no tables (uniform)` | `no tables (distance)` | `no relevance` |
| :--- | ---: | ---: | ---: | ---: | ---: | ---: |
| `pkg/dtd/lib` | 655 | 2.668 | 2.916 (`+0.248`) | **2.647 (`-0.021`)** | 2.803 (`+0.135`) | 5.442 (`+2.774`) |
| `pkg/dds/lib` | 4,342 | 2.566 | 2.597 (`+0.031`) | **2.546 (`-0.020`)** | 2.753 (`+0.187`) | 5.935 (`+3.370`) |
| `pkg/analysis_server/lib/src/lsp` | 18,065 | **2.253** | 2.369 (`+0.117`) | 2.400 (`+0.147`) | 2.573 (`+0.320`) | 6.327 (`+4.074`) |
| `pkg/analyzer/lib/src/dart/resolver` | 25,322 | **2.186** | 2.241 (`+0.054`) | 2.268 (`+0.081`) | 2.603 (`+0.417`) | 6.304 (`+4.117`) |

#### `instance member` Inverse MRR at `prefix-length = 0`:

| Target Directory | Instance Member Count | `shipping` | `no tables` (`0.0`) | `no tables (uniform)` | `no tables (distance)` |
| :--- | ---: | ---: | ---: | ---: | ---: |
| `pkg/dtd/lib` | 214 | 3.321 | 3.735 (`+0.414`) | **3.181 (`-0.141`)** | **3.071 (`-0.251`)** |
| `pkg/dds/lib` | 1,718 | **3.088** | 3.845 (`+0.757`) | 3.412 (`+0.324`) | 3.293 (`+0.205`) |
| `pkg/analysis_server/lib/src/lsp` | 6,813 | **2.524** | 3.013 (`+0.489`) | 2.754 (`+0.231`) | 2.709 (`+0.186`) |
| `pkg/analyzer/lib/src/dart/resolver` | 11,776 | **3.013** | 3.584 (`+0.572`) | 3.259 (`+0.246`) | 3.220 (`+0.208`) |

#### Why did `parameter` regress in `no tables (uniform)` / `no tables (distance)` at `prefix-length = 0`?
Comparing `_computeLocalVariableRelevance` and `_computeFormalParameterRelevance` in `RelevanceComputer` revealed another bug:
* `_computeLocalVariableRelevance` passes `distance: localVariableDistance` to `_computeElementKind(element, distance: localVariableDistance)` **and** passes `localVariableDistance` to `computeScore(...)`.
* `_computeFormalParameterRelevance` computes `localVariableDistance = featureComputer.distanceToPercent(suggestion.distance)`, passes it to `computeScore(...)`, **but forgets to pass `distance: localVariableDistance` to `_computeElementKind(element)`!**
* Consequently, when `elementKindFeature` uses a wide `[0.0, 1.0]` range (in `no tables (uniform)`) or returns `distance ?? 0.0` (in `no tables (distance)`), every `LocalVariableSuggestion` gets up to `+0.5` (or `+1.0`) from `elementKindFeature` for its `localVariableDistance`, whereas `FormalParameterSuggestion` gets `0.0` extra from `elementKindFeature` (because `distance` was `null`), causing all local variables to jump ahead of all parameters when `prefix-length = 0`.

---

## 2. Missing Completion Locations and Missing Tables (`--print-missing-information`)

Enabling `--print-missing-information` confirmed Brian's hypothesis that modern Dart syntax is missing from the 2020 tables (and in those locations, `elementKindFeature` returns `0.0` in `shipping`, silently dropping `inheritanceDistance` as well):

### A. AST locations where `OpType.completionLocation` is `null` (no location name assigned):
* `ArgumentList/AsExpression`
* `ArgumentList/ConditionalExpression`
* `ArgumentList/FunctionExpressionInvocation`
* `ArgumentList/PrefixedIdentifier`
* `ArgumentList/PropertyAccess`
* `ConstructorName/SimpleIdentifier`
* `DotShorthandConstructorInvocation/SimpleIdentifier`
* `DotShorthandPropertyAccess/SimpleIdentifier`
* `ListLiteral/PrefixedIdentifier`
* `ListLiteral/PropertyAccess`
* `MethodInvocation/SimpleIdentifier`
* `RecordLiteral/AsExpression`
* `RecordLiteral/PrefixedIdentifier`
* `RecordLiteral/PropertyAccess`
* `SetOrMapLiteral/PrefixedIdentifier`

### B. Completion locations assigned by `OpType` that have no table in `relevance_tables.g.dart`:
* `ArgumentList__unnamed`
* `MapPatternEntry_key`
* `NullAwareElement_value`
* `PatternAssignment_expression`
* `PatternVariableDeclaration_expression`
* `RecordLiteral_fields`
* `SwitchExpressionCase_expression`
* `SwitchExpression_body`
* `SwitchExpression_expression`
* `SwitchPatternCase_pattern`
* `WhenClause_expression`
* `visitForEachPartsWithPattern_iterable`

---

## 3. Raw Output (`--prefix-length=0`, with uniform & distance variants)

### 3.1 `pkg/dtd/lib` (`--prefix-length=0`)

```text
Command output:
Analyzing root: "/home/ubuntu/projects/dart-sdk/sdk/pkg/dtd/lib"
Computing completions at root: /home/ubuntu/projects/dart-sdk/sdk/pkg/dtd/lib
Resolving 10 files...
[                                                                    ]

Analyzing completion suggestions...
[                                                                    ]


Metrics computed in 0:00:11.147000

# Comparison of experiments

## Comparison of inverse mean reciprocal ranks

A lower value is better, so a negative delta is good.

                        shipping  |  no tables    delta  |  no tables (uniform)   delta  |  no tables (distance)   delta  |  no relevance   delta
all completions            2.668  |      2.916    0.248  |                2.647  -0.021  |                 2.803   0.135  |         5.442   2.774
successful completions     2.668  |      2.916    0.248  |                2.647  -0.021  |                 2.803   0.135  |         5.442   2.774
                                                                                                                                                 
class                     17.193  |     20.062    2.868  |               20.062   2.868  |                20.062   2.868  |        20.069   2.875
extension                 56.382  |     44.141  -12.240  |               82.122  25.740  |                81.844  25.462  |       118.650  62.268
instance member            3.321  |      3.735    0.414  |                3.181  -0.141  |                 3.071  -0.251  |         3.972   0.650
local variable             1.299  |      1.064   -0.234  |                1.045  -0.253  |                 1.045  -0.253  |         6.240   4.941
parameter                  1.339  |      1.284   -0.054  |                1.300  -0.039  |                 1.704   0.365  |         6.419   5.080
static member              3.100  |      3.836    0.735  |                2.878  -0.222  |                 2.620  -0.480  |         4.091   0.991
top level member           1.347  |     16.507   15.160  |               17.040  15.693  |                17.040  15.693  |        32.485  31.138

## Counts for 'relevance rank comparison' (total = 655)

                      count  percent
shipping              480    73.28%
no tables             66     10.08%
no tables (distance)  58     8.85%
no tables (uniform)   37     5.65%
no relevance          14     2.14%

## Comparison of other metrics

                                       shipping        no tables  no tables (uniform)  no tables (distance)     no relevance
ms per completion             0, 4.477863, 2153  0, 0.966412, 21      0, 0.912977, 10        0, 0.906870, 8   0, 0.328244, 3
chars_before_top                0, 3.112977, 32  0, 2.916031, 32      0, 3.087023, 32       0, 3.137405, 32  0, 2.360305, 18
chars_before_top_five           0, 2.013740, 32  0, 2.013740, 32      0, 2.117557, 32       0, 2.141985, 32   0, 0.784733, 3
insertion_length_theoretical    0, 7.103817, 30  0, 7.300763, 30      0, 7.129771, 30       0, 7.079389, 30  0, 7.856489, 30

shipping: [0] 651 [10] 3 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 1
no tables: [0] 654 [10] 0 [20] 1 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0
no tables (uniform): [0] 654 [10] 1 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0
no tables (distance): [0] 655 [10] 0 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0
no relevance: [0] 655 [10] 0 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0

## Comparison of completion counts

                   shipping      no tables  no tables (uniform)  no tables (distance)   no relevance
total                   655            655                  655                   655            655
successful    655 (100.00%)  655 (100.00%)        655 (100.00%)         655 (100.00%)  655 (100.00%)
unsuccessful     0 ( 0.00%)     0 ( 0.00%)           0 ( 0.00%)            0 ( 0.00%)     0 ( 0.00%)

# Completion metrics for shipping

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.374780        2.668  0.349440          2.862    655
successful completions  0.374780        2.668  0.349440          2.862    655
                                                                             
class                   0.058162       17.193  0.000000       Infinity     47
extension               0.017736       56.382  0.000000       Infinity     65
instance member         0.301070        3.321  0.274221          3.647    214
local variable          0.770115        1.299  0.764368          1.308     29
parameter               0.747021        1.339  0.742997          1.346    119
static member           0.322543        3.100  0.285992          3.497    163
top level member        0.742340        1.347  0.722222          1.385     18


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/PrefixedIdentifier
- ConstructorName/SimpleIdentifier
- MethodInvocation/SimpleIdentifier

# Completion metrics for no tables

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.342899        2.916  0.311120          3.214    655
successful completions  0.342899        2.916  0.311120          3.214    655
                                                                             
class                   0.049847       20.062  0.000000       Infinity     47
extension               0.022654       44.141  0.000000       Infinity     65
instance member         0.267702        3.735  0.235826          4.240    214
local variable          0.939655        1.064  0.939655          1.064     29
parameter               0.778645        1.284  0.772689          1.294    119
static member           0.260712        3.836  0.209305          4.778    163
top level member        0.060579       16.507  0.000000       Infinity     18


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/PrefixedIdentifier
- ConstructorName/SimpleIdentifier
- MethodInvocation/SimpleIdentifier

# Completion metrics for no tables (uniform)

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.377747        2.647  0.351247          2.847    655
successful completions  0.377747        2.647  0.351247          2.847    655
                                                                             
class                   0.049847       20.062  0.000000       Infinity     47
extension               0.012177       82.122  0.000000       Infinity     65
instance member         0.314402        3.181  0.280452          3.566    214
local variable          0.956897        1.045  0.956897          1.045     29
parameter               0.769434        1.300  0.764986          1.307    119
static member           0.347479        2.878  0.314519          3.179    163
top level member        0.058685       17.040  0.000000       Infinity     18


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/PrefixedIdentifier
- ConstructorName/SimpleIdentifier
- MethodInvocation/SimpleIdentifier

# Completion metrics for no tables (distance)

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.356759        2.803  0.330102          3.029    655
successful completions  0.356759        2.803  0.330102          3.029    655
                                                                             
class                   0.049847       20.062  0.000000       Infinity     47
extension               0.012218       81.844  0.000000       Infinity     65
instance member         0.325643        3.071  0.295483          3.384    214
local variable          0.956897        1.045  0.956897          1.045     29
parameter               0.586911        1.704  0.574930          1.739    119
static member           0.381620        2.620  0.348569          2.869    163
top level member        0.058685       17.040  0.000000       Infinity     18


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/PrefixedIdentifier
- ConstructorName/SimpleIdentifier
- MethodInvocation/SimpleIdentifier

# Completion metrics for no relevance

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.183746        5.442  0.130687          7.652    655
successful completions  0.183746        5.442  0.130687          7.652    655
                                                                             
class                   0.049829       20.069  0.000000       Infinity     47
extension               0.008428      118.650  0.000000       Infinity     65
instance member         0.251786        3.972  0.214252          4.667    214
local variable          0.160259        6.240  0.052299         19.121     29
parameter               0.155790        6.419  0.070168         14.251    119
static member           0.244422        4.091  0.183333          5.455    163
top level member        0.030783       32.485  0.000000       Infinity     18
```

### 3.2 `pkg/dds/lib` (`--prefix-length=0`)

```text
Analyzing root: "/home/ubuntu/projects/dart-sdk/sdk/pkg/dds/lib"
Computing completions at root: /home/ubuntu/projects/dart-sdk/sdk/pkg/dds/lib
Resolving 32 files...
Analyzing completion suggestions...

Metrics computed in 0:01:14.296000

# Comparison of experiments

## Comparison of inverse mean reciprocal ranks

A lower value is better, so a negative delta is good.

                        shipping  |  no tables     delta  |  no tables (uniform)     delta  |  no tables (distance)     delta  |  no relevance     delta
all completions            2.566  |      2.597     0.031  |                2.546    -0.020  |                 2.753     0.187  |         5.935     3.370
successful completions     2.538  |      2.569     0.031  |                2.518    -0.020  |                 2.723     0.185  |         5.871     3.333
                                                                                                                                                        
class                     28.483  |     25.806    -2.677  |               32.554     4.071  |                41.061    12.578  |        69.388    40.905
enum                       1.385  |      2.455     1.070  |                2.455     1.070  |                 5.324     3.939  |        12.103    10.719
extension                396.125  |    352.542   -43.584  |              337.849   -58.276  |               352.542   -43.584  |       354.309   -41.816
instance member            3.088  |      3.845     0.757  |                3.412     0.324  |                 3.293     0.205  |         4.453     1.365
local function            62.811  |     12.872   -49.939  |               25.292   -37.519  |                25.324   -37.487  |        10.266   -52.545
local variable             1.635  |      1.395    -0.239  |                1.355    -0.280  |                 1.338    -0.296  |         7.167     5.532
parameter                  1.694  |      1.666    -0.027  |                1.889     0.195  |                 3.073     1.379  |         7.725     6.032
prefix                   504.392  |     60.976  -443.416  |               60.976  -443.416  |                60.976  -443.416  |        17.626  -486.766
static member              3.614  |      3.694     0.080  |                3.241    -0.373  |                 3.151    -0.462  |         3.989     0.375
top level member          10.126  |     20.623    10.497  |               21.489    11.363  |                23.099    12.973  |        24.008    13.882
unknown                    3.429  |      1.333    -2.095  |                4.444     1.016  |                 4.444     1.016  |         1.333    -2.095

## Counts for 'relevance rank comparison' (total = 4342)

                      count  percent
shipping              2960   68.17%
no tables             477    10.99%
no tables (uniform)   386    8.89%
no tables (distance)  309    7.12%
no relevance          210    4.84%

## Comparison of other metrics

                                       shipping        no tables  no tables (uniform)  no tables (distance)     no relevance
ms per completion             0, 3.059139, 2051  0, 2.274505, 12      0, 2.220256, 14        0, 2.200000, 8   0, 0.908964, 5
chars_before_top                0, 2.290803, 31  0, 2.347381, 31      0, 2.356228, 31       0, 2.445169, 31  0, 2.506868, 23
chars_before_top_five           0, 0.836088, 31  0, 0.908265, 31      0, 0.880792, 31       0, 0.933644, 31  0, 1.131083, 23
insertion_length_theoretical    0, 7.597672, 46  0, 7.541094, 46      0, 7.532247, 46       0, 7.443306, 46  0, 7.381607, 46

shipping: [0] 4290 [10] 4 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 1
no tables: [0] 4294 [10] 1 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0
no tables (uniform): [0] 4292 [10] 3 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0
no tables (distance): [0] 4295 [10] 0 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0
no relevance: [0] 4295 [10] 0 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0

## Comparison of completion counts

                   shipping      no tables  no tables (uniform)  no tables (distance)   no relevance
total                  4342           4342                 4342                  4342           4342
successful    4295 (98.92%)  4295 (98.92%)        4295 (98.92%)         4295 (98.92%)  4295 (98.92%)
unsuccessful    47 ( 1.08%)    47 ( 1.08%)          47 ( 1.08%)           47 ( 1.08%)    47 ( 1.08%)

# Completion metrics for shipping

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.389757        2.566  0.365442          2.736   4342
successful completions  0.394022        2.538  0.369441          2.707   4295
                                                                             
class                   0.035109       28.483  0.013619         73.429    257
enum                    0.722222        1.385  0.722222          1.385      9
extension               0.002524      396.125  0.000000       Infinity      4
instance member         0.323839        3.088  0.291899          3.426   1718
local function          0.015921       62.811  0.000000       Infinity     25
local variable          0.611666        1.635  0.598292          1.671    878
parameter               0.590358        1.694  0.577800          1.731    756
prefix                  0.001983      504.392  0.000000       Infinity     40
static member           0.276735        3.614  0.233801          4.277    428
top level member        0.098754       10.126  0.070225         14.240    178
unknown                 0.291667        3.429  0.291667          3.429      2


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/ConditionalExpression
- ArgumentList/PrefixedIdentifier
- ArgumentList/PropertyAccess
- ConstructorName/SimpleIdentifier
- ListLiteral/PrefixedIdentifier
- ListLiteral/PropertyAccess
- MethodInvocation/SimpleIdentifier

## Missing tables for the following completion locations

- MapPatternEntry_key
- PatternVariableDeclaration_expression
- RecordLiteral_fields
- SwitchExpression_body
- SwitchExpression_expression
- SwitchPatternCase_pattern

# Completion metrics for no tables

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.385065        2.597  0.358372          2.790   4342
successful completions  0.389278        2.569  0.362293          2.760   4295
                                                                             
class                   0.038751       25.806  0.022049         45.353    257
enum                    0.407407        2.455  0.407407          2.455      9
extension               0.002837      352.542  0.000000       Infinity      4
instance member         0.260093        3.845  0.220809          4.529   1718
local function          0.077691       12.872  0.010000        100.000     25
local variable          0.716632        1.395  0.709434          1.410    878
parameter               0.600095        1.666  0.588360          1.700    756
prefix                  0.016400       60.976  0.000000       Infinity     40
static member           0.270721        3.694  0.221729          4.510    428
top level member        0.048489       20.623  0.017041         58.681    178
unknown                 0.750000        1.333  0.750000          1.333      2


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/ConditionalExpression
- ArgumentList/PrefixedIdentifier
- ArgumentList/PropertyAccess
- ConstructorName/SimpleIdentifier
- ListLiteral/PrefixedIdentifier
- ListLiteral/PropertyAccess
- MethodInvocation/SimpleIdentifier

## Missing tables for the following completion locations

- MapPatternEntry_key
- PatternVariableDeclaration_expression
- RecordLiteral_fields
- SwitchExpression_body
- SwitchExpression_expression
- SwitchPatternCase_pattern

# Completion metrics for no tables (uniform)

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.392802        2.546  0.366195          2.731   4342
successful completions  0.397101        2.518  0.370202          2.701   4295
                                                                             
class                   0.030718       32.554  0.015824         63.197    257
enum                    0.407407        2.455  0.407407          2.455      9
extension               0.002960      337.849  0.000000       Infinity      4
instance member         0.293057        3.412  0.252775          3.956   1718
local function          0.039538       25.292  0.000000       Infinity     25
local variable          0.737846        1.355  0.731017          1.368    878
parameter               0.529414        1.889  0.515101          1.941    756
prefix                  0.016400       60.976  0.000000       Infinity     40
static member           0.308548        3.241  0.265460          3.767    428
top level member        0.046535       21.489  0.015169         65.926    178
unknown                 0.225000        4.444  0.225000          4.444      2


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/ConditionalExpression
- ArgumentList/PrefixedIdentifier
- ArgumentList/PropertyAccess
- ConstructorName/SimpleIdentifier
- ListLiteral/PrefixedIdentifier
- ListLiteral/PropertyAccess
- MethodInvocation/SimpleIdentifier

## Missing tables for the following completion locations

- MapPatternEntry_key
- PatternVariableDeclaration_expression
- RecordLiteral_fields
- SwitchExpression_body
- SwitchExpression_expression
- SwitchPatternCase_pattern

# Completion metrics for no tables (distance)

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.363254        2.753  0.334493          2.990   4342
successful completions  0.367229        2.723  0.338153          2.957   4295
                                                                             
class                   0.024354       41.061  0.009144        109.362    257
enum                    0.187831        5.324  0.148148          6.750      9
extension               0.002837      352.542  0.000000       Infinity      4
instance member         0.303636        3.293  0.264261          3.784   1718
local function          0.039488       25.324  0.000000       Infinity     25
local variable          0.747142        1.338  0.741040          1.349    878
parameter               0.325432        3.073  0.294577          3.395    756
prefix                  0.016400       60.976  0.000000       Infinity     40
static member           0.317348        3.151  0.276168          3.621    428
top level member        0.043292       23.099  0.015169         65.926    178
unknown                 0.225000        4.444  0.225000          4.444      2


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/ConditionalExpression
- ArgumentList/PrefixedIdentifier
- ArgumentList/PropertyAccess
- ConstructorName/SimpleIdentifier
- ListLiteral/PrefixedIdentifier
- ListLiteral/PropertyAccess
- MethodInvocation/SimpleIdentifier

## Missing tables for the following completion locations

- MapPatternEntry_key
- PatternVariableDeclaration_expression
- RecordLiteral_fields
- SwitchExpression_body
- SwitchExpression_expression
- SwitchPatternCase_pattern

# Completion metrics for no relevance

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.168486        5.935  0.108487          9.218   4342
successful completions  0.170330        5.871  0.109674          9.118   4295
                                                                             
class                   0.014412       69.388  0.000000       Infinity    257
enum                    0.082621       12.103  0.000000       Infinity      9
extension               0.002822      354.309  0.000000       Infinity      4
instance member         0.224583        4.453  0.182586          5.477   1718
local function          0.097410       10.266  0.018000         55.556     25
local variable          0.139524        7.167  0.038781         25.786    878
parameter               0.129442        7.725  0.041997         23.811    756
prefix                  0.056734       17.626  0.000000       Infinity     40
static member           0.250688        3.989  0.203544          4.913    428
top level member        0.041653       24.008  0.014045         71.200    178
unknown                 0.750000        1.333  0.750000          1.333      2
```

### 3.3 `pkg/analysis_server/lib/src/lsp` (`--prefix-length=0`)

```text
Analyzing root: "/home/ubuntu/projects/dart-sdk/sdk/pkg/analysis_server/lib/src/lsp"
Computing completions at root: /home/ubuntu/projects/dart-sdk/sdk/pkg/analysis_server/lib/src/lsp
Resolving 113 files...
Analyzing completion suggestions...

Metrics computed in 0:35:06.644000

# Comparison of experiments

## Comparison of inverse mean reciprocal ranks

A lower value is better, so a negative delta is good.

                        shipping  |  no tables      delta  |  no tables (uniform)      delta  |  no tables (distance)      delta  |  no relevance      delta
all completions            2.253  |      2.369      0.117  |                2.400      0.147  |                 2.573      0.320  |         6.327      4.074
successful completions     2.235  |      2.350      0.116  |                2.381      0.146  |                 2.552      0.318  |         6.277      4.042
                                                                                                                                                            
class                      5.694  |      4.651     -1.043  |                5.161     -0.533  |                 6.411      0.717  |       284.116    278.421
enum                       1.741  |      3.342      1.601  |                3.407      1.667  |                 3.520      1.780  |        88.512     86.771
instance member            2.524  |      3.013      0.489  |                2.754      0.231  |                 2.709      0.186  |         4.535      2.011
local function           144.881  |      9.839   -135.042  |               16.822   -128.059  |                20.226   -124.655  |         5.600   -139.281
local variable             1.664  |      1.464     -0.200  |                1.422     -0.241  |                 1.426     -0.238  |         7.675      6.011
mixin                   3972.000  |   2377.000  -1595.000  |             2377.000  -1595.000  |              2377.000  -1595.000  |      2377.000  -1595.000
parameter                  1.722  |      1.640     -0.082  |                1.933      0.210  |                 2.706      0.984  |         7.731      6.009
prefix                   491.016  |    302.211   -188.804  |              380.307   -110.709  |               380.306   -110.709  |         9.733   -481.282
static member              3.698  |      3.575     -0.123  |                3.479     -0.219  |                 3.441     -0.257  |         3.704      0.006
top level member           2.937  |     27.825     24.887  |               38.010     35.073  |                55.139     52.202  |       144.384    141.446
unknown                    6.013  |      2.413     -3.601  |                9.188      3.174  |                 6.105      0.092  |         2.624     -3.390

## Counts for 'relevance rank comparison' (total = 18065)

                      count  percent
shipping              12736  70.50%
no tables             2621   14.51%
no tables (uniform)   1030   5.70%
no tables (distance)  901    4.99%
no relevance          777    4.30%

## Comparison of other metrics

                                        shipping         no tables  no tables (uniform)  no tables (distance)      no relevance
ms per completion             0, 21.183741, 3305  0, 20.617956, 99    0, 20.615891, 104     0, 20.848957, 106  0, 11.694844, 63
chars_before_top                 0, 2.353923, 32   0, 2.541625, 35      0, 2.547818, 35       0, 2.613938, 35   0, 2.861902, 35
chars_before_top_five            0, 0.945095, 28   0, 0.891865, 28      0, 0.868932, 28       0, 0.920824, 28   0, 1.358107, 27
insertion_length_theoretical     0, 7.750139, 41   0, 7.562437, 42      0, 7.556244, 41       0, 7.490124, 41   0, 7.242160, 41

shipping: [0] 6163 [10] 927 [20] 5404 [30] 4427 [40] 299 [50] 294 [60] 183 [70] 139 [80] 66 [90] 19 [100] 1
no tables: [0] 6231 [10] 870 [20] 4898 [30] 5078 [40] 298 [50] 228 [60] 130 [70] 118 [80] 55 [90] 16 [100] 0
no tables (uniform): [0] 6232 [10] 870 [20] 4479 [30] 5430 [40] 400 [50] 271 [60] 117 [70] 85 [80] 32 [90] 5 [100] 1
no tables (distance): [0] 6246 [10] 860 [20] 4309 [30] 5467 [40] 410 [50] 352 [60] 150 [70] 88 [80] 32 [90] 7 [100] 1
no relevance: [0] 6300 [10] 8868 [20] 2477 [30] 200 [40] 65 [50] 11 [60] 1 [70] 0 [80] 0 [90] 0 [100] 0

## Comparison of completion counts

                    shipping       no tables  no tables (uniform)  no tables (distance)    no relevance
total                  18065           18065                18065                 18065           18065
successful    17922 (99.21%)  17922 (99.21%)       17922 (99.21%)        17922 (99.21%)  17922 (99.21%)
unsuccessful    143 ( 0.79%)    143 ( 0.79%)         143 ( 0.79%)          143 ( 0.79%)    143 ( 0.79%)

# Completion metrics for shipping

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.443935        2.253  0.425268          2.351  18065
successful completions  0.447477        2.235  0.428661          2.333  17922
                                                                             
class                   0.175614        5.694  0.161460          6.193    813
enum                    0.574493        1.741  0.562417          1.778    251
instance member         0.396260        2.524  0.370527          2.699   6813
local function          0.006902      144.881  0.000000       Infinity     59
local variable          0.601035        1.664  0.587456          1.702   3776
mixin                   0.000252     3972.000  0.000000       Infinity      1
parameter               0.580644        1.722  0.568944          1.758   3440
prefix                  0.002037      491.016  0.000000       Infinity    208
static member           0.270389        3.698  0.234047          4.273   1167
top level member        0.340457        2.937  0.333253          3.001   1251
unknown                 0.166295        6.013  0.141259          7.079    143


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/ConditionalExpression
- ArgumentList/FunctionExpressionInvocation
- ArgumentList/PrefixedIdentifier
- ArgumentList/PropertyAccess
- ConstructorName/SimpleIdentifier
- DotShorthandConstructorInvocation/SimpleIdentifier
- DotShorthandPropertyAccess/SimpleIdentifier
- ListLiteral/PrefixedIdentifier
- ListLiteral/PropertyAccess
- MethodInvocation/SimpleIdentifier
- RecordLiteral/PrefixedIdentifier
- RecordLiteral/PropertyAccess
- SetOrMapLiteral/PrefixedIdentifier

## Missing tables for the following completion locations

- ArgumentList__unnamed
- NullAwareElement_value
- PatternVariableDeclaration_expression
- RecordLiteral_fields
- SwitchExpressionCase_expression
- SwitchExpression_body
- SwitchExpression_expression
- WhenClause_expression
- visitForEachPartsWithPattern_iterable

# Completion metrics for no tables

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.422098        2.369  0.400799          2.495  18065
successful completions  0.425466        2.350  0.403997          2.475  17922
                                                                             
class                   0.214993        4.651  0.206376          4.846    813
enum                    0.299245        3.342  0.285193          3.506    251
instance member         0.331903        3.013  0.300976          3.323   6813
local function          0.101635        9.839  0.025424         39.333     59
local variable          0.683120        1.464  0.671743          1.489   3776
mixin                   0.000421     2377.000  0.000000       Infinity      1
parameter               0.609668        1.640  0.594845          1.681   3440
prefix                  0.003309      302.211  0.000000       Infinity    208
static member           0.279707        3.575  0.242288          4.127   1167
top level member        0.035939       27.825  0.023421         42.696   1251
unknown                 0.414489        2.413  0.379021          2.638    143


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/ConditionalExpression
- ArgumentList/FunctionExpressionInvocation
- ArgumentList/PrefixedIdentifier
- ArgumentList/PropertyAccess
- ConstructorName/SimpleIdentifier
- DotShorthandConstructorInvocation/SimpleIdentifier
- DotShorthandPropertyAccess/SimpleIdentifier
- ListLiteral/PrefixedIdentifier
- ListLiteral/PropertyAccess
- MethodInvocation/SimpleIdentifier
- RecordLiteral/PrefixedIdentifier
- RecordLiteral/PropertyAccess
- SetOrMapLiteral/PrefixedIdentifier

## Missing tables for the following completion locations

- ArgumentList__unnamed
- NullAwareElement_value
- PatternVariableDeclaration_expression
- RecordLiteral_fields
- SwitchExpressionCase_expression
- SwitchExpression_body
- SwitchExpression_expression
- WhenClause_expression
- visitForEachPartsWithPattern_iterable

# Completion metrics for no tables (uniform)

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.416698        2.400  0.393829          2.539  18065
successful completions  0.420023        2.381  0.396971          2.519  17922
                                                                             
class                   0.193747        5.161  0.181406          5.512    813
enum                    0.293492        3.407  0.278619          3.589    251
instance member         0.363089        2.754  0.329072          3.039   6813
local function          0.059447       16.822  0.017797         56.190     59
local variable          0.703082        1.422  0.692368          1.444   3776
mixin                   0.000421     2377.000  0.000000       Infinity      1
parameter               0.517425        1.933  0.499365          2.003   3440
prefix                  0.002629      380.307  0.000000       Infinity    208
static member           0.287431        3.479  0.249100          4.014   1167
top level member        0.026309       38.010  0.013696         73.016   1251
unknown                 0.108840        9.188  0.098252         10.178    143


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/ConditionalExpression
- ArgumentList/FunctionExpressionInvocation
- ArgumentList/PrefixedIdentifier
- ArgumentList/PropertyAccess
- ConstructorName/SimpleIdentifier
- DotShorthandConstructorInvocation/SimpleIdentifier
- DotShorthandPropertyAccess/SimpleIdentifier
- ListLiteral/PrefixedIdentifier
- ListLiteral/PropertyAccess
- MethodInvocation/SimpleIdentifier
- RecordLiteral/PrefixedIdentifier
- RecordLiteral/PropertyAccess
- SetOrMapLiteral/PrefixedIdentifier

## Missing tables for the following completion locations

- ArgumentList__unnamed
- NullAwareElement_value
- PatternVariableDeclaration_expression
- RecordLiteral_fields
- SwitchExpressionCase_expression
- SwitchExpression_body
- SwitchExpression_expression
- WhenClause_expression
- visitForEachPartsWithPattern_iterable

# Completion metrics for no tables (distance)

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.388684        2.573  0.362837          2.756  18065
successful completions  0.391786        2.552  0.365732          2.734  17922
                                                                             
class                   0.155980        6.411  0.132329          7.557    813
enum                    0.284069        3.520  0.270916          3.691    251
instance member         0.369097        2.709  0.335449          2.981   6813
local function          0.049441       20.226  0.007627        131.111     59
local variable          0.701488        1.426  0.691133          1.447   3776
mixin                   0.000421     2377.000  0.000000       Infinity      1
parameter               0.369560        2.706  0.338391          2.955   3440
prefix                  0.002629      380.306  0.000000       Infinity    208
static member           0.290600        3.441  0.252128          3.966   1167
top level member        0.018136       55.139  0.006262        159.702   1251
unknown                 0.163790        6.105  0.121329          8.242    143


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/ConditionalExpression
- ArgumentList/FunctionExpressionInvocation
- ArgumentList/PrefixedIdentifier
- ArgumentList/PropertyAccess
- ConstructorName/SimpleIdentifier
- DotShorthandConstructorInvocation/SimpleIdentifier
- DotShorthandPropertyAccess/SimpleIdentifier
- ListLiteral/PrefixedIdentifier
- ListLiteral/PropertyAccess
- MethodInvocation/SimpleIdentifier
- RecordLiteral/PrefixedIdentifier
- RecordLiteral/PropertyAccess
- SetOrMapLiteral/PrefixedIdentifier

## Missing tables for the following completion locations

- ArgumentList__unnamed
- NullAwareElement_value
- PatternVariableDeclaration_expression
- RecordLiteral_fields
- SwitchExpressionCase_expression
- SwitchExpression_body
- SwitchExpression_expression
- WhenClause_expression
- visitForEachPartsWithPattern_iterable

# Completion metrics for no relevance

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.158051        6.327  0.100029          9.997  18065
successful completions  0.159312        6.277  0.100827          9.918  17922
                                                                             
class                   0.003520      284.116  0.000000       Infinity    813
enum                    0.011298       88.512  0.001328        753.000    251
instance member         0.220522        4.535  0.182609          5.476   6813
local function          0.178575        5.600  0.089831         11.132     59
local variable          0.130301        7.675  0.029480         33.921   3776
mixin                   0.000421     2377.000  0.000000       Infinity      1
parameter               0.129346        7.731  0.034835         28.707   3440
prefix                  0.102741        9.733  0.037500         26.667    208
static member           0.269958        3.704  0.230720          4.334   1167
top level member        0.006926      144.384  0.000320       3127.500   1251
unknown                 0.381126        2.624  0.340326          2.938    143
```

### 3.4 `pkg/analyzer/lib/src/dart/resolver` (`--prefix-length=0`)

```text
Analyzing root: "/home/ubuntu/projects/dart-sdk/sdk/pkg/analyzer/lib/src/dart/resolver"
Computing completions at root: /home/ubuntu/projects/dart-sdk/sdk/pkg/analyzer/lib/src/dart/resolver
Resolving 42 files...
Analyzing completion suggestions...

Metrics computed in 0:20:01.016000

# Comparison of experiments

## Comparison of inverse mean reciprocal ranks

A lower value is better, so a negative delta is good.

                        shipping  |  no tables      delta  |  no tables (uniform)      delta  |  no tables (distance)      delta  |  no relevance      delta
all completions            2.186  |      2.241      0.054  |                2.268      0.081  |                 2.603      0.417  |         6.304      4.117
successful completions     2.173  |      2.227      0.054  |                2.254      0.081  |                 2.587      0.414  |         6.265      4.092
                                                                                                                                                            
class                     11.445  |     10.842     -0.602  |               14.109      2.664  |                30.418     18.974  |       193.221    181.776
enum                       6.436  |      6.658      0.222  |                7.486      1.050  |                20.236     13.801  |        60.869     54.434
extension               3382.168  |   1236.272  -2145.896  |             1236.272  -2145.896  |              1236.272  -2145.896  |      1236.272  -2145.896
instance member            3.013  |      3.584      0.572  |                3.259      0.246  |                 3.220      0.208  |         5.623      2.611
local function           768.498  |     16.412   -752.086  |               43.277   -725.221  |                73.329   -695.169  |         6.924   -761.574
local variable             1.568  |      1.372     -0.196  |                1.350     -0.218  |                 1.349     -0.219  |         7.251      5.683
parameter                  1.522  |      1.534      0.012  |                1.772      0.250  |                 3.396      1.874  |         7.698      6.176
prefix                   788.166  |    326.647   -461.518  |              326.638   -461.527  |               326.638   -461.527  |        29.499   -758.667
static member              1.714  |      1.609     -0.105  |                1.572     -0.142  |                 1.551     -0.163  |         1.626     -0.088
top level member           5.136  |     97.172     92.036  |              143.535    138.399  |               229.756    224.620  |       371.120    365.984
unknown                    3.386  |      1.253     -2.133  |                4.482      1.096  |                 4.373      0.987  |         1.423     -1.962

## Counts for 'relevance rank comparison' (total = 25322)

                      count  percent
shipping              18419  72.74%
no tables             2663   10.52%
no tables (uniform)   2062   8.14%
no tables (distance)  1439   5.68%
no relevance          739    2.92%

## Comparison of other metrics

                                       shipping        no tables  no tables (uniform)  no tables (distance)     no relevance
ms per completion             0, 8.917196, 1664  0, 8.450771, 70      0, 8.469644, 62       0, 8.446480, 59  0, 4.435911, 36
chars_before_top                0, 2.185513, 42  0, 2.244636, 42      0, 2.282382, 42       0, 2.389860, 42  0, 2.811268, 42
chars_before_top_five           0, 0.657621, 27  0, 0.699539, 27      0, 0.692546, 27       0, 0.760847, 27  0, 1.123053, 27
insertion_length_theoretical    0, 8.429434, 44  0, 8.370312, 44      0, 8.332565, 44       0, 8.225087, 44  0, 7.803679, 44

shipping: [0] 10230 [10] 14774 [20] 146 [30] 10 [40] 2 [50] 4 [60] 1 [70] 0 [80] 0 [90] 0 [100] 1
no tables: [0] 11054 [10] 14013 [20] 84 [30] 5 [40] 9 [50] 2 [60] 0 [70] 1 [80] 0 [90] 0 [100] 0
no tables (uniform): [0] 11023 [10] 13895 [20] 232 [30] 8 [40] 2 [50] 6 [60] 2 [70] 0 [80] 0 [90] 0 [100] 0
no tables (distance): [0] 10944 [10] 13986 [20] 216 [30] 18 [40] 2 [50] 2 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0
no relevance: [0] 25023 [10] 139 [20] 5 [30] 1 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0

## Comparison of completion counts

                    shipping       no tables  no tables (uniform)  no tables (distance)    no relevance
total                  25322           25322                25322                 25322           25322
successful    25168 (99.39%)  25168 (99.39%)       25168 (99.39%)        25168 (99.39%)  25168 (99.39%)
unsuccessful    154 ( 0.61%)    154 ( 0.61%)         154 ( 0.61%)          154 ( 0.61%)    154 ( 0.61%)

# Completion metrics for shipping

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.457361        2.186  0.436437          2.291  25322
successful completions  0.460160        2.173  0.439107          2.277  25168
                                                                             
class                   0.087377       11.445  0.067116         14.900    334
enum                    0.155386        6.436  0.108190          9.243    175
extension               0.000296     3382.168  0.000000       Infinity      2
instance member         0.331909        3.013  0.300778          3.325  11776
local function          0.001301      768.498  0.000000       Infinity     24
local variable          0.637883        1.568  0.624520          1.601   5418
parameter               0.657045        1.522  0.647541          1.544   5416
prefix                  0.001269      788.166  0.000000       Infinity    263
static member           0.583490        1.714  0.572222          1.748    645
top level member        0.194698        5.136  0.179006          5.586   1036
unknown                 0.295359        3.386  0.289451          3.455     79


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/ConditionalExpression
- ArgumentList/FunctionExpressionInvocation
- ArgumentList/PrefixedIdentifier
- ArgumentList/PropertyAccess
- ConstructorName/SimpleIdentifier
- ListLiteral/PrefixedIdentifier
- ListLiteral/PropertyAccess
- MethodInvocation/SimpleIdentifier
- RecordLiteral/AsExpression
- RecordLiteral/PrefixedIdentifier
- RecordLiteral/PropertyAccess

## Missing tables for the following completion locations

- NullAwareElement_value
- PatternAssignment_expression
- PatternVariableDeclaration_expression
- RecordLiteral_fields
- SwitchExpressionCase_expression
- SwitchExpression_body
- SwitchExpression_expression
- WhenClause_expression

# Completion metrics for no tables

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.446276        2.241  0.424213          2.357  25322
successful completions  0.449007        2.227  0.426809          2.343  25168
                                                                             
class                   0.092231       10.842  0.065319         15.309    334
enum                    0.150195        6.658  0.129905          7.698    175
extension               0.000809     1236.272  0.000000       Infinity      2
instance member         0.278986        3.584  0.244640          4.088  11776
local function          0.060929       16.412  0.013889         72.000     24
local variable          0.729089        1.372  0.720103          1.389   5418
parameter               0.651985        1.534  0.637497          1.569   5416
prefix                  0.003061      326.647  0.000000       Infinity    263
static member           0.621472        1.609  0.607494          1.646    645
top level member        0.010291       97.172  0.007111        140.633   1036
unknown                 0.798101        1.253  0.794304          1.259     79


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/ConditionalExpression
- ArgumentList/FunctionExpressionInvocation
- ArgumentList/PrefixedIdentifier
- ArgumentList/PropertyAccess
- ConstructorName/SimpleIdentifier
- ListLiteral/PrefixedIdentifier
- ListLiteral/PropertyAccess
- MethodInvocation/SimpleIdentifier
- RecordLiteral/AsExpression
- RecordLiteral/PrefixedIdentifier
- RecordLiteral/PropertyAccess

## Missing tables for the following completion locations

- NullAwareElement_value
- PatternAssignment_expression
- PatternVariableDeclaration_expression
- RecordLiteral_fields
- SwitchExpressionCase_expression
- SwitchExpression_body
- SwitchExpression_expression
- WhenClause_expression

# Completion metrics for no tables (uniform)

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.440941        2.268  0.417060          2.398  25322
successful completions  0.443639        2.254  0.419612          2.383  25168
                                                                             
class                   0.070877       14.109  0.053693         18.625    334
enum                    0.133583        7.486  0.094667         10.563    175
extension               0.000809     1236.272  0.000000       Infinity      2
instance member         0.306829        3.259  0.269591          3.709  11776
local function          0.023107       43.277  0.010417         96.000     24
local variable          0.740690        1.350  0.732395          1.365   5418
parameter               0.564188        1.772  0.546935          1.828   5416
prefix                  0.003061      326.638  0.000000       Infinity    263
static member           0.636188        1.572  0.625685          1.598    645
top level member        0.006967      143.535  0.002043        489.449   1036
unknown                 0.223116        4.482  0.194304          5.147     79


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/ConditionalExpression
- ArgumentList/FunctionExpressionInvocation
- ArgumentList/PrefixedIdentifier
- ArgumentList/PropertyAccess
- ConstructorName/SimpleIdentifier
- ListLiteral/PrefixedIdentifier
- ListLiteral/PropertyAccess
- MethodInvocation/SimpleIdentifier
- RecordLiteral/AsExpression
- RecordLiteral/PrefixedIdentifier
- RecordLiteral/PropertyAccess

## Missing tables for the following completion locations

- NullAwareElement_value
- PatternAssignment_expression
- PatternVariableDeclaration_expression
- RecordLiteral_fields
- SwitchExpressionCase_expression
- SwitchExpression_body
- SwitchExpression_expression
- WhenClause_expression

# Completion metrics for no tables (distance)

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.384155        2.603  0.359459          2.782  25322
successful completions  0.386505        2.587  0.361659          2.765  25168
                                                                             
class                   0.032875       30.418  0.017515         57.094    334
enum                    0.049416       20.236  0.031429         31.818    175
extension               0.000809     1236.272  0.000000       Infinity      2
instance member         0.310519        3.220  0.274342          3.645  11776
local function          0.013637       73.329  0.000000       Infinity     24
local variable          0.741401        1.349  0.733699          1.363   5418
parameter               0.294455        3.396  0.269439          3.711   5416
prefix                  0.003061      326.638  0.000000       Infinity    263
static member           0.644781        1.551  0.634057          1.577    645
top level member        0.004352      229.756  0.000193       5180.000   1036
unknown                 0.228692        4.373  0.210127          4.759     79


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/ConditionalExpression
- ArgumentList/FunctionExpressionInvocation
- ArgumentList/PrefixedIdentifier
- ArgumentList/PropertyAccess
- ConstructorName/SimpleIdentifier
- ListLiteral/PrefixedIdentifier
- ListLiteral/PropertyAccess
- MethodInvocation/SimpleIdentifier
- RecordLiteral/AsExpression
- RecordLiteral/PrefixedIdentifier
- RecordLiteral/PropertyAccess

## Missing tables for the following completion locations

- NullAwareElement_value
- PatternAssignment_expression
- PatternVariableDeclaration_expression
- RecordLiteral_fields
- SwitchExpressionCase_expression
- SwitchExpression_body
- SwitchExpression_expression
- WhenClause_expression

# Completion metrics for no relevance

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.158634        6.304  0.094309         10.603  25322
successful completions  0.159605        6.265  0.094886         10.539  25168
                                                                             
class                   0.005175      193.221  0.001497        668.000    334
enum                    0.016429       60.869  0.007619        131.250    175
extension               0.000809     1236.272  0.000000       Infinity      2
instance member         0.177828        5.623  0.135093          7.402  11776
local function          0.144427        6.924  0.058333         17.143     24
local variable          0.137916        7.251  0.037219         26.868   5418
parameter               0.129897        7.698  0.027597         36.236   5416
prefix                  0.033900       29.499  0.000000       Infinity    263
static member           0.615189        1.626  0.600620          1.665    645
top level member        0.002695      371.120  0.000000       Infinity   1036
unknown                 0.702532        1.423  0.702532          1.423     79
```

## 4. Raw Output (`--prefix-length=2`, with uniform & distance variants)

### 4.1 `pkg/dtd/lib` (`--prefix-length=2`)

```text
Analyzing root: "/home/ubuntu/projects/dart-sdk/sdk/pkg/dtd/lib"
Computing completions at root: /home/ubuntu/projects/dart-sdk/sdk/pkg/dtd/lib
Resolving 10 files...
Analyzing completion suggestions...

Metrics computed in 0:00:08.012000

# Comparison of experiments

## Comparison of inverse mean reciprocal ranks

A lower value is better, so a negative delta is good.

                        shipping  |  no tables   delta  |  no tables (uniform)   delta  |  no tables (distance)   delta  |  no relevance   delta
all completions            1.228  |      1.229   0.000  |                1.202  -0.026  |                 1.203  -0.025  |         1.391   0.163
successful completions     1.129  |      1.129   0.000  |                1.105  -0.024  |                 1.106  -0.023  |         1.279   0.150
                                                                                                                                                
class                      1.232  |      1.234   0.003  |                1.234   0.003  |                 1.234   0.003  |         1.234   0.003
extension                  1.337  |      1.337   0.000  |                1.360   0.022  |                 1.360   0.022  |         1.975   0.637
instance member            1.083  |      1.074  -0.010  |                1.049  -0.035  |                 1.049  -0.035  |         1.110   0.026
local variable             1.087  |      1.048  -0.039  |                1.048  -0.039  |                 1.048  -0.039  |         1.655   0.567
parameter                  1.087  |      1.082  -0.005  |                1.083  -0.003  |                 1.088   0.001  |         1.797   0.711
static member              1.169  |      1.212   0.043  |                1.137  -0.032  |                 1.137  -0.032  |         1.137  -0.032
top level member           1.137  |      1.080  -0.057  |                1.080  -0.057  |                 1.080  -0.057  |         1.096  -0.040

## Counts for 'relevance rank comparison' (total = 655)

                     count  percent
shipping             569    86.87%
no relevance         62     9.47%
no tables            22     3.36%
no tables (uniform)  2      0.31%

## Comparison of other metrics

                                       shipping        no tables  no tables (uniform)  no tables (distance)     no relevance
ms per completion             0, 3.945183, 1982   0, 0.456811, 4       0, 0.440199, 5        0, 0.425249, 3   0, 0.357143, 3
chars_before_top                0, 0.642857, 18  0, 0.616279, 18      0, 0.586379, 18       0, 0.586379, 18  0, 1.237542, 18
chars_before_top_five            0, 0.009967, 3   0, 0.023256, 3       0, 0.019934, 3        0, 0.019934, 3   0, 0.171096, 3
insertion_length_theoretical    0, 9.255814, 32  0, 9.282392, 32      0, 9.312292, 32       0, 9.312292, 32  0, 8.661130, 31

shipping: [0] 600 [10] 1 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 1
no tables: [0] 602 [10] 0 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0
no tables (uniform): [0] 602 [10] 0 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0
no tables (distance): [0] 602 [10] 0 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0
no relevance: [0] 602 [10] 0 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0

## Comparison of completion counts

                  shipping     no tables  no tables (uniform)  no tables (distance)  no relevance
total                  655           655                  655                   655           655
successful    602 (91.91%)  602 (91.91%)         602 (91.91%)          602 (91.91%)  602 (91.91%)
unsuccessful   53 ( 8.09%)   53 ( 8.09%)          53 ( 8.09%)           53 ( 8.09%)   53 ( 8.09%)

# Completion metrics for shipping

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.814233        1.228  0.813639          1.229    655
successful completions  0.885918        1.129  0.885271          1.130    602
                                                                             
class                   0.811957        1.232  0.811957          1.232     46
extension               0.747807        1.337  0.747807          1.337     38
instance member         0.923084        1.083  0.921340          1.085    214
local variable          0.919540        1.087  0.919540          1.087     29
parameter               0.920304        1.087  0.920168          1.087    119
static member           0.855072        1.169  0.855072          1.169    138
top level member        0.879630        1.137  0.879630          1.137     18


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/PrefixedIdentifier
- ConstructorName/SimpleIdentifier
- MethodInvocation/SimpleIdentifier

# Completion metrics for no tables

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.813988        1.229  0.813359          1.229    655
successful completions  0.885651        1.129  0.884967          1.130    602
                                                                             
class                   0.810145        1.234  0.810145          1.234     46
extension               0.747807        1.337  0.747807          1.337     38
instance member         0.931365        1.074  0.929439          1.076    214
local variable          0.954023        1.048  0.954023          1.048     29
parameter               0.924370        1.082  0.924370          1.082    119
static member           0.824879        1.212  0.824879          1.212    138
top level member        0.925926        1.080  0.925926          1.080     18


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/PrefixedIdentifier
- ConstructorName/SimpleIdentifier
- MethodInvocation/SimpleIdentifier

# Completion metrics for no tables (uniform)

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.831763        1.202  0.831247          1.203    655
successful completions  0.904991        1.105  0.904430          1.106    602
                                                                             
class                   0.810145        1.234  0.810145          1.234     46
extension               0.735526        1.360  0.735526          1.360     38
instance member         0.953683        1.049  0.952103          1.050    214
local variable          0.954023        1.048  0.954023          1.048     29
parameter               0.922969        1.083  0.922969          1.083    119
static member           0.879227        1.137  0.879227          1.137    138
top level member        0.925926        1.080  0.925926          1.080     18


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/PrefixedIdentifier
- ConstructorName/SimpleIdentifier
- MethodInvocation/SimpleIdentifier

# Completion metrics for no tables (distance)

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.831127        1.203  0.830611          1.204    655
successful completions  0.904299        1.106  0.903738          1.107    602
                                                                             
class                   0.810145        1.234  0.810145          1.234     46
extension               0.735526        1.360  0.735526          1.360     38
instance member         0.953683        1.049  0.952103          1.050    214
local variable          0.954023        1.048  0.954023          1.048     29
parameter               0.919468        1.088  0.919468          1.088    119
static member           0.879227        1.137  0.879227          1.137    138
top level member        0.925926        1.080  0.925926          1.080     18


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/PrefixedIdentifier
- ConstructorName/SimpleIdentifier
- MethodInvocation/SimpleIdentifier

# Completion metrics for no relevance

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.718841        1.391  0.710204          1.408    655
successful completions  0.782127        1.279  0.772730          1.294    602
                                                                             
class                   0.810145        1.234  0.810145          1.234     46
extension               0.506410        1.975  0.493421          2.027     38
instance member         0.901142        1.110  0.897586          1.114    214
local variable          0.604367        1.655  0.586207          1.706     29
parameter               0.556382        1.797  0.525210          1.904    119
static member           0.879227        1.137  0.879227          1.137    138
top level member        0.912037        1.096  0.902778          1.108     18
```

### 4.2 `pkg/dds/lib` (`--prefix-length=2`)

```text
Analyzing root: "/home/ubuntu/projects/dart-sdk/sdk/pkg/dds/lib"
Computing completions at root: /home/ubuntu/projects/dart-sdk/sdk/pkg/dds/lib
Resolving 32 files...
Analyzing completion suggestions...

Metrics computed in 0:00:46.230000

# Comparison of experiments

## Comparison of inverse mean reciprocal ranks

A lower value is better, so a negative delta is good.

                        shipping  |  no tables   delta  |  no tables (uniform)   delta  |  no tables (distance)   delta  |  no relevance   delta
all completions            1.249  |      1.242  -0.007  |                1.238  -0.010  |                 1.260   0.011  |         1.721   0.472
successful completions     1.231  |      1.224  -0.007  |                1.220  -0.010  |                 1.242   0.011  |         1.696   0.465
                                                                                                                                                
class                      2.509  |      2.825   0.317  |                2.820   0.311  |                 2.895   0.386  |         3.309   0.801
enum                       1.000  |      1.000   0.000  |                1.000   0.000  |                 1.714   0.714  |         1.714   0.714
extension                 22.000  |     22.000   0.000  |               22.000   0.000  |                22.000   0.000  |        22.000   0.000
instance member            1.210  |      1.216   0.006  |                1.202  -0.008  |                 1.199  -0.011  |         1.243   0.034
local function             2.422  |      1.447  -0.975  |                1.730  -0.691  |                 1.730  -0.691  |         2.359  -0.062
local variable             1.088  |      1.063  -0.025  |                1.052  -0.035  |                 1.051  -0.036  |         2.467   1.380
parameter                  1.073  |      1.087   0.014  |                1.113   0.040  |                 1.220   0.147  |         2.460   1.387
prefix                     4.253  |      2.221  -2.031  |                2.221  -2.031  |                 2.221  -2.031  |         5.625   1.373
static member              1.313  |      1.305  -0.009  |                1.286  -0.027  |                 1.284  -0.029  |         1.479   0.165
top level member           1.818  |      1.631  -0.187  |                1.643  -0.174  |                 1.645  -0.173  |         1.865   0.047
unknown                    1.000  |      1.000   0.000  |                1.000   0.000  |                 1.000   0.000  |         1.000   0.000

## Counts for 'relevance rank comparison' (total = 4342)

                      count  percent
shipping              3892   89.64%
no tables             207    4.77%
no relevance          169    3.89%
no tables (uniform)   61     1.40%
no tables (distance)  13     0.30%

## Comparison of other metrics

                                       shipping        no tables  no tables (uniform)  no tables (distance)     no relevance
ms per completion             0, 1.968451, 1950  0, 1.349848, 13       0, 1.306380, 9        0, 1.291423, 8   0, 1.058191, 6
chars_before_top                0, 1.578874, 23  0, 1.618836, 23      0, 1.597102, 23       0, 1.657630, 23  0, 2.333489, 23
chars_before_top_five           0, 0.194672, 23  0, 0.352886, 23      0, 0.345875, 23       0, 0.378359, 23  0, 0.766768, 23
insertion_length_theoretical    0, 8.281374, 48  0, 8.241412, 48      0, 8.263146, 48       0, 8.202617, 48  0, 7.526759, 48

shipping: [0] 4274 [10] 4 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 1
no tables: [0] 4276 [10] 3 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0
no tables (uniform): [0] 4279 [10] 0 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0
no tables (distance): [0] 4279 [10] 0 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0
no relevance: [0] 4279 [10] 0 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0

## Comparison of completion counts

                   shipping      no tables  no tables (uniform)  no tables (distance)   no relevance
total                  4342           4342                 4342                  4342           4342
successful    4279 (98.55%)  4279 (98.55%)        4279 (98.55%)         4279 (98.55%)  4279 (98.55%)
unsuccessful    63 ( 1.45%)    63 ( 1.45%)          63 ( 1.45%)           63 ( 1.45%)    63 ( 1.45%)

# Completion metrics for shipping

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.800769        1.249  0.794998          1.258   4342
successful completions  0.812559        1.231  0.806703          1.240   4279
                                                                             
class                   0.398605        2.509  0.364902          2.740    255
enum                    1.000000        1.000  1.000000          1.000      6
extension               0.045455       22.000  0.000000       Infinity      4
instance member         0.826692        1.210  0.823525          1.214   1718
local function          0.412963        2.422  0.402000          2.488     25
local variable          0.919503        1.088  0.918700          1.088    877
parameter               0.932146        1.073  0.931325          1.074    755
prefix                  0.235149        4.253  0.200000          5.000     40
static member           0.761375        1.313  0.752307          1.329    419
top level member        0.550199        1.818  0.527528          1.896    178
unknown                 1.000000        1.000  1.000000          1.000      2


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/ConditionalExpression
- ArgumentList/PrefixedIdentifier
- ArgumentList/PropertyAccess
- ConstructorName/SimpleIdentifier
- ListLiteral/PrefixedIdentifier
- ListLiteral/PropertyAccess
- MethodInvocation/SimpleIdentifier

## Missing tables for the following completion locations

- MapPatternEntry_key
- PatternVariableDeclaration_expression
- RecordLiteral_fields
- SwitchExpression_body
- SwitchExpression_expression
- SwitchPatternCase_pattern

# Completion metrics for no tables

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.805354        1.242  0.799651          1.251   4342
successful completions  0.817212        1.224  0.811424          1.232   4279
                                                                             
class                   0.353941        2.825  0.314641          3.178    255
enum                    1.000000        1.000  1.000000          1.000      6
extension               0.045455       22.000  0.000000       Infinity      4
instance member         0.822389        1.216  0.819451          1.220   1718
local function          0.691282        1.447  0.683333          1.463     25
local variable          0.940904        1.063  0.940707          1.063    877
parameter               0.920217        1.087  0.918874          1.088    755
prefix                  0.450159        2.221  0.425000          2.353     40
static member           0.766485        1.305  0.757478          1.320    419
top level member        0.613289        1.631  0.594476          1.682    178
unknown                 1.000000        1.000  1.000000          1.000      2


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/ConditionalExpression
- ArgumentList/PrefixedIdentifier
- ArgumentList/PropertyAccess
- ConstructorName/SimpleIdentifier
- ListLiteral/PrefixedIdentifier
- ListLiteral/PropertyAccess
- MethodInvocation/SimpleIdentifier

## Missing tables for the following completion locations

- MapPatternEntry_key
- PatternVariableDeclaration_expression
- RecordLiteral_fields
- SwitchExpression_body
- SwitchExpression_expression
- SwitchPatternCase_pattern

# Completion metrics for no tables (uniform)

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.807551        1.238  0.802265          1.246   4342
successful completions  0.819441        1.220  0.814076          1.228   4279
                                                                             
class                   0.354595        2.820  0.314641          3.178    255
enum                    1.000000        1.000  1.000000          1.000      6
extension               0.045455       22.000  0.000000       Infinity      4
instance member         0.831969        1.202  0.830200          1.205   1718
local function          0.577949        1.730  0.570000          1.754     25
local variable          0.950456        1.052  0.950399          1.052    877
parameter               0.898535        1.113  0.896799          1.115    755
prefix                  0.450159        2.221  0.425000          2.353     40
static member           0.777398        1.286  0.768735          1.301    419
top level member        0.608607        1.643  0.589794          1.696    178
unknown                 1.000000        1.000  1.000000          1.000      2


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/ConditionalExpression
- ArgumentList/PrefixedIdentifier
- ArgumentList/PropertyAccess
- ConstructorName/SimpleIdentifier
- ListLiteral/PrefixedIdentifier
- ListLiteral/PropertyAccess
- MethodInvocation/SimpleIdentifier

## Missing tables for the following completion locations

- MapPatternEntry_key
- PatternVariableDeclaration_expression
- RecordLiteral_fields
- SwitchExpression_body
- SwitchExpression_expression
- SwitchPatternCase_pattern

# Completion metrics for no tables (distance)

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.793741        1.260  0.787713          1.269   4342
successful completions  0.805427        1.242  0.799311          1.251   4279
                                                                             
class                   0.345444        2.895  0.305490          3.273    255
enum                    0.583333        1.714  0.583333          1.714      6
extension               0.045455       22.000  0.000000       Infinity      4
instance member         0.834007        1.199  0.832334          1.201   1718
local function          0.577949        1.730  0.570000          1.754     25
local variable          0.951029        1.051  0.950969          1.052    877
parameter               0.819673        1.220  0.813444          1.229    755
prefix                  0.450159        2.221  0.425000          2.353     40
static member           0.778591        1.284  0.769928          1.299    419
top level member        0.608077        1.645  0.589326          1.697    178
unknown                 1.000000        1.000  1.000000          1.000      2


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/ConditionalExpression
- ArgumentList/PrefixedIdentifier
- ArgumentList/PropertyAccess
- ConstructorName/SimpleIdentifier
- ListLiteral/PrefixedIdentifier
- ListLiteral/PropertyAccess
- MethodInvocation/SimpleIdentifier

## Missing tables for the following completion locations

- MapPatternEntry_key
- PatternVariableDeclaration_expression
- RecordLiteral_fields
- SwitchExpression_body
- SwitchExpression_expression
- SwitchPatternCase_pattern

# Completion metrics for no relevance

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.581121        1.721  0.565269          1.769   4342
successful completions  0.589677        1.696  0.573592          1.743   4279
                                                                             
class                   0.302174        3.309  0.258954          3.862    255
enum                    0.583333        1.714  0.583333          1.714      6
extension               0.045455       22.000  0.000000       Infinity      4
instance member         0.804183        1.243  0.801261          1.248   1718
local function          0.423887        2.359  0.410000          2.439     25
local variable          0.405298        2.467  0.381452          2.622    877
parameter               0.406539        2.460  0.380508          2.628    755
prefix                  0.177764        5.625  0.137500          7.273     40
static member           0.676304        1.479  0.662609          1.509    419
top level member        0.536196        1.865  0.511798          1.954    178
unknown                 1.000000        1.000  1.000000          1.000      2
```

### 4.3 `pkg/analysis_server/lib/src/lsp` (`--prefix-length=2`)

```text
Analyzing root: "/home/ubuntu/projects/dart-sdk/sdk/pkg/analysis_server/lib/src/lsp"
Computing completions at root: /home/ubuntu/projects/dart-sdk/sdk/pkg/analysis_server/lib/src/lsp
Resolving 113 files...
Analyzing completion suggestions...

Metrics computed in 0:24:51.336000

# Comparison of experiments

## Comparison of inverse mean reciprocal ranks

A lower value is better, so a negative delta is good.

                        shipping  |  no tables   delta  |  no tables (uniform)   delta  |  no tables (distance)   delta  |  no relevance   delta
all completions            1.237  |      1.231  -0.006  |                1.199  -0.037  |                 1.207  -0.030  |         2.174   0.938
successful completions     1.214  |      1.208  -0.006  |                1.177  -0.037  |                 1.184  -0.029  |         2.134   0.920
                                                                                                                                                
class                      1.823  |      1.870   0.047  |                1.885   0.062  |                 1.902   0.079  |         5.904   4.081
enum                       1.907  |      2.025   0.118  |                2.021   0.114  |                 2.036   0.128  |         3.370   1.463
instance member            1.236  |      1.270   0.035  |                1.172  -0.064  |                 1.170  -0.065  |         1.415   0.179
local function             2.588  |      1.504  -1.084  |                1.633  -0.955  |                 1.633  -0.955  |         3.560   0.972
local variable             1.081  |      1.066  -0.014  |                1.061  -0.020  |                 1.061  -0.020  |         4.309   3.228
mixin                     16.000  |     16.000   0.000  |               16.000   0.000  |                16.000   0.000  |        16.000   0.000
parameter                  1.056  |      1.053  -0.003  |                1.069   0.013  |                 1.101   0.045  |         5.630   4.574
prefix                     5.993  |      5.065  -0.928  |                5.153  -0.840  |                 5.153  -0.840  |         1.637  -4.356
static member              1.306  |      1.313   0.007  |                1.304  -0.002  |                 1.304  -0.002  |         1.347   0.041
top level member           1.575  |      1.311  -0.264  |                1.352  -0.223  |                 1.357  -0.218  |         1.723   0.148
unknown                    1.021  |      1.021  -0.000  |                1.029   0.007  |                 1.029   0.007  |         1.021   0.000

## Counts for 'relevance rank comparison' (total = 18065)

                      count  percent
shipping              16058  88.89%
no tables             929    5.14%
no relevance          615    3.40%
no tables (uniform)   441    2.44%
no tables (distance)  22     0.12%

## Comparison of other metrics

                                        shipping         no tables  no tables (uniform)  no tables (distance)      no relevance
ms per completion             0, 16.531254, 3214  0, 16.098443, 87     0, 16.037008, 95     0, 15.995656, 102  0, 14.279759, 91
chars_before_top                 0, 1.618639, 29   0, 1.718211, 29      0, 1.671387, 29       0, 1.706307, 29   0, 3.181259, 29
chars_before_top_five            0, 0.381868, 20   0, 0.401896, 20      0, 0.361729, 20       0, 0.362688, 20   0, 1.305427, 20
insertion_length_theoretical     0, 8.455376, 43   0, 8.355805, 43      0, 8.402629, 43       0, 8.367708, 43   0, 6.892756, 42

shipping: [0] 6375 [10] 3352 [20] 5615 [30] 1834 [40] 504 [50] 36 [60] 3 [70] 6 [80] 0 [90] 0 [100] 1
no tables: [0] 6377 [10] 3393 [20] 5561 [30] 1866 [40] 490 [50] 30 [60] 3 [70] 5 [80] 1 [90] 0 [100] 0
no tables (uniform): [0] 6373 [10] 3415 [20] 5530 [30] 1859 [40] 508 [50] 29 [60] 5 [70] 5 [80] 1 [90] 1 [100] 0
no tables (distance): [0] 6376 [10] 3414 [20] 5516 [30] 1886 [40] 488 [50] 29 [60] 7 [70] 5 [80] 2 [90] 2 [100] 1
no relevance: [0] 6377 [10] 4476 [20] 5610 [30] 1167 [40] 83 [50] 7 [60] 4 [70] 1 [80] 0 [90] 1 [100] 0

## Comparison of completion counts

                    shipping       no tables  no tables (uniform)  no tables (distance)    no relevance
total                  18065           18065                18065                 18065           18065
successful    17726 (98.12%)  17726 (98.12%)       17726 (98.12%)        17726 (98.12%)  17726 (98.12%)
unsuccessful    339 ( 1.88%)    339 ( 1.88%)         339 ( 1.88%)          339 ( 1.88%)    339 ( 1.88%)

# Completion metrics for shipping

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.808562        1.237  0.802694          1.246  18065
successful completions  0.824026        1.214  0.818045          1.222  17726
                                                                             
class                   0.548596        1.823  0.537657          1.860    798
enum                    0.524351        1.907  0.468778          2.133    150
instance member         0.809331        1.236  0.802773          1.246   6755
local function          0.386340        2.588  0.355932          2.810     59
local variable          0.925309        1.081  0.924743          1.081   3768
mixin                   0.062500       16.000  0.000000       Infinity      1
parameter               0.946980        1.056  0.945888          1.057   3437
prefix                  0.166865        5.993  0.156000          6.410    200
static member           0.765614        1.306  0.756927          1.321   1167
top level member        0.634752        1.575  0.615051          1.626   1248
unknown                 0.979094        1.021  0.979021          1.021    143


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/ConditionalExpression
- ArgumentList/FunctionExpressionInvocation
- ArgumentList/PrefixedIdentifier
- ArgumentList/PropertyAccess
- ConstructorName/SimpleIdentifier
- DotShorthandPropertyAccess/SimpleIdentifier
- ListLiteral/PrefixedIdentifier
- ListLiteral/PropertyAccess
- MethodInvocation/SimpleIdentifier
- RecordLiteral/PrefixedIdentifier
- RecordLiteral/PropertyAccess
- SetOrMapLiteral/PrefixedIdentifier

## Missing tables for the following completion locations

- ArgumentList__unnamed
- NullAwareElement_value
- PatternVariableDeclaration_expression
- RecordLiteral_fields
- SwitchExpressionCase_expression
- SwitchExpression_body
- SwitchExpression_expression
- WhenClause_expression
- visitForEachPartsWithPattern_iterable

# Completion metrics for no tables

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.812453        1.231  0.807064          1.239  18065
successful completions  0.827991        1.208  0.822499          1.216  17726
                                                                             
class                   0.534798        1.870  0.524749          1.906    798
enum                    0.493707        2.025  0.433333          2.308    150
instance member         0.787245        1.270  0.780972          1.280   6755
local function          0.664783        1.504  0.659887          1.515     59
local variable          0.937845        1.066  0.937279          1.067   3768
mixin                   0.062500       16.000  0.000000       Infinity      1
parameter               0.950021        1.053  0.948279          1.055   3437
prefix                  0.197440        5.065  0.186833          5.352    200
static member           0.761386        1.313  0.752399          1.329   1167
top level member        0.762776        1.311  0.749319          1.335   1248
unknown                 0.979233        1.021  0.979021          1.021    143


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/ConditionalExpression
- ArgumentList/FunctionExpressionInvocation
- ArgumentList/PrefixedIdentifier
- ArgumentList/PropertyAccess
- ConstructorName/SimpleIdentifier
- DotShorthandPropertyAccess/SimpleIdentifier
- ListLiteral/PrefixedIdentifier
- ListLiteral/PropertyAccess
- MethodInvocation/SimpleIdentifier
- RecordLiteral/PrefixedIdentifier
- RecordLiteral/PropertyAccess
- SetOrMapLiteral/PrefixedIdentifier

## Missing tables for the following completion locations

- ArgumentList__unnamed
- NullAwareElement_value
- PatternVariableDeclaration_expression
- RecordLiteral_fields
- SwitchExpressionCase_expression
- SwitchExpression_body
- SwitchExpression_expression
- WhenClause_expression
- visitForEachPartsWithPattern_iterable

# Completion metrics for no tables (uniform)

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.833700        1.199  0.829148          1.206  18065
successful completions  0.849644        1.177  0.845005          1.183  17726
                                                                             
class                   0.530428        1.885  0.520071          1.923    798
enum                    0.494697        2.021  0.434444          2.302    150
instance member         0.853256        1.172  0.849494          1.177   6755
local function          0.612233        1.633  0.606215          1.650     59
local variable          0.942651        1.061  0.942122          1.061   3768
mixin                   0.062500       16.000  0.000000       Infinity      1
parameter               0.935650        1.069  0.933547          1.071   3437
prefix                  0.194075        5.153  0.182667          5.474    200
static member           0.766896        1.304  0.758112          1.319   1167
top level member        0.739465        1.352  0.725561          1.378   1248
unknown                 0.972100        1.029  0.972028          1.029    143


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/ConditionalExpression
- ArgumentList/FunctionExpressionInvocation
- ArgumentList/PrefixedIdentifier
- ArgumentList/PropertyAccess
- ConstructorName/SimpleIdentifier
- DotShorthandPropertyAccess/SimpleIdentifier
- ListLiteral/PrefixedIdentifier
- ListLiteral/PropertyAccess
- MethodInvocation/SimpleIdentifier
- RecordLiteral/PrefixedIdentifier
- RecordLiteral/PropertyAccess
- SetOrMapLiteral/PrefixedIdentifier

## Missing tables for the following completion locations

- ArgumentList__unnamed
- NullAwareElement_value
- PatternVariableDeclaration_expression
- RecordLiteral_fields
- SwitchExpressionCase_expression
- SwitchExpression_body
- SwitchExpression_expression
- WhenClause_expression
- visitForEachPartsWithPattern_iterable

# Completion metrics for no tables (distance)

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.828499        1.207  0.823861          1.214  18065
successful completions  0.844343        1.184  0.839617          1.191  17726
                                                                             
class                   0.525828        1.902  0.515476          1.940    798
enum                    0.491252        2.036  0.431000          2.320    150
instance member         0.854447        1.170  0.850716          1.175   6755
local function          0.612233        1.633  0.606215          1.650     59
local variable          0.942576        1.061  0.942047          1.062   3768
mixin                   0.062500       16.000  0.000000       Infinity      1
parameter               0.908224        1.101  0.905620          1.104   3437
prefix                  0.194075        5.153  0.182667          5.474    200
static member           0.766896        1.304  0.758112          1.319   1167
top level member        0.736820        1.357  0.722917          1.383   1248
unknown                 0.972240        1.029  0.972028          1.029    143


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/ConditionalExpression
- ArgumentList/FunctionExpressionInvocation
- ArgumentList/PrefixedIdentifier
- ArgumentList/PropertyAccess
- ConstructorName/SimpleIdentifier
- DotShorthandPropertyAccess/SimpleIdentifier
- ListLiteral/PrefixedIdentifier
- ListLiteral/PropertyAccess
- MethodInvocation/SimpleIdentifier
- RecordLiteral/PrefixedIdentifier
- RecordLiteral/PropertyAccess
- SetOrMapLiteral/PrefixedIdentifier

## Missing tables for the following completion locations

- ArgumentList__unnamed
- NullAwareElement_value
- PatternVariableDeclaration_expression
- RecordLiteral_fields
- SwitchExpressionCase_expression
- SwitchExpression_body
- SwitchExpression_expression
- WhenClause_expression
- visitForEachPartsWithPattern_iterable

# Completion metrics for no relevance

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.459903        2.174  0.445174          2.246  18065
successful completions  0.468698        2.134  0.453688          2.204  17726
                                                                             
class                   0.169381        5.904  0.137615          7.267    798
enum                    0.296712        3.370  0.221667          4.511    150
instance member         0.706778        1.415  0.697893          1.433   6755
local function          0.280907        3.560  0.271751          3.680     59
local variable          0.232078        4.309  0.215862          4.633   3768
mixin                   0.062500       16.000  0.000000       Infinity      1
parameter               0.177609        5.630  0.155223          6.442   3437
prefix                  0.610946        1.637  0.606833          1.648    200
static member           0.742308        1.347  0.732976          1.364   1167
top level member        0.580260        1.723  0.564971          1.770   1248
unknown                 0.979058        1.021  0.979021          1.021    143
```

### 4.4 `pkg/analyzer/lib/src/dart/resolver` (`--prefix-length=2`)

```text
Analyzing root: "/home/ubuntu/projects/dart-sdk/sdk/pkg/analyzer/lib/src/dart/resolver"
Computing completions at root: /home/ubuntu/projects/dart-sdk/sdk/pkg/analyzer/lib/src/dart/resolver
Resolving 42 files...
Analyzing completion suggestions...

Metrics computed in 0:12:16.451000

# Comparison of experiments

## Comparison of inverse mean reciprocal ranks

A lower value is better, so a negative delta is good.

                        shipping  |  no tables    delta  |  no tables (uniform)    delta  |  no tables (distance)    delta  |  no relevance    delta
all completions            1.225  |      1.240    0.015  |                1.221   -0.003  |                 1.227    0.002  |         2.153    0.929
successful completions     1.217  |      1.232    0.015  |                1.214   -0.003  |                 1.219    0.002  |         2.140    0.923
                                                                                                                                                    
class                      2.017  |      1.938   -0.080  |                2.000   -0.018  |                 2.008   -0.010  |         4.682    2.665
enum                       1.327  |      2.172    0.845  |                2.166    0.839  |                 2.450    1.123  |         6.562    5.235
extension                140.456  |     69.468  -70.988  |               69.468  -70.988  |                69.468  -70.988  |        69.468  -70.988
instance member            1.300  |      1.346    0.046  |                1.293   -0.006  |                 1.293   -0.007  |         1.442    0.142
local function             1.894  |      1.659   -0.236  |                1.659   -0.235  |                 1.691   -0.204  |         1.592   -0.302
local variable             1.063  |      1.048   -0.016  |                1.045   -0.019  |                 1.044   -0.020  |         3.766    2.703
parameter                  1.026  |      1.023   -0.003  |                1.033    0.007  |                 1.051    0.025  |         5.076    4.050
prefix                    34.329  |     20.844  -13.485  |               26.895   -7.434  |                26.895   -7.434  |         3.999  -30.330
static member              1.140  |      1.136   -0.004  |                1.136   -0.004  |                 1.136   -0.004  |         1.147    0.007
top level member           3.629  |      3.663    0.034  |                3.673    0.043  |                 3.686    0.057  |         3.788    0.159
unknown                    1.013  |      1.013    0.000  |                1.013    0.000  |                 1.013    0.000  |         1.039    0.027

## Counts for 'relevance rank comparison' (total = 25322)

                      count  percent
shipping              23826  94.09%
no tables             709    2.80%
no relevance          435    1.72%
no tables (uniform)   298    1.18%
no tables (distance)  54     0.21%

## Comparison of other metrics

                                       shipping        no tables  no tables (uniform)  no tables (distance)     no relevance
ms per completion             0, 5.570730, 1551  0, 5.303425, 27      0, 5.243980, 19       0, 5.210323, 29  0, 4.613248, 15
chars_before_top                0, 1.692442, 42  0, 1.753040, 42      0, 1.736072, 42       0, 1.760033, 42  0, 2.753437, 42
chars_before_top_five           0, 0.275411, 27  0, 0.307160, 27      0, 0.269053, 27       0, 0.276127, 27  0, 0.816181, 29
insertion_length_theoretical    0, 8.922554, 45  0, 8.861957, 45      0, 8.878924, 45       0, 8.854963, 45  0, 7.861559, 45

shipping: [0] 20979 [10] 4186 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 1
no tables: [0] 21628 [10] 3536 [20] 2 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0
no tables (uniform): [0] 21759 [10] 3407 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0
no tables (distance): [0] 21862 [10] 3303 [20] 1 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0
no relevance: [0] 23034 [10] 2132 [20] 0 [30] 0 [40] 0 [50] 0 [60] 0 [70] 0 [80] 0 [90] 0 [100] 0

## Comparison of completion counts

                    shipping       no tables  no tables (uniform)  no tables (distance)    no relevance
total                  25322           25322                25322                 25322           25322
successful    25166 (99.38%)  25166 (99.38%)       25166 (99.38%)        25166 (99.38%)  25166 (99.38%)
unsuccessful    156 ( 0.62%)    156 ( 0.62%)         156 ( 0.62%)          156 ( 0.62%)    156 ( 0.62%)

# Completion metrics for shipping

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.816572        1.225  0.811670          1.232  25322
successful completions  0.821634        1.217  0.816701          1.224  25166
                                                                             
class                   0.495680        2.017  0.468313          2.135    334
enum                    0.753390        1.327  0.750476          1.332    175
extension               0.007120      140.456  0.000000       Infinity      2
instance member         0.769246        1.300  0.761124          1.314  11776
local function          0.527848        1.894  0.516667          1.935     24
local variable          0.940297        1.063  0.939948          1.064   5417
parameter               0.974718        1.026  0.974561          1.026   5415
prefix                  0.029130       34.329  0.005070        197.250    263
static member           0.877056        1.140  0.876382          1.141    645
top level member        0.275543        3.629  0.266795          3.748   1036
unknown                 0.987342        1.013  0.987342          1.013     79


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/ConditionalExpression
- ArgumentList/FunctionExpressionInvocation
- ArgumentList/PrefixedIdentifier
- ArgumentList/PropertyAccess
- ConstructorName/SimpleIdentifier
- ListLiteral/PrefixedIdentifier
- ListLiteral/PropertyAccess
- MethodInvocation/SimpleIdentifier
- RecordLiteral/AsExpression
- RecordLiteral/PrefixedIdentifier
- RecordLiteral/PropertyAccess

## Missing tables for the following completion locations

- NullAwareElement_value
- PatternAssignment_expression
- PatternVariableDeclaration_expression
- RecordLiteral_fields
- SwitchExpressionCase_expression
- SwitchExpression_body
- SwitchExpression_expression
- WhenClause_expression

# Completion metrics for no tables

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.806390        1.240  0.801659          1.247  25322
successful completions  0.811389        1.232  0.806628          1.240  25166
                                                                             
class                   0.516087        1.938  0.498453          2.006    334
enum                    0.460430        2.172  0.454190          2.202    175
extension               0.014395       69.468  0.000000       Infinity      2
instance member         0.742888        1.346  0.734785          1.361  11776
local function          0.602944        1.659  0.597222          1.674     24
local variable          0.954370        1.048  0.954194          1.048   5417
parameter               0.977416        1.023  0.977408          1.023   5415
prefix                  0.047974       20.844  0.024081         41.526    263
static member           0.880276        1.136  0.879742          1.137    645
top level member        0.272971        3.663  0.263674          3.793   1036
unknown                 0.987342        1.013  0.987342          1.013     79


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/ConditionalExpression
- ArgumentList/FunctionExpressionInvocation
- ArgumentList/PrefixedIdentifier
- ArgumentList/PropertyAccess
- ConstructorName/SimpleIdentifier
- ListLiteral/PrefixedIdentifier
- ListLiteral/PropertyAccess
- MethodInvocation/SimpleIdentifier
- RecordLiteral/AsExpression
- RecordLiteral/PrefixedIdentifier
- RecordLiteral/PropertyAccess

## Missing tables for the following completion locations

- NullAwareElement_value
- PatternAssignment_expression
- PatternVariableDeclaration_expression
- RecordLiteral_fields
- SwitchExpressionCase_expression
- SwitchExpression_body
- SwitchExpression_expression
- WhenClause_expression

# Completion metrics for no tables (uniform)

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.818777        1.221  0.814547          1.228  25322
successful completions  0.823852        1.214  0.819596          1.220  25166
                                                                             
class                   0.500024        2.000  0.482884          2.071    334
enum                    0.461699        2.166  0.454190          2.202    175
extension               0.014395       69.468  0.000000       Infinity      2
instance member         0.773110        1.293  0.766109          1.305  11776
local function          0.602699        1.659  0.597222          1.674     24
local variable          0.957355        1.045  0.957178          1.045   5417
parameter               0.968234        1.033  0.968156          1.033   5415
prefix                  0.037181       26.895  0.012674         78.900    263
static member           0.880276        1.136  0.879742          1.137    645
top level member        0.272284        3.673  0.263304          3.798   1036
unknown                 0.987342        1.013  0.987342          1.013     79


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/ConditionalExpression
- ArgumentList/FunctionExpressionInvocation
- ArgumentList/PrefixedIdentifier
- ArgumentList/PropertyAccess
- ConstructorName/SimpleIdentifier
- ListLiteral/PrefixedIdentifier
- ListLiteral/PropertyAccess
- MethodInvocation/SimpleIdentifier
- RecordLiteral/AsExpression
- RecordLiteral/PrefixedIdentifier
- RecordLiteral/PropertyAccess

## Missing tables for the following completion locations

- NullAwareElement_value
- PatternAssignment_expression
- PatternVariableDeclaration_expression
- RecordLiteral_fields
- SwitchExpressionCase_expression
- SwitchExpression_body
- SwitchExpression_expression
- WhenClause_expression

# Completion metrics for no tables (distance)

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.815098        1.227  0.810634          1.234  25322
successful completions  0.820151        1.219  0.815659          1.226  25166
                                                                             
class                   0.498028        2.008  0.480888          2.079    334
enum                    0.408080        2.450  0.393429          2.542    175
extension               0.014395       69.468  0.000000       Infinity      2
instance member         0.773599        1.293  0.766626          1.304  11776
local function          0.591415        1.691  0.583333          1.714     24
local variable          0.958023        1.044  0.957947          1.044   5417
parameter               0.951391        1.051  0.950299          1.052   5415
prefix                  0.037181       26.895  0.012674         78.900    263
static member           0.880276        1.136  0.879742          1.137    645
top level member        0.271319        3.686  0.262339          3.812   1036
unknown                 0.987342        1.013  0.987342          1.013     79


## Missing completion location in the following places

- ArgumentList/AsExpression
- ArgumentList/ConditionalExpression
- ArgumentList/FunctionExpressionInvocation
- ArgumentList/PrefixedIdentifier
- ArgumentList/PropertyAccess
- ConstructorName/SimpleIdentifier
- ListLiteral/PrefixedIdentifier
- ListLiteral/PropertyAccess
- MethodInvocation/SimpleIdentifier
- RecordLiteral/AsExpression
- RecordLiteral/PrefixedIdentifier
- RecordLiteral/PropertyAccess

## Missing tables for the following completion locations

- NullAwareElement_value
- PatternAssignment_expression
- PatternVariableDeclaration_expression
- RecordLiteral_fields
- SwitchExpressionCase_expression
- SwitchExpression_body
- SwitchExpression_expression
- WhenClause_expression

# Completion metrics for no relevance

## Mean Reciprocal Rank

                        mrr       inverse mrr  mrr_5     inverse mrr_5  count
all completions         0.464414        2.153  0.450804          2.218  25322
successful completions  0.467293        2.140  0.453598          2.205  25166
                                                                             
class                   0.213584        4.682  0.192715          5.189    334
enum                    0.152390        6.562  0.127143          7.865    175
extension               0.014395       69.468  0.000000       Infinity      2
instance member         0.693310        1.442  0.684132          1.462  11776
local function          0.627944        1.592  0.625000          1.600     24
local variable          0.265515        3.766  0.244785          4.085   5417
parameter               0.196991        5.076  0.178538          5.601   5415
prefix                  0.250069        3.999  0.236882          4.222    263
static member           0.871464        1.147  0.870594          1.149    645
top level member        0.264004        3.788  0.255486          3.914   1036
unknown                 0.962025        1.039  0.962025          1.039     79
```
