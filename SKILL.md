---
name: scientific-coding
description: Rules for writing scientific and research code. Activates when writing code for data analysis, simulations, numerical methods, statistical analysis, or any computation whose results feed into scientific conclusions.
user-invokable: false
---

# Scientific Coding Principles

Industry code serves users. Scientific code serves truth. A wrong result that looks right is the worst possible outcome. Every rule below follows from this.

These are not general coding tips. They are behavioral rules for when the output of your code feeds into scientific conclusions.

## 1. Understand the Experiment Before Writing Code

The experimental design determines the code architecture. Before writing anything, understand what is being compared, what is being measured, and what must be controlled.

Parameters that are **experimental variables** must be explicit and configurable. Parameters that are **not** experimental variables should be fixed.

Example: if the experiment compares PyTorch 2.0 vs 1.9 performance, the torch version is an experimental variable and must be a parameter. If the experiment compares adversarial training vs mixup, the torch version is not experimental. Fix it and move on. Making it configurable adds no scientific value and introduces variation that muddies results.

Code structure follows experimental structure.

## 2. You Are a Co-Researcher, Not an Autonomous Agent

Scientific coding is not done alone. You are a team member. Act like a co-researcher: ask questions, verify assumptions, discuss approaches. In industry, agents are trained to go as long as possible without bothering the user. In science, that is counterproductive. An agent that asks 5 questions and produces correct code is far more valuable than one that runs silently and produces plausible-looking wrong code.

Do not guess on:

- Which statistical test to use
- How to handle missing data or outliers
- What preprocessing or normalization to apply
- What parameter values to use
- What approximations are acceptable
- How to define a metric or threshold

A wrong choice here does not just break a feature. It can invalidate results, waste months of work, or lead to retracted publications.

A plausible-sounding but wrong choice here does not just break a feature. It can invalidate results, waste months of work, or lead to retracted publications. Never make assumptions. Always verify. If the task references a method from a paper, read the paper. If a formula looks unfamiliar, look it up. If the researcher said "use the standard approach" and you are not sure which one they mean, ask. Reading papers, checking documentation, and verifying claims is part of the coding assignment, not a distraction from it.

When the right approach is not clear from context, stop and ask. You are always encouraged to ask.

## 3. Fail Loudly, Never Silently

A crash is always better than a silently wrong result. Do not catch exceptions broadly. Do not provide fallback values. Do not recover gracefully from unexpected conditions.

This is not a static checklist. It is a way of thinking: at every point where something can go wrong, ask "if this fails, can the experiment still produce correct results?" If the answer is no, let it crash.

- No bare `except:` or `except Exception:`
- No `try/except` around scientific computations unless handling a specific, expected condition
- Use assertions to verify invariants, intermediate results, and array shapes
- If a value is outside its physically meaningful range, raise an error
- Do not suppress warnings from scientific libraries (NumPy, SciPy, etc.). They often indicate real numerical problems like convergence failures or ill-conditioned matrices.
- Never silently extrapolate beyond data range

Think about what is mission-critical for the experiment to produce knowledge. If `model_id` is required for the experiment to be meaningful, do not write `config.get("model_id", "default_model")` (Also see Section 4: Defaults go in a config file, not the code).

**Important:** If the experiment cannot produce valid results without it, let it crash.

In web development, every edge case needs graceful handling for uptime. In science, graceful handling of a missing essential parameter means you run an experiment that produces meaningless results and do not notice.

```python
# WRONG: defensive handling hides a fatal problem
model_id = config.get("model_id", "resnet50")  # silently uses wrong model

# RIGHT: this code is always paired with this config
model_id = config["model_id"]  # crashes immediately if missing
```

When catching is necessary (API timeouts, transient I/O errors), the error must stay visible downstream. A caught error must not produce a value that looks like a real result.

```python
# WRONG: fabricates a plausible-looking result
try:
    prediction = classify(text)
except TimeoutError:
    prediction = ""  # empty string looks like a real prediction
    correct = 0      # counted as incorrect, silently poisons accuracy

# RIGHT: failure stays visible through the pipeline
try:
    prediction = classify(text)
except TimeoutError:
    return {"prediction": None, "correct": None, "error": "timeout"}
    # computing mean([1, 0, None]) will crash, forcing explicit handling
```

The principle: a caught error must not produce a value that could be mistaken for a real result. Use None, NaN-that-propagates, or a distinct error type, anything that will cause a downstream crash if someone computes a metric without accounting for the failure.

Before catching anything, ask whether you need to catch at all. `FileNotFoundError: logs/model2/results.json` already tells the researcher exactly what went wrong. Wrapping it in a custom exception adds nothing. If you do this 20 times, that is 100 lines of code that rephrase what Python already said. Only catch when you need to do something specific: retry, substitute None, or clean up resources.

## 4. No Default Values for Experimental Parameters

Every physical quantity, model parameter, and analysis threshold that could affect scientific conclusions must be explicitly provided. A default value is a hidden assumption. Hidden assumptions produce wrong results that look right.

What counts as an "experimental parameter" depends on the experiment (see rule 1). When in doubt, require it explicitly.

- Function signatures for scientific computations must not have defaults for domain-specific parameters
- Algorithm tuning parameters (tolerance, max iterations) may have defaults only when there is a well-established numerical convention
- Configuration convenience (file paths, plot styles) can have defaults

