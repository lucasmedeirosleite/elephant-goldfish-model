<!-- Copyright 2026 Google LLC. SPDX-License-Identifier: Apache-2.0 -->

# T3 Code integration (experimental)

T3 Code runs Claude Code and Codex threads underneath its orchestration tools.
This guide targets the T3 model-role interface: the session exposes T3's
`delegate_task` tool, and `orchestrator_capabilities` lists `agentModelRoles`.
Check those interfaces in the current session rather than inferring support
from the product name. Installation alone does not establish workflow or
Goldfish isolation support.

## Install

Install EGM as a Claude Code plugin and/or Codex skills using the
[Claude Code](claude.md#invocation-and-resources) and
[Codex](codex.md#install-and-locate-the-plugin) installation instructions.
T3 threads on those providers load the installed EGM resources. Install the
whole plugin, not just `skills/`: personas, standards, and supporting files in
`agents/`, `resources/`, and `docs/` are required at run time.

Start a fresh T3 thread after installation. Use the installed skill's absolute
path and resolved bundle root; do not guess a cache location or version.
No separate T3 manifest or fork of the canonical skills and personas is needed.

## Paths and dispatch

Resolve `agents/`, `resources/`, and `docs/` against the installed bundle root.
Resolve `references/` against the invoked skill's directory. Resolve user files
and generated artifacts against the project working directory, not the plugin
cache. Confirm the persona file and its frontmatter `name` before dispatch:
`egm-design`, for example, comes from `agents/design_critic.md`.

Use T3's `delegate_task` for every subagent, including same-provider children
and reviewers started by a synthesizer. Do not use the underlying provider's
native spawn tool: each child must be a T3 thread so the user's model-role
mapping applies. Map the canonical EGM task as follows:

- Set `agent` to the persona's frontmatter `name`, not its filename or a
  provider-prefixed alias.
- Set `task` to `Read <absolute path to agents/<file>.md> first and act as that persona`,
  followed by the EGM prompt verbatim. Replace the path with the confirmed
  installed persona path; retain the prompt's required reads and deliverables.
- Omit `target`; the user's role settings select the provider and model.
- Use `mode: "async"` for parallel stages, such as the six brainstorm lenses
  or reviewers within one review stage. Follow T3's wake-up delivery and wait
  for every child to complete before synthesizing the stage.
- Use `mode: "wait"` for a single sequential step.

The synthesizer runs as a child and delegates reviewers as grandchildren.
Its task text must explicitly instruct it to start those reviewers. T3 supports
nesting, so the standard nested brainstorm and review flows apply. Pass the
same persona-path and `agent` mapping rules to the synthesizer.

Pass absolute paths for the target, standards, and explicitly selected sources.
Do not add parent conversation history, private reasoning, or unsolicited
memory to a Goldfish task. A launch is not a completed review: require actual
completion and required reads, and report missing evidence as incomplete.
Keep review stages sequential and retain human approval gates; parallelize only
within the current stage. Do not send follow-up design context to Goldfish
threads to compensate for an incomplete initial task.

## Model roles

Configure `modelRoles` and `agentModelRoles` in the global
`<T3 state dir>/settings.json`, not a project settings file. Each model role
is an ordered, nonempty list of `{instanceId, model, options?}` entries.
`agentModelRoles` maps persona names to role names. The role names below are
examples; replace the model selections with models available to your provider
instances. Model `options`, when supplied, use the provider's option descriptors.

This example defines `slow`, `task`, and `smol`, with an alternate provider
for each role:

```json
{
  "modelRoles": {
    "slow": [
      { "instanceId": "claudeAgent", "model": "claude-fable-5-1" },
      { "instanceId": "codex", "model": "gpt-6-astra" }
    ],
    "task": [
      { "instanceId": "codex", "model": "gpt-6.1-sol" },
      { "instanceId": "claudeAgent", "model": "claude-sonnet-5-5" }
    ],
    "smol": [
      { "instanceId": "claudeAgent", "model": "claude-haiku-5-5" },
      { "instanceId": "codex", "model": "gpt-6-luna" }
    ]
  }
}
```

Merge this recommended mapping for all nine canonical personas into the same
settings object. Adjust the example role assignments to your own roles:

```json
{
  "agentModelRoles": {
    "egm-synthesizer": "task",
    "egm-design": "slow",
    "egm-zoom-out": "slow",
    "egm-codebase-details": "task",
    "egm-comprehension": "task",
    "egm-implementation-readiness": "task",
    "egm-structural": "smol",
    "egm-style": "smol",
    "egm-bootstrap-orchestrator": "task"
  }
}
```

For local delegations, model selection precedence is:

1. A mapped agent uses its role; any caller-supplied `target` is ignored.
2. Otherwise, an explicit provider/model target is used.
3. Without either, a `task` role is used if defined.
4. Otherwise, the child inherits the caller's model.

A mapping to an unknown role is treated as unmapped. Remote-machine delegations
ignore roles and retain their existing target behavior.

Subscriptions that are out of quota are skipped at launch. On `usage_limit` or
`provider_error`, T3 relaunches the child fresh on the next role entry under the
same task id. The failed attempt stays visible as a child thread. This is a
new attempt, not a continuation of the failed child's conversation. If no usable
entry remains, report the task failure rather than claiming a completed review.

## Verification limits

T3 support is **experimental** until brainstorm, draft, and review have each
run end to end and the runs are recorded in the
[TESTING.md harness record](https://github.com/google/elephant-goldfish-model/blob/main/TESTING.md#harness-test-record).
Offline checks do not establish those live outcomes:

```sh
pytest tests/unit/test_harnesses.py -k t3code
pytest tests/unit
```

These tests check canonical persona names in the recommended mapping and the
harness listing, along with the existing packaging contracts. They do not
exercise live T3 dispatch, provider loading, fallback, or reviewer isolation.
Record the host version, EGM commit, provider/model selections, starts,
completions, required reads, artifacts, and approval pauses for live workflows.

Known limitations:

1. T3 children are ordinary threads and load project instruction files. There
   is no `omitClaudeMd` equivalent; a fresh child does not exclude those files.
2. There is no read-only runtime mode for reviewers. T3's narrow modes pause
   for approval rather than enforcing a read-only reviewer toolset. Persona
   instructions are not a permission boundary.
3. Goldfish threads accept follow-up messages. Their continued availability
   does not preserve Goldfish isolation if later messages supply design context.

## Interface sources

The model-role integration uses these interface sources in the
[T3 Code fork](https://github.com/lucasmedeirosleite/t3code).
Check the current model-role checkout; this guide does not establish that a
released T3 version exposes the interface.

- `packages/contracts/src/orchestratorMcp.ts`: `delegate_task` inputs and
  `orchestrator_capabilities` output.
- `packages/contracts/src/settings.ts`: global `modelRoles` and
  `agentModelRoles` settings.
- `apps/server/src/mcp/OrchestratorMcpService.ts`: role resolution and launch.
- `packages/provider-core/src/server/orchestrationInstructions.ts`: waiting,
  wake-up delivery, and nested delegation instructions.

The underlying resource conventions are documented in the
[Claude Code guide](claude.md) and [Codex guide](codex.md).
