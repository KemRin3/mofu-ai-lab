# Skill Quarantine

> **Status: research prototype / 研究中**

Skill Quarantine is an experimental workflow for inspecting untrusted agent
skills before they are activated or installed.

An agent skill can contain more than documentation. It may reference scripts,
installers, network services, credentials, sibling packages, or host-specific
commands. It can also contain instructions that attempt to influence the agent
performing the inspection. This experiment asks a narrow question:

> Can an LLM inspect a skill as untrusted data, while a trusted host enforces a
> read-only boundary and saves a structured result?

This is not a malware scanner, sandbox, or security guarantee. It is a working
research prototype for improving the intake process around agent skills.

## What is included

- [`SKILL.md`](SKILL.md) — the inspection procedure used by the quarantine agent.
- [`THREAT_MODEL.md`](THREAT_MODEL.md) — current trust boundaries, threats, and known limitations.
- [`sample-result.json`](sample-result.json) — an anonymized example of the result contract.
- [`LLMBUILDER_INTEGRATION.md`](LLMBUILDER_INTEGRATION.md) — notes from the current host integration.

## Inspection model

The workflow deliberately separates two questions:

1. **Safety:** Is activating the skill acceptably safe?
2. **Compatibility:** Can the packaged skill work on the current host as described?

Each question receives one of four verdicts:

| Verdict | Meaning |
|---|---|
| `PASS` | The inspected evidence supports activation or compatibility. |
| `HOLD` | A known issue requires review or setup before proceeding. |
| `FAIL` | The inspected evidence shows the package should not proceed. |
| `UNKNOWN` | Required evidence could not be inspected or validated. |

`overall_verdict` is `PASS` only when both independent verdicts are `PASS`.
An uninspected required item must remain `UNKNOWN`; absence of evidence is not
treated as evidence of safety.

## Current workflow

```text
untrusted skill package
        |
        v
trusted host selects exactly one target
        |
        v
read/list-only quarantine agent
        |
        +-- prompt-injection review
        +-- dangerous-operation review
        +-- external-communication review
        +-- dependency/platform review
        +-- package-completeness review
        |
        v
structured JSON verdict
        |
        v
trusted host validates and persists the report
```

Repair is a separate, approval-gated phase. The inspection agent does not
download dependencies, execute target code, move the package, or activate the
skill.

## Experimental observations

The current local prototype has been used on a batch of more than 60 skill
packages. It identified several classes of packaging and compatibility issues:

- files documented by a skill but absent from the installed artifact;
- dependencies on sibling skills that were not packaged together;
- Unix- or Claude-specific path conventions on a Windows/Codex host;
- features that were safe to read but required live services or machine setup;
- invalid or incomplete LLM result payloads, which were retained as `UNKNOWN`.

Some incomplete packages were later repaired in a separate staging area after
their source and pinned revision were verified. Those repair operations are not
part of the read-only inspection contract.

## Running it

`SKILL.md` is designed to be injected by a trusted controller that can enforce:

- read/list access only;
- access to exactly one target package;
- no shell, execution, network, install, move, or activation capability;
- validation of the final `QUARANTINE_RESULT_JSON` payload;
- report persistence outside the untrusted package.

Using the prompt without those host-enforced boundaries does **not** reproduce
the security model.

## Research status and limitations

- The inspector is still an LLM and can miss suspicious behavior.
- Static inspection does not prove that executable code is benign.
- The result depends on the files made visible to the inspector.
- Compatibility checks may require a real runtime or service and therefore
  remain `HOLD` or `UNKNOWN` during quarantine.
- The present implementation is coupled to a local LLMBuilder/OpenCode host;
  only the portable inspection contract is published here.
- False positives and inconsistent classifications still require human review.

The immediate research direction is to build a small adversarial corpus of
skills containing prompt injection, hidden execution paths, broken references,
and ambiguous package layouts, then measure detection consistency across
models and repeated runs.
