# Complete Analysis Example

Demonstrates applying scientific coding principles to a Monte Carlo estimation of pi.

## The Problem

Estimate pi by sampling random points in a unit square and counting how many fall inside the inscribed quarter circle.

## Wrong Approach

```python
import numpy as np

def estimate_pi(n=100000):
    x = np.random.rand(n)
    y = np.random.rand(n)
    inside = np.sum(x**2 + y**2 <= 1.0)  # float comparison with <=
    return 4 * inside / n

print(f"Pi is approximately {estimate_pi():.6f}")
```

Problems:
- Default sample size hides a critical parameter
- Global random state, irreproducible
- No uncertainty estimate
- No convergence check
- No validation against known answer
- Single run, no information about variance

## Right Approach

```python
import numpy as np

def estimate_pi(n_samples, rng):
    """Estimate pi via Monte Carlo sampling of the unit quarter circle.

    Args:
        n_samples: number of random points to sample
        rng: numpy Generator for reproducibility

    Returns:
        dict with estimate, standard error, and 95% confidence interval
    """
    x = rng.uniform(0, 1, size=n_samples)
    y = rng.uniform(0, 1, size=n_samples)

    inside = (x**2 + y**2) < 1.0  # strict < for open disk
    fraction_inside = np.mean(inside)

    pi_estimate = 4.0 * fraction_inside

    # Standard error from binomial proportion
    se = 4.0 * np.sqrt(fraction_inside * (1 - fraction_inside) / n_samples)

    return {
        "estimate": pi_estimate,
        "standard_error": se,
        "ci_95": (pi_estimate - 1.96 * se, pi_estimate + 1.96 * se),
        "n_samples": n_samples,
    }


def validate_convergence(sample_sizes, rng):
    """Verify that the estimator converges as expected.

    For Monte Carlo, standard error should scale as 1/sqrt(n).
    """
    results = []
    for n in sample_sizes:
        result = estimate_pi(n, rng)
        results.append(result)

    # Check that SE decreases roughly as 1/sqrt(n)
    for i in range(1, len(results)):
        ratio_n = sample_sizes[i] / sample_sizes[i - 1]
        ratio_se = results[i - 1]["standard_error"] / results[i]["standard_error"]
        expected_ratio = np.sqrt(ratio_n)
        assert abs(ratio_se - expected_ratio) / expected_ratio < 0.3, (
            f"Convergence rate off: SE ratio {ratio_se:.2f}, "
            f"expected ~{expected_ratio:.2f} for n ratio {ratio_n}"
        )

    return results


if __name__ == "__main__":
    rng = np.random.default_rng(seed=2024)

    # Validate convergence behavior
    sample_sizes = [1_000, 10_000, 100_000, 1_000_000]
    results = validate_convergence(sample_sizes, rng)

    for r in results:
        error = abs(r["estimate"] - np.pi)
        within_ci = r["ci_95"][0] <= np.pi <= r["ci_95"][1]
        print(
            f"n={r['n_samples']:>10,}: "
            f"pi={r['estimate']:.6f} +/- {r['standard_error']:.6f}  "
            f"error={error:.6f}  "
            f"pi in 95% CI: {within_ci}"
        )
```

## Why It Matters

The wrong version produces a single number with no context. You cannot tell if the estimate is reliable, how it depends on sample size, or whether it is consistent with theory. The right version:

- Forces explicit sample size and RNG
- Provides uncertainty quantification (standard error and confidence interval)
- Validates convergence rate against theory (SE ~ 1/sqrt(n))
- Checks whether the known answer falls within the confidence interval
- Produces enough information to judge the quality of the result
