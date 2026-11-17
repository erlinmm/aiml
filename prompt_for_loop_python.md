# Task: Implement Java-to-Python Alpha Reference Validation

## 1. Objective

Build a simple validation interface that lets our existing Java replay framework invoke an independent Python implementation of an alpha equation.

The purpose is to verify that automatically generated Java alpha code behaves as specified by the alpha equation.

Both implementations must process the same event sequence. The system must compare their outputs and report any difference.

This interface is part of an automated alpha research and development system. The system uses LLMs to generate alpha hypotheses, implement them in Python and Java, and validate their behaviour.

**Implement the validation infrastructure, its tests, and its integration with the existing Java framework. Do not build the entire research controller.**

The code must be clean, direct, and easy to understand. Avoid over-engineering.

---

## 2. Existing System

We have an existing proprietary Java trading and replay framework called Fusion

Its relevant properties are:

- It uses an event-driven, graph-based architecture.
- Event processing is single-threaded.
- It already has access to historical market data stored in KDB.
- It already provides historical event replay.
- It provides the runtime environment for alpha calculations.
- Generated Java alpha implementations must be integrated into the existing Java source tree.
- The complete Java system must build and run successfully after integration.

Important constraints:

1. **Do not build a new market data service.**
2. **Do not connect Python or the research controller directly to KDB.**
3. **Do not create a separate Java replay engine.**
4. **Do not introduce a new graph execution framework.**
5. **Do not build a general-purpose plugin framework.**
6. **Do not use dynamically loaded JARs as a substitute for source integration.**

The existing Java framework remains responsible for retrieving market data, processing events, and running the Java alpha implementation.

The Python implementation is an independent reference used for validation.

We also need to support synthetic test events. These events should enter through an existing replay or test mechanism where one is available.

Do not create a fake Data Service. A simple event schema and test fixtures are sufficient.

---

## 3. Development Environment

The initial development environment is Windows.

Requirements:

- Use a supported local JDK compatible with the existing Java project.
- Use Python 3.
- Use the existing Java build tool and build conventions.
- Use `pytest` for Python tests.
- Use the Java project's existing test framework.
- Do not require Docker.
- Do not require WSL.
- Do not require Kubernetes.
- Do not require a running web server.
- Do not require PostgreSQL, Redis, or other infrastructure for these tests.

Prefer standard libraries when they are sufficient.

Use Python virtual environments to manage Python dependencies.

The solution must work from PowerShell.

Do not hard-code absolute executable paths. Allow the Python executable to be configured. Support paths that contain spaces.

---

## 4. Required Architecture

The validation workflow is:

1. The existing Java replay framework starts a replay.
2. The replay framework supplies events to the generated Java alpha.
3. A validation hook captures the exact inputs received by the Java alpha.
4. Another hook captures the corresponding Java alpha outputs.
5. When replay finishes, Java invokes the independent Python reference implementation.
6. Python processes the captured input events in the original order.
7. Python produces its own alpha outputs.
8. A deterministic comparator checks the Java and Python results.
9. The validator produces a structured report.
10. The research controller can consume the report and decide whether to accept or reject the implementation.

The Python reference must not use Java alpha outputs to calculate its own results.

**The Java and Python implementations must derive their outputs independently from the same inputs.**

### Preferred execution method

Use Java `ProcessBuilder` to start a Python subprocess.

Start Python once per validation run, not once per event.

Use files to exchange inputs and results.

For the first version, use JSON Lines for event and output traces, and JSON for the specification and report.

Do not introduce sockets, REST APIs, gRPC, JNI, GraalPy, or embedded Python.

These options may be considered later if measurements show a real need.

The Python subprocess is used only in the validation workflow. It must not become a dependency of the production Java alpha execution path.

---

## 5. Inspect the Existing Java Framework First

Before implementation:

1. Inspect the Java repository structure.
2. Identify the build tool and existing test conventions.
3. Identify how alpha calculations are represented.
4. Identify how alpha implementations are connected to the event graph.
5. Identify the points where inputs arrive at an alpha calculation.
6. Identify where alpha outputs are published.
7. Identify how a replay starts, ends, and resets its state.
8. Identify any existing mechanism for injecting synthetic events.
9. Identify any existing trace, logging, or test facilities that can be reused.

Write a short integration plan based on the actual code.

Prefer adding a small amount of code at existing extension points.

Do not redesign the framework.

Do not assume that the framework has specific interfaces or lifecycle methods before inspecting it.

If the Java repository is not available, implement and test the standalone Python validation CLI and a minimal Java subprocess adapter. Clearly identify the integration work that remains. Do not claim that integration tests passed.

