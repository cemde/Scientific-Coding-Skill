---
name: scientific-coding
description: Rules for writing scientific and research code. Activates when writing code for data analysis, simulations, numerical methods, statistical analysis, or any computation whose results feed into scientific conclusions.
user-invokable: false
---

# Scientific Coding Principles

Industry code serves users. Scientific code serves truth. A wrong result that looks right is the worst possible outcome. Every rule below follows from this.

These are not general coding tips. They are behavioral rules for developers accustomed to industry patterns who now need to write code where the output feeds into scientific conclusions.

## 1. Understand the Experiment Before Writing Code

The experimental design determines the code architecture. Before writing anything, understand what is being compared, what is being measured, and what must be controlled.

Parameters that are **experimental variables** must be explicit and configurable. Parameters that are **not** experimental variables should be fixed. Making everything configurable adds complexity and creates opportunities for error.

Example: if the experiment compares PyTorch 2.0 vs 1.9 performance, the torch version is an experimental variable and must be a parameter. If the experiment compares adversarial training vs mixup, the torch version is not experimental -- fix it and move on. Making it configurable here adds no scientific value and introduces a dimension of variation that muddies the results.

This means you must understand the research question before deciding what to parameterize, what to hard-code, and what to validate. Code structure follows experimental structure.

## 2. When Uncertain, Ask the Researcher

In industry, a developer can make reasonable decisions about button placement or error messages without asking. In science, a coding agent making assumptions about methodology is dangerous.

Do not guess on:
- Which statistical test to use
- How to handle missing data or outliers
- What preprocessing or normalization to apply
- What parameter values to use
- What approximations are acceptable
- How to define a metric or threshold

A plausible-sounding but wrong choice here does not just break a feature. It can invalidate results, waste months of work, or lead to retracted publications. "I chose X because it seemed reasonable" is not acceptable when the choice affects scientific conclusions.

When the right approach is not clear from context or documentation, stop and ask. This is the single most important difference from industry coding, where initiative and reasonable defaults are virtues.

## 3. Fail Loudly, Never Silently

A crash is always better than a silently wrong result. Do not catch exceptions broadly. Do not provide fallback values. Do not recover gracefully from unexpected conditions.

- No bare `except:` or `except Exception:`
- No `try/except` around scientific computations unless handling a specific, expected condition
- Use assertions to verify invariants, intermediate results, and array shapes
- If a value is outside its physically meaningful range, raise an error
- Do not suppress warnings from scientific libraries (NumPy, SciPy, etc.) -- they often indicate real numerical problems like convergence failures or ill-conditioned matrices
- Never silently extrapolate beyond data range -- error or warn explicitly

Think about what is mission-critical for the experiment to produce knowledge. If a config field like `model_id` is required for the experiment to be meaningful, do not write `config.get("model_id", "default_model")`. If it is missing, the experiment cannot possibly generate valid results. Let it crash. Defensive handling of impossible-but-fatal cases hides the real problem.

This is the opposite of web development, where every edge case needs graceful handling for uptime. In science, graceful handling of a missing essential parameter means you might run an experiment that produces meaningless results and not notice.

```python
# WRONG: defensive handling hides a fatal problem
model_id = config.get("model_id", "resnet50")  # silently uses wrong model

# RIGHT: this code is always paired with this config -- model_id is essential
model_id = config["model_id"]  # crashes immediately if missing
```

When catching exceptions is necessary (e.g., API timeouts, transient I/O errors), the error must propagate in a form that stays visible downstream. The question is not whether to catch, but what the failure looks like to the code that computes metrics.

```python
# WRONG: fabricates a plausible-looking result
try:
    prediction = classify(text)
except TimeoutError:
    prediction = ""  # empty string looks like a real prediction
    correct = 0      # counted as incorrect -- silently poisons accuracy

# RIGHT: failure stays visible through the pipeline
try:
    prediction = classify(text)
except TimeoutError:
    return {"prediction": None, "correct": None, "error": "timeout"}
    # computing mean([1, 0, None]) will crash -- forcing you to handle it explicitly
```

The principle: a caught error must not produce a value that could be mistaken for a real result. Use None, NaN-that-propagates, or a distinct error type -- anything that will cause a downstream crash if someone computes a metric without accounting for the failure.

Before catching anything, ask whether you need to catch at all. `FileNotFoundError: logs/model2/results.json` already tells the researcher exactly what went wrong and where. Wrapping it in a custom exception or reraising with a message adds nothing. If you do this 20 times across a codebase, that is 100 lines of code that exist only to rephrase what Python already said. Let standard errors propagate. Only catch when you need to do something specific with the failure (retry, substitute None, clean up resources).

## 4. No Default Values for Experimental Parameters

Every physical quantity, model parameter, and analysis threshold that could affect scientific conclusions must be explicitly provided. A default value is a hidden assumption. Hidden assumptions produce wrong results that look right.

What counts as an "experimental parameter" depends on the experiment (see rule 1). But when in doubt, require it explicitly.

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

## 5. Be Explicit About Everything

State every assumption in code. Units, array shapes, coordinate systems, reference frames, sign conventions. If two quantities need to be in the same units, convert explicitly and visibly.

