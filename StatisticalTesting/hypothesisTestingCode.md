**standalone synthetic benchmark pipelines** 
with resampling, permutation testing, ROC/AUC, change-point detection, and runtime/memory profiling.

---

## 01 — Yang–Mills: Spectral Zeta Moments and Carleman Indeterminacy

```python
# file: 01_yang_mills_spectral_zeta_indeterminacy.py
import time, tracemalloc
import numpy as np
from scipy import stats

def bench():
    tracemalloc.start()
    t0 = time.perf_counter()
    return t0

def end_bench(t0):
    cur, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    return {"runtime_ms": (time.perf_counter() - t0) * 1e3, "peak_mb": peak / 1024 / 1024}

def carleman_like_sum(moments):
    k = np.arange(1, len(moments) + 1)
    x = np.maximum(np.asarray(moments), 1e-12)
    return float(np.sum(x ** (-1.0 / (2.0 * k))))

def bootstrap_slope(data, ks, n_boot=500, seed=0):
    r = np.random.default_rng(seed)
    vals = []
    for _ in range(n_boot):
        sample = r.choice(data, size=len(data), replace=True)
        vals.append(stats.linregress(np.log(ks), np.log(sample)).slope)
    return float(np.mean(vals)), tuple(np.quantile(vals, [0.025, 0.975]))

t0 = bench()
r = np.random.default_rng(1)
n, K = 300, 20
ks = np.arange(1, K + 1)

mom_h0 = np.exp(0.08 * ks) * r.lognormal(mean=0.0, sigma=0.08, size=(n, K))
mom_h1 = np.exp(0.18 * ks) * r.lognormal(mean=0.0, sigma=0.12, size=(n, K))

slope_h0 = np.array([stats.linregress(np.log(ks), np.log(row)).slope for row in mom_h0])
slope_h1 = np.array([stats.linregress(np.log(ks), np.log(row)).slope for row in mom_h1])

obs = slope_h0.mean() - slope_h1.mean()
p_perm = (np.sum(np.abs(r.permutation(np.r_[slope_h0, slope_h1])[:n].mean() - r.permutation(np.r_[slope_h0, slope_h1])[n:].mean()) >= abs(obs)) + 1) / 2

c0 = np.array([carleman_like_sum(row) for row in mom_h0])
c1 = np.array([carleman_like_sum(row) for row in mom_h1])

boot_h0 = bootstrap_slope(mom_h0.mean(axis=0), ks, seed=1)
boot_h1 = bootstrap_slope(mom_h1.mean(axis=0), ks, seed=2)

out = {
    "slope_h0_mean": float(slope_h0.mean()),
    "slope_h1_mean": float(slope_h1.mean()),
    "slope_gap": float(obs),
    "bootstrap_ci_h0": boot_h0[1],
    "bootstrap_ci_h1": boot_h1[1],
    "carleman_h0_mean": float(c0.mean()),
    "carleman_h1_mean": float(c1.mean()),
    "identifiability_flag": bool(abs(obs) > 0.02),
}
out.update(end_bench(t0))
print(out)
```

---

## 02 — Yang–Mills: TDA Persistence Gap Detection

```python
# file: 02_yang_mills_tda_gap_detection.py
import time, tracemalloc
import numpy as np
from scipy import stats
from sklearn.metrics import roc_auc_score, average_precision_score

def bench():
    tracemalloc.start()
    return time.perf_counter()

def end_bench(t0):
    cur, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    return {"runtime_ms": (time.perf_counter() - t0) * 1e3, "peak_mb": peak / 1024 / 1024}

def persistence_proxy(x):
    x = np.asarray(x)
    return float(np.sum(np.maximum(x - np.median(x), 0.0)))

t0 = bench()
r = np.random.default_rng(2)
n, T = 400, 120
gap, nogap = [], []

for _ in range(n):
    x_gap = np.cumsum(r.normal(0, 0.18, T)) + 1.4 * np.exp(-np.linspace(0, 4, T))
    x_nogap = np.cumsum(r.normal(0, 0.18, T))
    gap.append(persistence_proxy(x_gap))
    nogap.append(persistence_proxy(x_nogap))

gap = np.array(gap)
nogap = np.array(nogap)

obs = gap.mean() - nogap.mean()
p_perm = stats.permutation_test((gap, nogap), lambda a, b: np.mean(a) - np.mean(b),
                                 permutation_type="independent", alternative="two-sided",
                                 n_resamples=3000, random_state=2).pvalue

y_true = np.r_[np.ones(n), np.zeros(n)]
scores = np.r_[gap, nogap]
auc = roc_auc_score(y_true, scores)
ap = average_precision_score(y_true, scores)
w = stats.wasserstein_distance(gap, nogap)

out = {
    "persistence_gap": float(obs),
    "perm_p": float(p_perm),
    "roc_auc": float(auc),
    "pr_auc": float(ap),
    "wasserstein": float(w),
    "gap_proxy_threshold": float(np.median(scores)),
}
out.update(end_bench(t0))
print(out)
```