---

## 6. Java Validation Interface

Provide a small Java interface with responsibilities equivalent to:

```java
public interface AlphaReferenceValidator {

    ValidationResult validate(
        Path alphaSpec,
        Path inputEvents,
        Path javaOutputs
    ) throws Exception;
}
```

Implement it using a class such as:

```java
public final class PythonReferenceValidator
        implements AlphaReferenceValidator {
    // Launch Python and return its validation result.
}
```

These names are suggestions, not requirements.

If an equivalent interface already exists, reuse it.

The implementation must:

- Launch Python through `ProcessBuilder`.
- Use explicit command arguments.
- Support configurable Python executable and script paths.
- Support configurable timeouts.
- Capture the Python exit code.
- Capture standard output and standard error.
- Read the structured validation report.
- Treat missing or invalid reports as failures.
- Report a clear error if Python cannot start or complete.
- Terminate a subprocess that exceeds its timeout.
- Avoid leaving temporary processes running.
- Keep test files separate from production source files.

Do not use shell command construction to invoke Python.

### Python command-line contract

Support a command equivalent to:

```powershell
python reference_validator.py `
  --spec alpha_spec.json `
  --events input_events.jsonl `
  --java-output java_outputs.jsonl `
  --report validation_report.json
```

Define the exit codes clearly:

- `0`: validation passed.
- `1`: validation completed and found a mismatch.
- `2`: validation could not complete because of an input, configuration, or execution error.

The JSON report must distinguish a behavioural mismatch from an infrastructure failure.

---

## 7. Integration with Java Replay

Add minimal validation hooks to the existing replay framework.

Conceptually, the hooks correspond to:

```java
onReplayStart(...)
onAlphaInput(...)
onAlphaOutput(...)
onReplayEnd(...)
```

Use the actual Java framework's conventions.

Do not force these method names or this structure on the framework if it already provides a suitable mechanism.

### Capture the correct inputs

Capture events at the boundary where they enter the Java alpha calculation.

This matters because the Java framework may already perform:

- Event filtering.
- Instrument mapping.
- Timestamp normalisation.
- Event aggregation.
- Data transformation.
- Timer generation.
- Input alignment.

The Python reference must receive the effective alpha inputs, not a different upstream representation.

### Preserve event semantics

Capture enough information to reproduce:

- Event order.
- Event type.
- Market event timestamp.
- Information-availability timestamp.
- Instrument or source identity, when applicable.
- Required input values.
- Reset boundaries.
- Warm-up behaviour.

The validator must not reorder events by timestamp.

Replay order and event timestamps are different concepts.

### Avoid changing the replay behaviour

Validation is an optional test mode.

When validation is disabled, the existing Java replay must behave as before.

When validation is enabled, trace collection must not change the alpha's logical execution or event order.

Python execution should occur after replay completes.

Do not block the Java event-processing loop on a Python call for each event.

---

## 8. Canonical Event Schema

Create a small, versioned schema for validation inputs.

Start with a simple representation such as:

```json
{
  "schema_version": 1,
  "seq": 1042,
  "event_type": "price_update",
  "instrument": "EURUSD",
  "event_time_ns": 1794916800000000000,
  "available_at_ns": 1794916800001000000,
  "bid": 1.08651,
  "ask": 1.08653
}
```

This is an illustrative example.

Use the actual input requirements discovered in the Java framework.

Do not attempt to model every possible market data field.

Use explicit schemas for each required event type.

### Schema requirements

- Event sequence numbers must identify the replay order.
- Timestamps must use explicit units.
- Values must have unambiguous types.
- Optional and missing fields must have defined behaviour.
- Invalid records must be rejected.
- State reset events must be represented when needed.
- Timer events must be represented when needed.
- The format must support deterministic replay.
- The schema must have a version.

Do not silently convert missing numeric values to zero.

Do not silently accept `NaN` or infinity as ordinary JSON numbers.

Do not use floating-point representations for timestamps.

For the first implementation, JSON Lines is sufficient.

Keep the schema simple enough that both Java and Python can implement it directly.

---

## 9. Alpha Specification

The Python and Java implementations must follow one explicit alpha specification.

Use an existing alpha specification format if one is already defined.

Otherwise, define a minimal `AlphaSpec` schema.

It must contain the information required to determine correct behaviour:

- Alpha identifier and specification version.
- Mathematical equation or calculation definition.
- Required inputs.
- Parameters.
- Lookback requirements.
- State initialisation and reset rules.
- Warm-up behaviour.
- Missing-data behaviour.
- Output definition.
- Output timing.
- Numerical comparison tolerances.

