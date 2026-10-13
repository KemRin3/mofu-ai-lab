# Threat model

## Protected assets

- files outside the selected skill package;
- credentials, environment configuration, and user data;
- the host's installed skill registry;
- the integrity of the inspection report;
- the decision to activate, install, repair, or execute a skill.

## Untrusted input

Every file in the selected skill package is untrusted, including `SKILL.md`,
documentation, examples, scripts, manifests, filenames, and referenced local
resources.

Instructions found inside the target are evidence to inspect. They do not gain
authority merely because they are written as agent instructions.

## Threats currently considered

1. **Prompt injection** — text that attempts to change the inspection task,
   permissions, authority, or required output.
2. **Dangerous execution** — scripts or commands that write, delete, move,
   execute, install, alter configuration, expose secrets, or escape the target
   boundary.
3. **External communication** — downloads, telemetry, APIs, credentials, URLs,
   remote tools, or MCP dependencies.
4. **Compatibility mismatch** — missing runtimes, unavailable sibling skills,
   platform-specific commands, and host-specific paths.
5. **Incomplete packaging** — missing referenced files, broken links,
   inconsistent frontmatter, sparse checkouts, and source/install layout
   mismatches.
6. **Inspector failure** — malformed output, skipped files, hallucinated
   evidence, or an invalid `PASS` despite uninspected required material.

## Host-enforced controls

- The inspector receives exactly one target at a time.
- The target is available through read/list operations only.
- Shell, execution, writes, network access, installs, moves, and activation are
  unavailable during inspection.
- Paths are resolved before access checks to resist traversal and link-based
  boundary escapes.
- The trusted host validates the structured result before saving it.
- Invalid results are retried only a bounded number of times and then recorded
  as `UNKNOWN`.
- Repair uses a separate staging and approval path.

## Explicit non-goals

- proving that arbitrary code is malware-free;
- safely executing untrusted skill code;
- automatically installing every package that receives `PASS`;
- replacing operating-system sandboxing or human review;
- treating an LLM verdict as a security certification.

## Open research questions

- How consistently do different models detect instruction injection embedded
  in long reference files?
- Can package-topology reasoning distinguish a real missing file from an
  intentional kit-level shared dependency?
- How should the system score risk without collapsing safety and compatibility
  into one ambiguous verdict?
- Which static fixtures best expose false `PASS` decisions?