---

## 03 — Yang–Mills: Borel Stability Bounds

```python
# file: 03_yang_mills_borel_stability.py
import time, tracemalloc
import numpy as np
from scipy import stats

def bench():
    tracemalloc.start()
    return time.perf_counter()

def end_bench(t0):
    cur, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    return {"runtime_ms": (time.perf_counter() - t0) * 1e3, "peak_mb": peak / 1024 / 1024}

t0 = bench()
r = np.random.default_rng(3)

x = np.linspace(0.0, 0.96, 300)
lead = 1.0 / (1.0 - 0.82 * x)
corr = 0.05 * r.normal(size=x.size) * (1 + x)
stable_margin = np.min(np.abs(lead)) - np.max(np.abs(corr))

margins = []
for _ in range(500):
    noise = 0.05 * r.normal(size=x.size) * (1 + x)
    margins.append(np.min(np.abs(lead)) - np.max(np.abs(noise)))
margins = np.array(margins)

mean_margin = float(margins.mean())
ci = tuple(np.quantile(margins, [0.025, 0.975]))
tt = stats.ttest_1samp(margins, 0.0)

out = {
    "stable_margin": float(stable_margin),
    "bootstrap_ci": ci,
    "margin_mean": mean_margin,
    "t_stat": float(tt.statistic),
    "p_value": float(tt.pvalue),
    "stability_flag": bool(mean_margin > 0),
}
out.update(end_bench(t0))
print(out)
```

---

## 04 — Yang–Mills: Instanton Trans-Series Decomposition

```python
# file: 04_yang_mills_instanton_transseries.py
import time, tracemalloc
import numpy as np
from scipy import stats

def bench():
    tracemalloc.start()
    return time.perf_counter()

def end_bench(t0):
    cur, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    return {"runtime_ms": (time.perf_counter() - t0) * 1e3, "peak_mb": peak / 1024 / 1024}

t0 = bench()
r = np.random.default_rng(4)

K, N = 10, 300
k = np.arange(1, K + 1)
amps = []
for _ in range(N):
    a = np.exp(-0.85 * k) * np.exp(r.normal(0, 0.12, K))
    amps.append(a)
amps = np.vstack(amps)

log_mean = np.log(amps.mean(axis=0))
fit = stats.linregress(k, log_mean)
resid = log_mean - (fit.intercept + fit.slope * k)
rmse = float(np.sqrt(np.mean(resid**2)))

out = {
    "decay_slope": float(fit.slope),
    "decay_r2": float(fit.rvalue**2),
    "residual_rmse": rmse,
    "sector_energy_mean": float(np.mean(amps)),
    "sector_variance": float(np.var(amps)),
}
out.update(end_bench(t0))
print(out)
```

---

## 05 — Goldbach: Anti-Symmetry and Winding Parity

```python
# file: 05_goldbach_antisymmetry.py
import time, tracemalloc
import numpy as np
from scipy import stats

def bench():
    tracemalloc.start()
    return time.perf_counter()

def end_bench(t0):
    cur, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    return {"runtime_ms": (time.perf_counter() - t0) * 1e3, "peak_mb": peak / 1024 / 1024}

t0 = bench()
r = np.random.default_rng(5)
n = 512
theta = np.linspace(0, 2*np.pi, n, endpoint=False)

z = np.exp(1j * theta) + 0.03 * (r.normal(size=n) + 1j * r.normal(size=n))
resid = np.abs(z[:n//2] + z[n//2:])

winding = int(np.round(np.unwrap(np.angle(z)).ptp() / (2*np.pi)))
stat, p = stats.wilcoxon(resid)

out = {
    "antisym_residual_max": float(resid.max()),
    "antisym_residual_mean": float(resid.mean()),
    "wilcoxon_stat": float(stat),
    "wilcoxon_p": float(p),
    "winding_number": winding,
    "winding_parity_even": bool(winding % 2 == 0),
}
out.update(end_bench(t0))
print(out)
```

---

## 06 — Goldbach: Persistent Homology Duality

