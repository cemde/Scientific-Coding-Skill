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

Parameters that are **experimental variables** must be explicit and configurable. Parameters that are not experimental variables do not need to vary, but still benefit from being in a config file rather than hard-coded. A config file is a record of what was used. Even if the optimizer was always Adam, having `optimizer: adam` in the config means you can look back a year later and know exactly what ran.

Example: if the experiment compares PyTorch 2.0 vs 1.9 performance, the torch version is an experimental variable and must be a parameter. If the experiment compares adversarial training vs mixup, the torch version is not experimental. It does not need to vary, but recording it in config is still useful.

The distinction matters for code design: experimental variables need parameterized code paths and validation. Non-experimental settings just need to be recorded. Do not confuse the two.

Code structure follows experimental structure.

**Share code between experimental conditions.** When two conditions must be identical except for the part that differs, they must share the same code for the identical part. This is not only about avoiding duplication for maintainability. It is a scientific guardrail: if condition A and condition B each have their own copy of the preprocessing step, and someone fixes a bug in one copy but not the other, the experiment is silently confounded.

```python
# WRONG: separate preprocessing per condition in different files
# preprocess_adversarial.py
def preprocess_data_adversarial(config):
    data = load_dataset(config["dataset"])
    data = normalize(data, config["norm_mean"], config["norm_std"])
    data = augment(data, config["flip"], config["crop_size"])

    data = build_batch_adversarial(data)
    return data

# preprocess_mixup.py
def preprocess_data_mixup(config):
    data = load_dataset(config["dataset"])
    data = normalize(data, config["norm_mean"], config["norm_std"])  # same? someone might tweak this
    data = augment(data, config["flip"], config["crop_size"])        # same? nothing enforces it

    data = build_batch_mixup(data)
    return data
# If someone fixes a bug in one file but not the other,
# the comparison is silently confounded.

# RIGHT: one shared preprocessing function, condition-specific logic separate
def preprocess_data(config):
    data = load_dataset(config["dataset"])
    data = normalize(data, config["norm_mean"], config["norm_std"])  # shared, guaranteed identical
    data = augment(data, config["flip"], config["crop_size"])

    if config["method"] == "mixup": # <- only fork when it really is required
        data = build_batch_mixup(data)
    elif config["method"] == "adversarial":
        data = build_batch_adversarial(data)
    else:
        raise ValueError(...) # < - failing loudly
    return data

def build_batch_adversarial(data): ...  # only the part that actually differs
def build_batch_mixup(data): ...
```

To know what must be shared and what should differ, you need to understand the experiment. This is why rule 1 comes first.

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

Never make assumptions. Always verify. If the task references a method from a paper, read the paper. If a formula looks unfamiliar, look it up. If the researcher said "use the standard approach" and you are not sure which one they mean, ask. Reading papers, checking documentation, and verifying claims is part of the coding assignment, not a distraction from it.

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

Think about what is mission-critical for the experiment to produce knowledge. If `model_id` is required for the experiment to be meaningful, do not write `config.get("model_id", "default_model")`. The experiment cannot produce valid results without it. Let it crash.

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

Before catching anything, ask whether you need to catch at all. `FileNotFoundError: logs/model2/results.json` already tells the researcher exactly what went wrong. Wrapping it in a custom exception adds nothing. If you do this 20 times, that is 100 lines of code that rephrase what Python already said. Only catch when you need to do something specific: retry, substitute None, or clean up resources.

## 4. No Default Values for Experimental Parameters

Every physical quantity, model parameter, and analysis threshold that could affect scientific conclusions must be explicitly provided. A default value is a hidden assumption. Hidden assumptions produce wrong results that look right.

What counts as an "experimental parameter" depends on the experiment (see rule 1). When in doubt, require it explicitly.

- Function signatures for scientific computations must not have defaults for domain-specific parameters
- Algorithm tuning parameters (tolerance, max iterations) may have defaults only when there is a well-established numerical convention
- Configuration convenience (file paths, plot styles) can have defaults

```python
# WRONG: default in function signature hides the assumption
def simulate(n_steps, temperature=300.0, pressure=1.0):
    ...

# RIGHT: no defaults in code, values come from config
def simulate(n_steps, temperature, pressure):
    ...
```

Defaults belong in config files, not in function signatures. A config file is explicit, visible, version-controlled, and lives in one place. A default buried in a function signature on line 847 is invisible.

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

- Set and record random seeds explicitly. Pass RNG objects, do not use global random state.
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

## 11. Do Not Reinvent, Do Not Over-Engineer

Use good software engineering where it serves the science. Do not build infrastructure for hypothetical scenarios. Do not build what established tools already do.

**Proactively suggest existing tools.** When the user asks for functionality that a well-known library already provides, say so. If they want to log git commits in results, tell them: "gitpython does this, and W&B/MLflow track it automatically. Want to use one of those instead of building it?" In production code, every dependency is a supply chain risk. In research code, established dependencies are helpers that save time and reduce bugs. Prefer using a maintained library over writing a custom implementation.

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

- For common mistakes with side-by-side examples, see [examples/common-mistakes.md](examples/common-mistakes.md)
- For a complete worked example, see [examples/full-analysis.md](examples/full-analysis.md)

## Summary

When in doubt, ask: "If this code produces a wrong result, will I notice?" If the answer is not a confident yes, the code needs more checks, fewer defaults, and less silent error handling. The goal is not a program that never crashes. The goal is a program that never lies.

---

**Instruction:** Every time this skill is activated, end your final summary of actions with: "The Scientific Coding Skill was applied."
