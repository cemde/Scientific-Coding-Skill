# Common Mistakes in Scientific Code

Side-by-side examples of mistakes LLMs frequently make when writing scientific code, and how to fix them.

## 1. Silent Error Recovery

### Wrong

```python
def compute_free_energy(energies, temperature):
    try:
        beta = 1.0 / (KB * temperature)
        return -np.log(np.sum(np.exp(-beta * energies))) / beta
    except (OverflowError, FloatingPointError):
        return np.nan  # silently produces NaN that propagates everywhere
```

### Right

```python
def compute_free_energy(energies, temperature):
    if temperature <= 0:
        raise ValueError(f"Temperature must be positive, got {temperature}")
    beta = 1.0 / (KB * temperature)
    # Use logsumexp for numerical stability instead of catching overflow
    return -logsumexp(-beta * energies) / beta
```

Why: The wrong version hides overflow from the scientist. The right version avoids overflow entirely through a numerically stable algorithm, and crashes on unphysical input.

---

## 2. Default Scientific Parameters

### Wrong

```python
def boltzmann_distribution(energies, temperature=298.15, kb=8.617e-5):
    """Calculate Boltzmann weights. Defaults to room temperature, eV units."""
    beta = 1.0 / (kb * temperature)
    weights = np.exp(-beta * energies)
    return weights / weights.sum()
```

### Right

```python
from scipy.constants import Boltzmann

def boltzmann_distribution(energies_joules, temperature_kelvin):
    """Calculate Boltzmann weights.

    Args:
        energies_joules: Energy levels in Joules.
        temperature_kelvin: Temperature in Kelvin. Must be positive.
    """
    assert temperature_kelvin > 0, f"Temperature must be positive, got {temperature_kelvin}"
    beta = 1.0 / (Boltzmann * temperature_kelvin)
    log_weights = -beta * energies_joules
    log_weights -= logsumexp(log_weights)  # numerically stable normalization
    return np.exp(log_weights)
```

Why: The wrong version has a hard-coded Boltzmann constant that fixes your unit system, and a default temperature that someone will forget to change. The right version forces explicit units and uses scipy's maintained constants.

---

## 3. Random Seeds and Reproducibility

### Wrong

```python
def bootstrap_mean_ci(data, n_bootstrap=10000):
    means = []
    for _ in range(n_bootstrap):
        sample = np.random.choice(data, size=len(data), replace=True)
        means.append(np.mean(sample))
    return np.percentile(means, [2.5, 97.5])
```

### Right

```python
def bootstrap_mean_ci(data, n_bootstrap, rng):
    """Bootstrap confidence interval for the mean.

    Args:
        data: observed samples
        n_bootstrap: number of bootstrap resamples
        rng: numpy Generator instance (e.g., np.random.default_rng(seed=42))
    """
    means = np.empty(n_bootstrap)
    for i in range(n_bootstrap):
        sample = rng.choice(data, size=len(data), replace=True)
        means[i] = np.mean(sample)
    return np.percentile(means, [2.5, 97.5])
```

Why: The wrong version uses global random state, making results irreproducible. It also silently defaults to 10000 bootstraps. The right version requires an explicit RNG and sample count.

---

## 4. Implicit Broadcasting

### Wrong

```python
# velocities shape: (n_particles, 3)
# masses shape: (n_particles,)
kinetic_energy = 0.5 * masses * velocities**2  # broadcasting "works" but is wrong
total_ke = np.sum(kinetic_energy)
```

### Right

```python
# velocities shape: (n_particles, 3)
# masses shape: (n_particles,)
assert velocities.ndim == 2 and velocities.shape[1] == 3
assert masses.shape == (velocities.shape[0],)
speed_squared = np.sum(velocities**2, axis=1)  # (n_particles,)
kinetic_energy = 0.5 * masses * speed_squared   # (n_particles,)
total_ke = np.sum(kinetic_energy)
```

Why: The wrong version broadcasts masses (n,) against velocities**2 (n,3), producing KE per component instead of per particle. It silently gives an answer that is too large by a factor related to the velocity distribution. The right version makes shapes explicit and computes |v|^2 first.

---

## 5. Float Comparison

### Wrong

```python
def find_degenerate_states(energies):
    """Find groups of states with equal energy."""
    groups = []
    for i, e in enumerate(energies):
        found = False
        for g in groups:
            if energies[g[0]] == e:  # float equality
                g.append(i)
                found = True
                break
        if not found:
            groups.append([i])
    return groups
```

### Right

```python
def find_degenerate_states(energies, atol):
    """Find groups of states with energy within atol of each other.

    Args:
        energies: energy levels (same units as atol)
        atol: absolute tolerance defining degeneracy (in same energy units)
    """
    groups = []
    for i, e in enumerate(energies):
        found = False
        for g in groups:
            if abs(energies[g[0]] - e) < atol:
                g.append(i)
                found = True
                break
        if not found:
            groups.append([i])
    return groups
```

Why: Float equality comparison will miss degenerate states due to rounding. The tolerance must be explicit and problem-specific, not a hidden default.

---

## 6. Unvalidated Statistical Tests

### Wrong

```python
from scipy.stats import ttest_ind

def compare_groups(group_a, group_b):
    stat, pvalue = ttest_ind(group_a, group_b)
    return pvalue < 0.05  # just returns True/False
```

### Right

```python
from scipy.stats import ttest_ind, shapiro, levene

def compare_groups(group_a, group_b, alpha):
    """Two-sample comparison with assumption checking.

    Returns dict with test statistic, p-value, effect size, and assumption checks.
    Raises ValueError if assumptions are severely violated.
    """
    # Check normality assumption
    _, p_normal_a = shapiro(group_a)
    _, p_normal_b = shapiro(group_b)
    if p_normal_a < 0.01 or p_normal_b < 0.01:
        raise ValueError(
            f"Normality assumption violated (Shapiro p={p_normal_a:.4f}, {p_normal_b:.4f}). "
            "Consider a non-parametric test."
        )

    # Check equal variance assumption
    _, p_levene = levene(group_a, group_b)
    equal_var = p_levene > 0.05

    stat, pvalue = ttest_ind(group_a, group_b, equal_var=equal_var)

    # Effect size (Cohen's d)
    pooled_std = np.sqrt((np.var(group_a, ddof=1) + np.var(group_b, ddof=1)) / 2)
    cohens_d = (np.mean(group_a) - np.mean(group_b)) / pooled_std

    return {
        "statistic": stat,
        "pvalue": pvalue,
        "significant": pvalue < alpha,
        "cohens_d": cohens_d,
        "n_a": len(group_a),
        "n_b": len(group_b),
        "equal_var_assumed": equal_var,
    }
```

Why: The wrong version ignores test assumptions, discards the test statistic, reports no effect size, hard-codes the significance level, and reduces the result to a boolean. The right version checks assumptions, crashes if they are violated, and returns enough information to evaluate the result.