```python
# file: 06_goldbach_tda_duality.py
import time, tracemalloc
import numpy as np
from scipy import stats
from sklearn.metrics import roc_auc_score

def bench():
    tracemalloc.start()
    return time.perf_counter()

def end_bench(t0):
    cur, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    return {"runtime_ms": (time.perf_counter() - t0) * 1e3, "peak_mb": peak / 1024 / 1024}

def pers_score(x):
    x = np.asarray(x)
    return float(np.sum(np.maximum(x - np.median(x), 0.0)))

t0 = bench()
r = np.random.default_rng(6)
n = 300

pos_h0 = r.normal(0.18, 0.05, n)
freq_h1 = r.normal(0.22, 0.05, n)
pos_h1 = r.normal(0.82, 0.05, n)
freq_h0 = r.normal(0.78, 0.05, n)

dual0 = pos_h0 + freq_h0
dual1 = pos_h1 + freq_h1

obs = dual1.mean() - dual0.mean()
p = stats.permutation_test((dual1, dual0), lambda a, b: np.mean(a) - np.mean(b),
                            permutation_type="independent", alternative="two-sided",
                            n_resamples=2500, random_state=6).pvalue
auc = roc_auc_score(np.r_[np.zeros(n), np.ones(n)], np.r_[dual0, dual1])

out = {
    "joint_obstruction_gap": float(obs),
    "perm_p": float(p),
    "roc_auc": float(auc),
    "dual_persistence_score_h0": float(dual0.mean()),
    "dual_persistence_score_h1": float(dual1.mean()),
}
out.update(end_bench(t0))
print(out)
```

---

## 07 — Goldbach: Dirichlet-Square Natural Boundary

```python
# file: 07_goldbach_natural_boundary.py
import time, tracemalloc
import numpy as np
from scipy import stats

def bench():
    tracemalloc.start()
    return time.perf_counter()

def end_bench(t0):
    cur, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    return {"runtime_ms": (time.perf_counter() - t0) * 1e3, "peak_mb": peak / 1024 / 1024}

t0 = bench()
r = np.random.default_rng(7)

x = np.linspace(0.05, 0.98, 300)
y = 1.0 / (1.0 - x) + 0.02 * r.normal(size=x.size)

split = 220
coef = np.polyfit(x[:split], y[:split], deg=3)
pred = np.polyval(coef, x[split:])
rmse = float(np.sqrt(np.mean((y[split:] - pred)**2)))
r2 = float(1.0 - np.sum((y[split:] - pred)**2) / np.sum((y[split:] - y[split:].mean())**2))

obs_tail = float(np.mean(np.abs(y[split:])))
p = stats.ttest_1samp(y[split:] - pred, 0.0).pvalue

out = {
    "holdout_rmse": rmse,
    "holdout_r2": r2,
    "tail_magnitude": obs_tail,
    "continuation_p": float(p),
    "boundary_obstruction_flag": bool(r2 < 0.8),
}
out.update(end_bench(t0))
print(out)
```

---

## 08 — Goldbach: Prime Zeta Basin Conditions

```python
# file: 08_goldbach_basin_conditions.py
import time, tracemalloc
import numpy as np
from scipy import stats
from sklearn.metrics import roc_auc_score

def bench():
    tracemalloc.start()
    return time.perf_counter()

def end_bench(t0):
    cur, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    return {"runtime_ms": (time.perf_counter() - t0) * 1e3, "peak_mb": peak / 1024 / 1024}

t0 = bench()
r = np.random.default_rng(8)

N, T = 300, 40
c = r.uniform(-1.25, 0.2, N)
x0 = r.uniform(-2, 2, N)

bounded = np.zeros(N, dtype=int)
score = np.zeros(N)

for i in range(N):
    x = x0[i]
    for _ in range(T):
        x = x*x + c[i]
        if abs(x) > 20:
            break
    bounded[i] = int(abs(x) <= 20)
    score[i] = 1.0 / (1.0 + abs(2*x0[i]))

auc = roc_auc_score(bounded, score)
tt = stats.ttest_ind(score[bounded == 1], score[bounded == 0], equal_var=False)

out = {
    "bounded_rate": float(bounded.mean()),
    "score_auc": float(auc),
    "score_t": float(tt.statistic),
    "score_p": float(tt.pvalue),
    "mean_c": float(c.mean()),
}
out.update(end_bench(t0))
print(out)
```

---

## 09 — Protein: Reduced-Order Diagnostics