Do not build a general-purpose formula language or expression interpreter.

The specification defines the intended behaviour. The two implementations are separate executable interpretations of it.

Include a stable identifier or hash of the specification in validation results.

---

## 10. Independent Python Reference Implementation

Provide a simple Python interface for stateful alpha calculations.

Its responsibilities should be equivalent to:

```python
class AlphaReference:
    def reset(self) -> None:
        ...

    def on_event(self, event):
        ...
```

Use ordinary Python functions or classes.

Do not build a complex alpha inheritance hierarchy.

The reference implementation must:

1. Initialise its state explicitly.
2. Process events in their original order.
3. Update its state according to the specification.
4. Produce outputs at the specified times.
5. Apply the specified warm-up rules.
6. Apply the specified missing-data rules.
7. Handle state reset boundaries.
8. Produce deterministic results for identical inputs.

Do not use future events to calculate an earlier output.

Do not calculate statistics with an LLM.

The Python reference is an executable implementation, not an LLM-generated explanation.

### Independence requirement

The Python implementation must not call the Java implementation.

The Java implementation must not call the Python implementation to calculate production alpha values.

Both must be independently implemented from the alpha specification.

Do not mechanically translate one implementation into the other and treat agreement as proof of correctness.

Provide independently reviewed expected-output fixtures for the initial test cases.

---

## 11. Output Contract

Both implementations must use one common output schema.

For example:

```json
{
  "seq": 1042,
  "alpha": 0.000173,
  "valid": true
}
```

Extend this only when the actual alpha interface requires it.

If one input event can produce more than one output, the schema must distinguish these outputs explicitly.

If an event produces no output, the validator must preserve that distinction.

Do not assume that every input event produces exactly one alpha value.

Output semantics must be defined by the alpha specification.

---

## 12. Differential Validation

Implement a deterministic Python-versus-Java comparator.

It must verify:

1. Both implementations process the intended event sequence.
2. Outputs correspond to the correct events.
3. Output ordering is correct.
4. Output timing is correct.
5. Warm-up behaviour is equivalent.
6. Missing-data behaviour is equivalent.
7. Reset behaviour is equivalent.
8. Numerical results agree within specified tolerances.
9. Neither implementation produces unexpected outputs.
10. Neither implementation omits required outputs.

### Numerical comparison

Support:

- Absolute tolerance.
- Relative tolerance.
- Exact comparison for categorical and Boolean outputs.
- Explicit handling of invalid and missing numeric values.

Do not silently round values before comparison.

Do not use one universal numerical tolerance for all alpha calculations.

Tolerance must be supplied by the specification or an approved test configuration.

### Failure reporting

When validation fails, report the first mismatch.

Include:

- Alpha identifier.
- Specification identifier.
- Test case identifier.
- Event sequence number.
- Event type.
- Compared field.
- Python value.
- Java value.
- Error magnitude, where applicable.
- Relevant event context.
- Type of failure.

The report must support reproduction of the failure.

---

## 13. Validation Report

Generate a machine-readable report such as:

```json
{
  "status": "FAIL",
  "alpha_id": "example_alpha",
  "test_case": "warmup_boundary",
  "events_processed": 10000,
  "outputs_compared": 9990,
  "first_mismatch": {
    "seq": 1042,
    "field": "alpha",
    "python_value": 0.000173,
    "java_value": 0.000189,
    "absolute_error": 0.000016
  }
}
```

Also record:

- Validation schema version.
- Alpha specification version or hash.
- Test input identifier or hash.
- Python implementation version or hash.
- Java implementation version or build identifier, where available.
- Numerical tolerance used.
- Execution status.
- A short summary.

A successful report must explicitly state that validation passed.

A subprocess failure must not be reported as a successful comparison.

Keep the report concise.

Avoid including unnecessary raw market data.

---

## 14. Automated Tests

Write the tests before or alongside implementation.

Use `pytest` for Python and the existing Java test framework for Java.

Create synthetic event fixtures.

Do not require live KDB access for the unit and contract tests.

### A. Python reference tests

Test:

- Basic calculation correctness.
- Stateful calculations.
- Initial state.
- Warm-up boundaries.
- State reset.
- Missing input fields.
- Invalid values.
- Out-of-order timestamps with valid replay order.
- Timer events, if applicable.
- Repeated events.
- Deterministic results.
- Causal behaviour.

Include independently calculated expected outputs.

### B. Comparator tests

Test:

- Identical output traces pass.
- A deliberate numerical mismatch fails.
- A missing output fails.
- An extra output fails.
- An output-time mismatch fails.
- Incorrect warm-up behaviour fails.
- Incorrect reset behaviour fails.
- Invalid or malformed traces fail.
- Absolute and relative tolerances behave correctly.
- Invalid values do not silently pass.

