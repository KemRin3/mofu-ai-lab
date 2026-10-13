# LLMBuilder integration notes

The current experimental host integrates Skill Quarantine into a local
LLMBuilder/OpenCode workflow. The full application is not included in this
repository; this document records the relevant behavior without publishing
machine-specific configuration or inspected third-party packages.

## Controller responsibilities

The trusted Python controller:

- selects one pending skill package;
- starts a fresh dedicated quarantine-agent session;
- injects the canonical `SKILL.md` procedure;
- enables quarantine-specific environment flags;
- disables web search and unrelated image inputs;
- extracts and validates `QUARANTINE_RESULT_JSON`;
- stores an individual JSON report and rebuilds an aggregate Markdown summary;
- resumes from the next unprocessed package;
- converts repeated invalid responses into an explicit `UNKNOWN` result.

## Boundary plugin responsibilities

A host plugin applies the runtime access boundary. In quarantine mode it allows
read/list operations only when their resolved paths stay inside the workspace
or the explicitly selected target. Other tools and operations are denied.

The implementation also tests symlink/junction resolution rather than relying
only on string-prefix path checks.

## Tests in the local prototype

The current test suite covers:

- read/list-only permission construction;
- path traversal and external-path rejection;
- symlink/junction boundary behavior;
- independent denial logging;
- required result-field validation;
- per-skill report persistence and resumable queues;
- fail-closed behavior when the canonical quarantine procedure is missing;
- forced use of a dedicated agent and fresh session;
- bounded retries followed by `UNKNOWN`.

## Why the host matters

The inspection procedure is useful guidance, but the prompt is not the security
boundary. A target skill can tell the model to ignore prior instructions or run
an installer. The design assumes those actions remain technically unavailable,
even if the model follows the hostile text.