```python
# file: 09_protein_reduced_order_diagnostics.py
import time, tracemalloc
import numpy as np
from scipy import stats
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import roc_auc_score, average_precision_score

def bench():
    tracemalloc.start()
    return time.perf_counter()

def end_bench(t0):
    cur, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    return {"runtime_ms": (time.perf_counter() - t0) * 1e3, "peak_mb": peak / 1024 / 1024}

def change_point(x, window=20, thresh=3.0):
    x = np.asarray(x)
    vars_ = np.array([np.var(x[i:i+window]) for i in range(len(x)-window)])
    z = np.abs(stats.zscore(vars_))
    idx = np.argmax(z > thresh)
    return int(idx) if np.any(z > thresh) else -1

t0 = bench()
r = np.random.default_rng(9)
n, T = 300, 160
t = np.linspace(0, 10, T)

rg = np.exp(-0.35*t)[None, :] + 0.03 * r.normal(size=(n, T))
tp = 2.2 * rg + 0.05 * r.normal(size=(n, T))

y = (rg.mean(axis=1) < np.median(rg.mean(axis=1))).astype(int)
X = np.c_[rg.mean(axis=1), tp.mean(axis=1), tp.std(axis=1)]

clf = make_pipeline(StandardScaler(), LogisticRegression(max_iter=1000))
clf.fit(X, y)
scores = clf.predict_proba(X)[:, 1]

auc = roc_auc_score(y, scores)
ap = average_precision_score(y, scores)
ons = np.array([change_point(tp[i]) for i in range(n)])

out = {
    "roc_auc": float(auc),
    "pr_auc": float(ap),
    "mean_onset_index": float(np.mean(ons[ons >= 0])) if np.any(ons >= 0) else -1,
    "folded_rate": float(y.mean()),
    "mean_rg_tp_corr": float(np.mean([stats.pearsonr(rg[i], tp[i])[0] for i in range(n)])),
}
out.update(end_bench(t0))
print(out)
```

---

## 10 — Protein: Gauge-Persistence Transform Invariance

```python
# file: 10_protein_gauge_persistence_invariance.py
import time, tracemalloc
import numpy as np
from scipy import stats

def bench():
    tracemalloc.start()
    return time.perf_counter()

def end_bench(t0):
    cur, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    return {"runtime_ms": (time.perf_counter() - t0) * 1e3, "peak_mb": peak / 1024 / 1024}

def pers_score(x):
    x = np.asarray(x)
    return float(np.sum(np.maximum(x - np.median(x), 0.0)))

t0 = bench()
r = np.random.default_rng(10)
n, T = 240, 140
base = np.exp(-0.25 * np.linspace(0, 8, T))

scores = []
for _ in range(n):
    a = base + 0.02 * r.normal(size=T)
    b = 1.07 * a + 0.03 * r.normal(size=T)
    c = 0.95 * b + 0.03 * r.normal(size=T)
    scores.append([pers_score(a), pers_score(b), pers_score(c), np.corrcoef(a, c)[0, 1]])
scores = np.array(scores)

ratio12 = scores[:, 1] / (scores[:, 0] + 1e-12)
ratio23 = scores[:, 2] / (scores[:, 1] + 1e-12)

out = {
    "mean_corr": float(scores[:, 3].mean()),
    "corr_ci": tuple(np.quantile(scores[:, 3], [0.025, 0.975])),
    "ratio12_mean": float(ratio12.mean()),
    "ratio23_mean": float(ratio23.mean()),
    "invariance_flag": bool(np.std([ratio12.mean(), ratio23.mean()]) < 0.1),
}
out.update(end_bench(t0))
print(out)
```

---

## 11 — Protein: Local Gauge-Persistence Transform

```python
# file: 11_protein_local_lgpt.py
import time, tracemalloc
import numpy as np
from scipy import stats

def bench():
    tracemalloc.start()
    return time.perf_counter()

def end_bench(t0):
    cur, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    return {"runtime_ms": (time.perf_counter() - t0) * 1e3, "peak_mb": peak / 1024 / 1024}

t0 = bench()
r = np.random.default_rng(11)

n_slices, n = 6, 80
groups = []
slice_means = []

for s in range(n_slices):
    vals = 1.0 + 0.03 * s + 0.05 * r.normal(size=n)
    groups.append(vals)
    slice_means.append(vals.mean())

F, p = stats.f_oneway(*groups)

allv = np.concatenate(groups)
grand = allv.mean()
ssb = sum(len(g) * (g.mean() - grand) ** 2 for g in groups)
ssw = sum(((g - g.mean()) ** 2).sum() for g in groups)
icc = ssb / (ssb + ssw + 1e-12)

pairwise = []
for i in range(n_slices - 1):
    pairwise.append(stats.wasserstein_distance(groups[i], groups[i+1]))

out = {
    "anova_F": float(F),
    "anova_p": float(p),
    "icc": float(icc),
    "slice_mean_sd": float(np.std(slice_means, ddof=1)),
    "mean_wasserstein_neighbor": float(np.mean(pairwise)),
}
out.update(end_bench(t0))
print(out)
```

---

## 12 — Protein: Statistical Analysis of LGPT Trajectories