- Document units in variable names or comments at every step of a computation. Better: use a units library.
- Verify array shapes explicitly before operations that rely on broadcasting
- Use established constant libraries (`scipy.constants`, `astropy.units`) instead of hard-coded values
- Document which convention or reference you use when multiple exist
- If the units do not work out, the code is wrong. Dimensional analysis is a debugging tool.

```python
# WRONG: unclear what units, relies on implicit broadcasting
result = energy * temperature

# RIGHT: units and shapes are explicit
energy_joules = energy_ev * EV_TO_JOULE  # convert eV to J
assert energy_joules.shape == temperature_kelvin.shape
result_joules = energy_joules * temperature_kelvin
```

## 6. Correctness Over Performance

A fast wrong answer is worthless. Numerical stability, precision, and correctness always come before speed. Optimize only after profiling shows a real bottleneck, and only in ways that preserve correctness.

- Use numerically stable algorithms (logsumexp, Kahan summation, etc.)
- Be aware of catastrophic cancellation, accumulated rounding error, and precision loss
- Document when and why an approximation is used, including its error bounds
- Choose interpolation methods for physical reasons, not convenience (linear vs spline matters for energy conservation)
- Prefer higher precision when the cost is negligible

## 7. Raw Data Is Immutable

Never modify input data. Never overwrite raw data files. Every transformation creates a new object. The chain from raw data to final result must be traceable.

- Copy data before transforming
- Keep raw data files read-only
- Log every transformation applied to data
- Prefer pure functions that take inputs and return outputs over mutable state
- Do not store intermediate computation state in globals or object attributes that get silently reused
- Store intermediate results when computations are expensive

## 8. Reproducibility Is Non-Negotiable

Every result must be reproducible. Not approximately. Exactly.

- Set and record random seeds explicitly -- pass RNG objects, do not use global random state
- Pin dependency versions (exact versions, not ranges)
- Log all parameters, software versions, and environment details with results
- Use deterministic algorithms where possible
- If non-deterministic algorithms are necessary, document why and how to control variance
- For critical results, save the exact script that produced them alongside the output

## 9. Validate Against Reality, Not Specifications

Scientific validation is fundamentally different from industry testing. Industry tests check "does the software match its spec." Scientific validation checks "does the code produce results consistent with physical or mathematical reality."

- Test against analytical solutions where they exist (the most powerful validation there is)
- Verify conservation laws: energy, mass, momentum, probability must be conserved where physics requires it
- Test limiting cases and symmetries (T->0, t->infinity, symmetric input -> symmetric output)
- Check convergence: refining the grid, timestep, or sample size should improve results
- Use regression tests: after refactoring, results must be identical or within documented tolerance
- Visualization is a legitimate debugging tool in science. Plot intermediate results. Patterns visible in a plot catch errors that assertions miss.
- "It runs without error" proves nothing. A test that only checks the code does not crash is nearly worthless.
- Coverage percentage is meaningless for scientific correctness. 100% line coverage does not mean the physics is right.

## 10. Statistical Honesty

- Do not select statistical tests after seeing the data
- Report effect sizes, confidence intervals, and sample sizes, not just p-values
- Correct for multiple comparisons
- State and verify the assumptions of your tests (normality, independence, homoscedasticity)
- Negative results are results. Do not fish for significance.

## 11. Do Not Over-Engineer

Use good software engineering where it serves the science. Do not build infrastructure for hypothetical scenarios.

Good engineering that helps scientific code:
- Base classes that factor out shared logic. If three experiment variants share 19 lines and differ in 1, a base class with an abstract `compute_diverging_step()` is clearer and safer than three copy-pasted blocks.
- Dataclasses and pydantic models that make structure explicit and self-documenting.
- Established tools (Weights & Biases, MLflow, Hydra, etc.) for logging, config management, and experiment tracking. Do not reinvent logging infrastructure.
- Comments that explain the science, math, and physical meaning -- not the syntax. Reference equations and papers.

Over-engineering that hurts scientific code:
- Defensive error handling for cases that cannot happen in this experimental setup. The code is paired with this config. Do not guard against the config being wrong in ways that would make the experiment meaningless (see rule 3).
- Building configurability for things that are not experimental variables (see rule 1). If the experiment does not vary the optimizer, do not build an optimizer factory.
- A `DataProcessor` class with one method is a function wearing a disguise.
- Dependency injection, service layers, or plugin architectures for a single experiment.
- Custom logging, custom config parsing, or custom experiment tracking when established tools exist.

Match format to stage: exploration in notebooks is fine, analysis pipelines belong in scripts, community tools become packages.

## 12. Do Not Refactor Validated Code

If a script has been validated and produces correct results, do not restructure it unless asked. Refactoring validated scientific code risks introducing bugs that change results in subtle, hard-to-detect ways.

A working, validated, ugly script is more valuable than a beautifully refactored one that might have introduced a sign error. Treat validated scientific code as you would a published equation: carefully.

---

## Additional Resources

- For common mistakes with side-by-side examples, see [examples/common-mistakes.md](examples/common-mistakes.md)
- For a complete worked example, see [examples/full-analysis.md](examples/full-analysis.md)

## Summary

When in doubt, ask: "If this code produces a wrong result, will I notice?" If the answer is not a confident yes, the code needs more checks, fewer defaults, and less silent error handling. The goal is not a program that never crashes. The goal is a program that never lies.