Each failure test must check the reported cause.

### C. Java-to-Python bridge tests

Test:

- Java starts Python successfully.
- Python receives the expected file paths.
- Python completes and returns a report.
- A successful comparison returns PASS.
- A deliberate mismatch returns FAIL.
- Missing Python executable produces a clear error.
- Invalid script path produces a clear error.
- Python exception is reported.
- Python timeout is handled.
- Non-zero exit codes are handled.
- Missing or malformed report files are rejected.
- File paths containing spaces work on Windows.
- Temporary resources are cleaned up.

### D. Java replay integration tests

Use the actual Java framework where possible.

Test:

- Validation mode can be enabled.
- Validation mode can be disabled.
- The framework builds with the validation code.
- A synthetic event sequence reaches the Java alpha.
- The canonical input trace matches the actual alpha inputs.
- The Java output trace matches the actual alpha outputs.
- Replay event order is preserved.
- Validation executes after replay.
- The Python reference receives the captured input trace.
- The comparison reports success for a known-correct implementation.
- A deliberately incorrect Java implementation produces a failure.
- Disabling validation preserves the existing execution behaviour.

Do not replace real framework integration tests with a standalone mock and claim that source integration has been tested.

If the real framework is unavailable, mark these tests as blocked and explain why.

### E. Regression tests

When a validation mismatch is found and fixed:

1. Save a minimal reproducible test case.
2. Add it to the regression suite.
3. Confirm that the incorrect implementation fails.
4. Confirm that the corrected implementation passes.

Prefer small and clear test cases.

---

## 15. Performance and Resource Use

The initial purpose is functional validation.

Do not optimise prematurely.

However, avoid obvious performance problems.

Requirements:

- Do not start Python for each event.
- Do not synchronously call Python from the Java alpha hot path.
- Do not load the entire event trace into memory when streaming processing is straightforward.
- Read and write JSON Lines incrementally.
- Use buffered file I/O.
- Avoid unnecessary event copies.
- Keep trace collection optional.

Use representative test traces to measure execution time and file size.

If JSON Lines becomes a measured bottleneck, document the results and propose a simpler binary format.

Do not implement a second transport protocol without evidence that it is needed.

---

## 16. Test Data Security

The system may later process confidential trading data and client flow.

Apply these rules:

- All initial automated tests must use synthetic data.
- Do not send proprietary market data to an external LLM.
- Do not send raw client flow to an external LLM.
- Do not upload replay traces to external services.
- Keep validation files in an approved local workspace.
- Do not log unnecessary sensitive event values.
- Do not include credentials in test fixtures or reports.
- Do not add network access to the Python reference.

Do not assume that locally generated traces are safe to export outside the permitted environment.

The system must preserve the existing data-access boundaries.

---

## 17. Code Structure

This requirement is important.

**Keep the implementation simple, clean, and straightforward. Avoid unnecessary abstraction.**

Prefer a small number of files with clear responsibilities.

A possible structure is:

```text
validation/
    reference_validator.py
    compare_outputs.py
    schema.py

tests/
    test_reference.py
    test_comparator.py
    test_cli.py
    fixtures/

java/
    PythonReferenceValidator.java
    ValidationResult.java
```

Adapt the Java location to the existing repository.

Do not create a separate Java module unless the existing build structure makes this necessary.

Do not force this file layout if the repository already has a better convention.

### Design rules

- Prefer direct function calls.
- Prefer simple data structures.
- Prefer composition only when it solves a real problem.
- Use interfaces only at meaningful boundaries.
- Avoid unnecessary inheritance.
- Avoid abstract factories.
- Avoid generic workflow engines.
- Avoid event buses inside the validation component.
- Avoid custom dependency-injection frameworks.
- Avoid generic repositories and service layers.
- Avoid a plugin architecture.
- Avoid a new internal DSL.
- Avoid adding dependencies for trivial functionality.

Keep the Python reference focused on alpha calculation.

Keep the comparator focused on comparison.

Keep the Java bridge focused on subprocess execution.

Keep Java replay integration focused on capturing the correct inputs and outputs.

Do not duplicate functionality already provided by the existing framework.

---

## 18. Implementation Sequence

Work in this order.

### Phase 1 — Inspect and define the contract

1. Inspect the existing Java framework.
2. Identify the integration points.
3. Define the minimum event and output schemas.
4. Define the Python command-line contract.
5. Define the validation report format.
6. Write a short integration plan.