```python
# file: 12_protein_lgpt_trajectory_stats.py
import time, tracemalloc
import numpy as np
from scipy import stats

def bench():
    tracemalloc.start()
    return time.perf_counter()

def end_bench(t0):
    cur, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    return {"runtime_ms": (time.perf_counter() - t0) * 1e3, "peak_mb": peak / 1024 / 1024}

def change_point(x, window=20, thresh=3.0):
    x = np.asarray(x)
    vars_ = np.array([np.var(x[i:i+window]) for i in range(len(x)-window)])
    z = np.abs(stats.zscore(vars_))
    idx = np.argmax(z > thresh)
    return int(idx) if np.any(z > thresh) else -1

t0 = bench()
r = np.random.default_rng(12)
n, T = 250, 180
t = np.linspace(0, 10, T)

rg = np.exp(-0.3 * t)[None, :] + 0.03 * r.normal(size=(n, T))
tp = 1.8 * rg + 0.06 * r.normal(size=(n, T))

corrs = np.array([stats.pearsonr(rg[i], tp[i])[0] for i in range(n)])
ons = np.array([change_point(tp[i]) for i in range(n)])

pre = tp[:, :T//2].mean(axis=1)
post = tp[:, T//2:].mean(axis=1)
tt = stats.ttest_rel(pre, post)

out = {
    "mean_corr": float(np.nanmean(corrs)),
    "corr_ci": tuple(np.nanquantile(corrs, [0.025, 0.975])),
    "onset_rate": float((ons >= 0).mean()),
    "paired_t": float(tt.statistic),
    "paired_p": float(tt.pvalue),
}
out.update(end_bench(t0))
print(out)
```

---

## 13 — Protein: Free Energy and SO(3) Symmetry Regression

```python
# file: 13_protein_free_energy_so3.py
import time, tracemalloc
import numpy as np
from scipy import stats
from sklearn.linear_model import LinearRegression
from sklearn.metrics import r2_score

def bench():
    tracemalloc.start()
    return time.perf_counter()

def end_bench(t0):
    cur, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    return {"runtime_ms": (time.perf_counter() - t0) * 1e3, "peak_mb": peak / 1024 / 1024}

t0 = bench()
r = np.random.default_rng(13)
n, T = 260, 120

F = np.linspace(2.0, -1.0, T)[None, :] + 0.05 * r.normal(size=(n, T))
rot = r.normal(size=(n, T))
tp = -0.7 * F + 0.2 * rot + 0.08 * r.normal(size=(n, T))

X = np.c_[F.mean(axis=1), rot.mean(axis=1)]
y = tp.mean(axis=1)

lr = LinearRegression().fit(X, y)
pred = lr.predict(X)

out = {
    "r2": float(r2_score(y, pred)),
    "coef_free_energy": float(lr.coef_[0]),
    "coef_so3": float(lr.coef_[1]),
    "corr_F_tp": float(stats.pearsonr(F.mean(axis=1), y)[0]),
}
out.update(end_bench(t0))
print(out)
```

---

## 14 — Applied: Arithmetic Matched Filtering

```python
# file: 14_arithmetic_matched_filtering.py
import time, tracemalloc
import numpy as np
from scipy import stats
from sklearn.metrics import roc_auc_score

def bench():
    tracemalloc.start()
    return time.perf_counter()

def end_bench(t0):
    cur, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    return {"runtime_ms": (time.perf_counter() - t0) * 1e3, "peak_mb": peak / 1024 / 1024}

t0 = bench()
r = np.random.default_rng(14)
N, T = 500, 128
template = np.sin(np.linspace(0, 2*np.pi, T))

scores, labels = [], []
for _ in range(N):
    x0 = 0.8 * r.normal(size=T)
    x1 = x0 + 0.7 * template + 0.2 * r.normal(size=T)
    scores.extend([np.dot(x0, template), np.dot(x1, template)])
    labels.extend([0, 1])

scores = np.array(scores)
labels = np.array(labels)
thr = np.median(scores)
pred = (scores > thr).astype(int)

tp = np.sum((pred == 1) & (labels == 1))
fp = np.sum((pred == 1) & (labels == 0))
tn = np.sum((pred == 0) & (labels == 0))
fn = np.sum((pred == 0) & (labels == 1))

far = fp / (fp + tn + 1e-12)
pd = tp / (tp + fn + 1e-12)
auc = roc_auc_score(labels, scores)
p = stats.ttest_ind(scores[labels == 1], scores[labels == 0], equal_var=False).pvalue

out = {
    "roc_auc": float(auc),
    "false_alarm_rate": float(far),
    "prob_detection": float(pd),
    "threshold": float(thr),
    "class_separation_p": float(p),
}
out.update(end_bench(t0))
print(out)
```

---

## 15 — Applied: Network Contagion Reduction