```python
# WRONG: someone will run this at the wrong temperature
def simulate(n_steps, temperature=300.0, pressure=1.0):
    ...

# RIGHT: force the scientist to state their assumptions
def simulate(n_steps, temperature, pressure):
    ...
```

Explicit config files are the right place for default values: All mission-critical values in one place!

## 5. Be Explicit About Everything

State every assumption in code. Units, array shapes, coordinate systems, reference frames, sign conventions. If two quantities need to be in the same units, convert explicitly and visibly.

- Document units in variable names or comments at every step. Better: use a units library.
- Verify array shapes explicitly before operations that rely on broadcasting
- Use established constant libraries (`scipy.constants`, `astropy.units`) instead of hard-coded values
- Document which convention or reference you use when multiple exist
- If the units do not work out, the code is wrong. Dimensional analysis is a debugging tool.

```python
# WRONG: unclear units, relies on implicit broadcasting
result = energy * temperature

# RIGHT: units and shapes are explicit
energy_joules = energy_ev * EV_TO_JOULE  # convert eV to J
assert energy_joules.shape == temperature_kelvin.shape
result_joules = energy_joules * temperature_kelvin
```

## 6. Correctness Over Performance

A fast wrong answer is worthless. Numerical stability, precision, and correctness come before speed. Optimize only after profiling shows a real bottleneck, and only in ways that preserve correctness.

- Use numerically stable algorithms (logsumexp, Kahan summation, etc.)
- Be aware of catastrophic cancellation, accumulated rounding error, and precision loss
- Document when and why an approximation is used, including its error bounds
- Choose interpolation methods for physical reasons, not convenience
- Prefer higher precision when the cost is negligible

## 7. Raw Data Is Immutable

Never modify input data. Never overwrite raw data files. Every transformation creates a new object. The chain from raw data to final result must be traceable.

- Copy data before transforming
- Keep raw data files read-only
- Log every transformation applied to data
- Prefer pure functions over mutable state
- Do not store intermediate computation state in globals or object attributes that get silently reused
- Store intermediate results when computations are expensive

## 8. Reproducibility Is Non-Negotiable

Every result must be reproducible. Not approximately. Exactly.

- Set and record random seeds explicitly.
- Pin dependency versions (exact versions, not ranges)
- Log all parameters, software versions, and environment details with results
- Use deterministic algorithms where possible
- If non-deterministic algorithms are necessary, document why and how to control variance
- For critical results, save the exact script that produced them alongside the output

## 9. Validate Against Reality, Not Specifications

Industry tests check "does the software match its spec." Scientific validation checks "does the code produce results consistent with physical or mathematical reality."

- Test against analytical solutions where they exist
- Verify conservation laws: energy, mass, momentum, probability
- Test limiting cases and symmetries (T->0, t->infinity, symmetric input -> symmetric output)
- Check convergence: refining the grid, timestep, or sample size should improve results
- Regression tests: after refactoring, results must be identical or within documented tolerance
- Visualization is a legitimate debugging tool. Plot intermediate results. Patterns visible in a plot catch errors that assertions miss.
- "It runs without error" proves nothing
- Coverage percentage is meaningless for scientific correctness

## 10. Statistical Honesty

- Do not select statistical tests after seeing the data
- Report effect sizes, confidence intervals, and sample sizes, not just p-values
- Correct for multiple comparisons
- State and verify the assumptions of your tests (normality, independence, homoscedasticity)
- Negative results are results. Do not fish for significance.

## 11. Do Not Over-Engineer

Use good software engineering where it serves the science. Do not build infrastructure for hypothetical scenarios.

Good engineering that helps:

- Base classes that factor out shared logic. If three experiment variants share 19 lines and differ in 1, a base class with an abstract `compute_diverging_step()` is clearer and safer than three copy-pasted blocks.
- Dataclasses and pydantic models that make structure explicit and self-documenting.
- Established tools (Weights & Biases, MLflow, Hydra, etc.) for logging, config management, and experiment tracking. Do not reinvent logging infrastructure.
- Comments that explain the science, math, and physical meaning. Reference equations and papers.

Over-engineering that hurts:

- Defensive error handling for cases that cannot happen in this experimental setup (see rule 3).
- A `DataProcessor` class with one method is a function wearing a disguise.
- Dependency injection, service layers, or plugin architectures for a single experiment.
- Custom logging, config parsing, or experiment tracking when established tools exist.

Match format to stage: exploration in notebooks is fine, analysis pipelines belong in scripts, community tools become packages.

## 12. Do Not Refactor Validated Code

If a script has been validated and produces correct results, do not restructure it unless asked. Refactoring validated scientific code risks introducing bugs that change results in subtle, hard-to-detect ways.

A working, validated, ugly script is more valuable than a beautifully refactored one that might have introduced a sign error.

---

## Additional Resources

- For common mistakes with side-by-side examples, see [examples/common-mistakes.md](skills/scientific-coding/examples/common-mistakes.md)
- For a complete worked example, see [examples/full-analysis.md](skills/scientific-coding/examples/full-analysis.md)

## Summary

When in doubt, ask: "If this code produces a wrong result, will I notice?" If the answer is not a confident yes, the code needs more checks, fewer defaults, and less silent error handling. The goal is not a program that never crashes. The goal is a program that never lies.