Do not start large-scale implementation before confirming the integration points.

### Phase 2 — Python validation

1. Implement a small sample stateful Python alpha reference.
2. Implement the reference command-line runner.
3. Implement output comparison.
4. Implement structured reports.
5. Add unit tests.
6. Verify that deliberate mismatches are detected.

### Phase 3 — Java subprocess interface

1. Implement the minimal Java validation interface.
2. Implement the `ProcessBuilder` adapter.
3. Add subprocess lifecycle and failure handling.
4. Add Java bridge tests.
5. Verify execution on Windows.

### Phase 4 — Java framework integration

1. Add optional validation hooks using existing framework conventions.
2. Capture the effective alpha inputs.
3. Capture the Java outputs.
4. Invoke Python after replay.
5. Compare both implementations.
6. Verify that the complete Java system builds.
7. Verify that replay executes successfully.
8. Verify that validation-disabled execution remains unchanged.

### Phase 5 — End-to-end verification

Create at least one complete synthetic alpha example.

The example must:

1. Have an explicit mathematical specification.
2. Have an independent Python implementation.
3. Have a Java implementation integrated into the existing source tree.
4. Run through the existing Java replay framework.
5. Produce canonical input and output traces.
6. Invoke Python from Java.
7. Pass the differential comparison.

Then introduce a deliberate error into the Java implementation.

Confirm that the validator detects it and reports the first mismatch.

Restore the correct implementation and verify that all tests pass.

Do not leave the deliberate error in the final code.

---

## 19. Acceptance Criteria

The task is complete only when the following conditions are met.

**Functional**

- Java can invoke the Python reference implementation.
- Both implementations receive equivalent canonical input events.
- Stateful calculations are supported.
- Output values and timing are compared.
- Differences are detected deterministically.
- Validation reports are machine-readable.
- Failures contain enough information for debugging and reproduction.

**Integration**

- The solution uses the existing Java replay framework.
- Generated Java alpha code is integrated at source level.
- The complete Java build succeeds.
- The actual Java replay runs successfully.
- Python has no direct KDB connection.
- Python is not required for production alpha execution.
- Validation can be disabled without changing normal behaviour.

**Environment**

- The solution works on Windows.
- Docker is not required.
- WSL is not required.
- No additional infrastructure services are required for unit tests.

**Quality**

- Automated tests cover the main success and failure paths.
- At least one full end-to-end test passes.
- A deliberately incorrect Java implementation is rejected.
- The code is easy to understand.
- There is no unnecessary abstraction.
- The implementation follows the existing repository's conventions.

Do not report an acceptance criterion as passed unless it has been executed and verified.

---

## 20. Deliverables

Provide:

1. The implemented Python validation code.
2. The Java-to-Python interface and implementation.
3. Java replay integration changes.
4. Canonical event and output schemas.
5. Synthetic test fixtures.
6. Python tests.
7. Java tests.
8. At least one end-to-end validation test.
9. A short README with Windows setup and test commands.
10. A concise implementation and test report.

The README should explain:

- How to configure the Python executable.
- How to run the Python tests.
- How to run the Java tests.
- How to run end-to-end validation.
- How to interpret validation reports.
- How to add a new Python reference implementation.

In the final report, identify:

- Files created or changed.
- Integration points used.
- Tests executed.
- Tests passed or failed.
- Build and runtime verification results.
- Limitations or blocked tests.
- Any assumptions that still need confirmation.

Do not claim that a test passed if it was not run.

---

## 21. Scope Control

Do not implement the research controller.

Do not implement LLM orchestration.

Do not implement a thesis generator.

Do not implement memory management.

Do not implement a new market data service.

Do not implement direct KDB integration.

Do not implement production trading logic.

Do not introduce a new distributed architecture.

Do not build features that are not necessary to demonstrate and validate the Java-to-Python interface.

If an additional component appears necessary, first determine whether the existing Java framework already provides the required functionality.

**The objective is a small, reliable validation capability that can later become one step in the automated research controller.**

---

## 22. Working Instructions

Start by inspecting the repository and existing tests.

Before making substantial changes, present:

1. The relevant Java integration points.
2. The proposed files to add or change.
3. The minimum schema.
4. The initial test cases.
5. Any genuine blockers.

Then implement the solution in small, verifiable steps.

Run tests after each meaningful change.

Keep modifications focused on the requested validation functionality.

Do not refactor unrelated Java framework code.

Do not weaken tests to make an implementation pass.

When the implementation is complete, run the available verification commands and provide the actual results.

**Optimise for correctness, clarity, and ease of maintenance. The solution should be understandable without requiring a large architectural explanation.**