```python
# file: 15_network_contagion_reduction.py
import time, tracemalloc
import numpy as np
from scipy import stats
from sklearn.linear_model import LinearRegression
from sklearn.metrics import mean_squared_error

def bench():
    tracemalloc.start()
    return time.perf_counter()

def end_bench(t0):
    cur, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    return {"runtime_ms": (time.perf_counter() - t0) * 1e3, "peak_mb": peak / 1024 / 1024}

def pref_attach(n=160, m=3, seed=0):
    r = np.random.default_rng(seed)
    A = np.zeros((n, n), dtype=int)
    deg = np.ones(n)
    for i in range(1, n):
        p = deg[:i] / deg[:i].sum()
        k = min(m, i)
        tgt = r.choice(i, size=k, replace=False, p=p)
        A[i, tgt] = 1
        A[tgt, i] = 1
        deg[i] += k
        deg[tgt] += 1
    return A

t0 = bench()
r = np.random.default_rng(15)
n, T = 160, 40
A = pref_attach(n=n, m=3, seed=15)

x = np.zeros(n, dtype=int)
x[r.integers(0, n)] = 1
prev = []

beta, gamma = 0.18, 0.08
for _ in range(T):
    force = A @ x
    inf_p = 1 - np.exp(-beta * force)
    new_x = x.copy()
    new_x[(x == 0) & (r.random(n) < inf_p)] = 1
    new_x[(x == 1) & (r.random(n) < gamma)] = 0
    x = new_x
    prev.append(x.mean())

prev = np.array(prev)
X = prev[:-1].reshape(-1, 1)
y = prev[1:]

lr = LinearRegression().fit(X, y)
pred = lr.predict(X)

thr_time = int(np.argmax(prev > 0.5)) if np.any(prev > 0.5) else -1
rmse = float(np.sqrt(mean_squared_error(y, pred)))

out = {
    "forecast_rmse": rmse,
    "critical_spread_time": thr_time,
    "slope": float(lr.coef_[0]),
    "final_prevalence": float(prev[-1]),
}
out.update(end_bench(t0))
print(out)
```

---

## 16 — Applied: Candidate Spectral Triple Construction

```python
# file: 16_candidate_spectral_triple.py
import time, tracemalloc
import numpy as np
from scipy import stats
from scipy.special import gammaln
from sklearn.linear_model import LinearRegression
from sklearn.metrics import mean_squared_error, r2_score

def bench():
    tracemalloc.start()
    return time.perf_counter()

def end_bench(t0):
    cur, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    return {"runtime_ms": (time.perf_counter() - t0) * 1e3, "peak_mb": peak / 1024 / 1024}

t0 = bench()
r = np.random.default_rng(16)
q = np.linspace(0.05, 5.0, 240)
y = 0.5 * gammaln(1 + q) + 0.02 * r.normal(size=q.size)

X = np.c_[np.ones_like(q), q, q**2, np.log1p(q)]
lr = LinearRegression(fit_intercept=False).fit(X, y)
pred = lr.predict(X)

out = {
    "rmse": float(np.sqrt(mean_squared_error(y, pred))),
    "r2": float(r2_score(y, pred)),
    "smoothness": float(np.mean(np.abs(np.diff(pred, n=2)))),
    "coef_norm": float(np.linalg.norm(lr.coef_)),
}
out.update(end_bench(t0))
print(out)
```

---

## 17 — Applied: Sub-Nyquist Compressed Sensing

```python
# file: 17_sub_nyquist_compressed_sensing.py
import time, tracemalloc
import numpy as np
from sklearn.linear_model import OrthogonalMatchingPursuit
from sklearn.metrics import mean_squared_error

def bench():
    tracemalloc.start()
    return time.perf_counter()

def end_bench(t0):
    cur, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    return {"runtime_ms": (time.perf_counter() - t0) * 1e3, "peak_mb": peak / 1024 / 1024}

t0 = bench()
r = np.random.default_rng(17)
n_features, n_meas, sparsity = 800, 220, 12

X = r.normal(size=(n_meas, n_features)) / np.sqrt(n_meas)
coef = np.zeros(n_features)
support = r.choice(n_features, size=sparsity, replace=False)
coef[support] = r.normal(size=sparsity)
y = X @ coef + 0.03 * r.normal(size=n_meas)

omp = OrthogonalMatchingPursuit(n_nonzero_coefs=sparsity)
omp.fit(X, y)
pred = omp.coef_

true_sup = set(np.where(coef != 0)[0])
pred_sup = set(np.where(pred != 0)[0])

out = {
    "mse": float(mean_squared_error(coef, pred)),
    "support_recall": float(len(true_sup & pred_sup) / len(true_sup)),
    "nnz_pred": int((pred != 0).sum()),
    "matrix_mb": float(X.nbytes / 1024 / 1024),
}
out.update(end_bench(t0))
print(out)
```

---

## 18 — Quantum Sensing: Unruh–Bogoliubov Thermal Filtering

```python
# file: 18_unruh_bogoliubov_filter.py
import time, tracemalloc
import numpy as np
from statsmodels.stats.diagnostic import acorr_ljungbox

def bench():
    tracemalloc.start()
    return time.perf_counter()

def end_bench(t0):
    cur, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    return {"runtime_ms": (time.perf_counter() - t0) * 1e3, "peak_mb": peak / 1024 / 1024}

def one_over_f_noise(n, alpha=1.0, seed=0):
    r = np.random.default_rng(seed)
    freqs = np.fft.rfftfreq(n)
    freqs[0] = freqs[1]
    amp = 1 / (freqs ** (alpha / 2))
    phase = r.normal(size=len(freqs)) + 1j * r.normal(size=len(freqs))
    x = np.fft.irfft(amp * phase, n=n)
    return (x - x.mean()) / (x.std() + 1e-12)

def ema(y, alpha=0.02):
    y = np.asarray(y)
    out = np.empty_like(y, dtype=float)
    out[0] = y[0]
    for i in range(1, len(y)):
        out[i] = alpha * y[i] + (1 - alpha) * out[i - 1]
    return out

t0 = bench()
r = np.random.default_rng(18)
n = 3000
drift = one_over_f_noise(n, alpha=1.0, seed=18)
signal = np.sin(np.linspace(0, 12, n)) + 0.6 * drift + 0.15 * r.normal(size=n)
filtered = signal - 0.9 * ema(signal, alpha=0.01)

truth = np.sin(np.linspace(0, 12, n))
raw_res = signal - truth
fil_res = filtered - truth

raw_lb = acorr_ljungbox(raw_res, lags=[10], return_df=True)["lb_pvalue"].iloc[0]
fil_lb = acorr_ljungbox(fil_res, lags=[10], return_df=True)["lb_pvalue"].iloc[0]

out = {
    "mae_raw": float(np.mean(np.abs(raw_res))),
    "mae_filtered": float(np.mean(np.abs(fil_res))),
    "ljung_box_raw": float(raw_lb),
    "ljung_box_filtered": float(fil_lb),
}
out.update(end_bench(t0))
print(out)
```

---

## 19 — Quantum Sensing: Rindler–Bogoliubov Equivalence Proxy

```python
# file: 19_rindler_bogoliubov_equivalence.py
import time, tracemalloc
import numpy as np
from scipy import stats

def bench():
    tracemalloc.start()
    return time.perf_counter()

def end_bench(t0):
    cur, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    return {"runtime_ms": (time.perf_counter() - t0) * 1e3, "peak_mb": peak / 1024 / 1024}

t0 = bench()
r = np.random.default_rng(19)
n = 2500

x = np.sin(np.linspace(0, 8, n)) + 0.2 * r.normal(size=n)
y = 1.08 * x + 0.22 * r.normal(size=n)

ks = stats.ks_2samp((x - x.mean()) / x.std(), (y - y.mean()) / y.std())
corr = np.corrcoef(x, y)[0, 1]
mse = float(np.mean((x - y) ** 2))

out = {
    "ks_stat": float(ks.statistic),
    "ks_p": float(ks.pvalue),
    "corr": float(corr),
    "mse": mse,
    "equivalence_flag": bool(corr > 0.85 and ks.pvalue > 0.05),
}
out.update(end_bench(t0))
print(out)
```

---

## 20 — Quantum Sensing: Vacuum State Transitions

```python
# file: 20_vacuum_state_transitions.py
import time, tracemalloc
import numpy as np
from scipy import stats

def bench():
    tracemalloc.start()
    return time.perf_counter()

def end_bench(t0):
    cur, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    return {"runtime_ms": (time.perf_counter() - t0) * 1e3, "peak_mb": peak / 1024 / 1024}

def cusum_detect(x, thresh=7.0):
    x = np.asarray(x)
    z = (x - x.mean()) / (x.std() + 1e-12)
    c = np.cumsum(z)
    idx = int(np.argmax(np.abs(c)))
    return idx if np.abs(c[idx]) > thresh else -1

t0 = bench()
r = np.random.default_rng(20)
n = 2400
cp = int(0.6 * n)

resp = 0.08 * r.normal(size=n)
resp[cp:] += 0.35

det = cusum_detect(resp)
delay = det - cp if det >= 0 else np.nan

tt = stats.ttest_ind(resp[:cp], resp[cp:], equal_var=False)

out = {
    "true_cp": cp,
    "detected_cp": det,
    "delay": float(delay),
    "ttest_p": float(tt.pvalue),
    "shift_mean_diff": float(resp[cp:].mean() - resp[:cp].mean()),
}
out.update(end_bench(t0))
print(out)
```

---

## 21 — Quantum Sensing: Atomic Clock Decoherence / IMU Noise

```python
# file: 21_atomic_clock_decoherence.py
import time, tracemalloc
import numpy as np
from statsmodels.stats.diagnostic import acorr_ljungbox
from sklearn.metrics import mean_squared_error

def bench():
    tracemalloc.start()
    return time.perf_counter()

def end_bench(t0):
    cur, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    return {"runtime_ms": (time.perf_counter() - t0) * 1e3, "peak_mb": peak / 1024 / 1024}

def ema(y, alpha=0.02):
    y = np.asarray(y)
    out = np.empty_like(y, dtype=float)
    out[0] = y[0]
    for i in range(1, len(y)):
        out[i] = alpha * y[i] + (1 - alpha) * out[i - 1]
    return out

def kalman_local_level(y, q=0.01, r=0.05):
    x, p = 0.0, 1.0
    xs, innov = [], []
    for z in y:
        p += q
        k = p / (p + r)
        e = z - x
        x = x + k * e
        p = (1 - k) * p
        xs.append(x)
        innov.append(e)
    return np.array(xs), np.array(innov)

t0 = bench()
r = np.random.default_rng(21)
n = 2200
latent = np.cumsum(0.01 * r.normal(size=n))
y = latent + 0.05 * r.normal(size=n)

kf, innov = kalman_local_level(y, q=0.005, r=0.05)
em = ema(y, alpha=0.02)

rmse_k = np.sqrt(mean_squared_error(latent, kf))
rmse_e = np.sqrt(mean_squared_error(latent, em))
lb = acorr_ljungbox(innov, lags=[10], return_df=True)["lb_pvalue"].iloc[0]
decoh = int(np.argmax(np.abs(innov) > 3 * np.std(innov))) if np.any(np.abs(innov) > 3 * np.std(innov)) else -1

out = {
    "rmse_kalman": float(rmse_k),
    "rmse_ema": float(rmse_e),
    "ljung_box_p": float(lb),
    "decoherence_t": decoh,
    "innovation_std": float(np.std(innov)),
}
out.update(end_bench(t0))
print(out)
```

---

## 22 — Signal/Filtering: O(N) Non-Markovian Noise and Adaptive Thresholding

```python
# file: 22_non_markovian_noise_adaptive_filters.py
import time, tracemalloc
import numpy as np
from statsmodels.stats.diagnostic import acorr_ljungbox
from statsmodels.stats.multitest import multipletests
from sklearn.metrics import mean_squared_error

def bench():
    tracemalloc.start()
    return time.perf_counter()

def end_bench(t0):
    cur, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    return {"runtime_ms": (time.perf_counter() - t0) * 1e3, "peak_mb": peak / 1024 / 1024}

def one_over_f_noise(n, alpha=0.8, seed=0):
    r = np.random.default_rng(seed)
    freqs = np.fft.rfftfreq(n)
    freqs[0] = freqs[1]
    amp = 1 / (freqs ** (alpha / 2))
    phase = r.normal(size=len(freqs)) + 1j * r.normal(size=len(freqs))
    x = np.fft.irfft(amp * phase, n=n)
    return (x - x.mean()) / (x.std() + 1e-12)

def ema(y, alpha=0.05):
    y = np.asarray(y)
    out = np.empty_like(y, dtype=float)
    out[0] = y[0]
    for i in range(1, len(y)):
        out[i] = alpha * y[i] + (1 - alpha) * out[i - 1]
    return out

def sma(y, w=31):
    k = np.ones(w) / w
    return np.convolve(y, k, mode="same")

def kalman_local_level(y, q=0.01, r=0.15):
    x, p = 0.0, 1.0
    xs, innov = [], []
    for z in y:
        p += q
        k = p / (p + r)
        e = z - x
        x = x + k * e
        p = (1 - k) * p
        xs.append(x)
        innov.append(e)
    return np.array(xs), np.array(innov)

t0 = bench()
r = np.random.default_rng(22)
n = 3000
trend = np.sin(np.linspace(0, 10, n))
noise = 0.5 * one_over_f_noise(n, alpha=0.8, seed=22) + 0.1 * r.normal(size=n)
y = trend + noise

filters = {
    "sma": sma(y, 31),
    "ema": ema(y, 0.05),
    "kalman": kalman_local_level(y, q=0.01, r=0.15)[0],
    "hp": y - sma(y, 61),
}

metrics = []
pvals = []
names = []

for name, f in filters.items():
    res = y - f
    rmse = np.sqrt(mean_squared_error(trend, f))
    lbp = acorr_ljungbox(res, lags=[10], return_df=True)["lb_pvalue"].iloc[0]
    metrics.append((name, float(rmse), float(lbp)))
    pvals.append(lbp)
    names.append(name)

rej, qvals = multipletests(pvals, alpha=0.05, method="fdr_bh")[:2]
best = min(metrics, key=lambda z: z[1])

out = {
    "metrics": metrics,
    "best_rmse_filter": best[0],
    "best_rmse": best[1],
    "fdr_reject": dict(zip(names, rej.tolist())),
    "qvals": dict(zip(names, qvals.tolist())),
}
out.update(end_bench(t0))
print(out)
```

---
