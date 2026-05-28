# Workspace Resource Plugin Guide and Contract

- **Status:** Proposed
- **Author:** Hermes Agent
- **Created:** 2026-05-27
- **Scope:** Prompt-level contract and guide for workspace-aware plugins that consume the workspace resource

## Problem

Workspace-aware plugins need to behave consistently even when the model, agent, or host process is running somewhere else.

If a plugin derives workspace identity from its own runtime location, it can attach the wrong workspace, mislabel work, or drift across sessions. The contract has to describe the resource itself, the identity rules for that resource, and the maintenance rules that keep the contract current as the repo evolves.

This document is meant to be a guide for building a plugin around that resource. It is not a code-level resolver. It is a human- and model-facing contract that says how to think about the workspace, how to name it, and how to keep the docs honest.

## Intended audience

- plugin authors
- prompt authors
- maintainers reviewing workspace-aware changes
- anyone wiring a plugin or workflow to the active workspace resource

## Goals

- Make the workspace resource explicit and stable.
- Ensure the plugin chooses identity from the work location, not the agent runtime location.
- Keep the contract prompt-level, so it can be used in instructions and docs.
- Define the full decision chain for workspace UID selection.
- Make the nearest enclosing git repository win when work happens inside git.
- Use the repository remote name as the UID when git is present.
- Use the parent folder name of the project when the project is not a git repo.
- Fall back to `root` when no more specific identity exists.
- Give plugin authors a readable decision guide instead of a hidden heuristic.
- Require future PRs to keep this contract and its downstream docs up to date.

## Non-goals

- No code-enforced resolver is required for the first version.
- No deterministic config surface is required.
- No new schema or state store is required.
- No host-specific heuristics are required.
- No reliance on the agent's launch directory, container cwd, or shell cwd as a source of truth.
- No attempt is made here to support multiple competing workspace identity systems.

## Contract summary

The workspace UID must be chosen from the work location, not from where the model is running.

The full priority order is:

1. If the work is happening inside a git repository, use the repository remote name as the UID.
2. Otherwise, if the project is not a git repo, use the base directory name of the folder that holds the project.
3. Otherwise, use the literal string `root`.

## Detailed contracts

### 1. Work-location contract

The plugin must treat the place where work is happening as the source of truth.

This means:

- the model's execution environment does not define identity
- the agent's cwd does not define identity
- a remote shell, container, or delegated worker does not define identity by itself
- the active project/workspace location defines identity

If the work is being done on a specific project directory, the plugin must resolve identity from that directory's project context.

### 2. Git repository contract

When the work is inside a git repository:

- choose the nearest enclosing git repo
- use the repo's remote name as the UID
- do not skip to a higher-level directory just because the agent is launched elsewhere
- do not replace git-based identity with a configured runtime identity

The nearest enclosing repo wins because it is the repository that actually owns the work.

### 3. Non-git project contract

When the work is not inside a git repository:

- use the base directory name of the folder that holds the project
- do not derive identity from the agent process folder
- do not invent a separate configured identity unless the contract is explicitly expanded later

This keeps the identity tied to the project container, not the runtime container.

### 4. Fallback contract

If neither git nor a project-folder identity exists:

- use `root`
- treat `root` as the literal fallback string
- do not normalize it into another label

### 5. Prompt contract

The plugin should receive a prompt-level instruction that is short, explicit, and easy to repeat.

Proposed wording:

> Choose the workspace UID from the work location, not the model runtime location. If the work is inside a git repo, use the repo remote name. If it is not a git repo, use the parent folder name of the project. Otherwise use `root`.

### 6. Plugin consumption contract

A plugin that consumes this resource must:

- read the work location as the primary input
- use the contract above to derive the UID
- avoid recomputing identity from its own launch path
- preserve the chosen UID for downstream use once resolved
- surface the resolved UID consistently wherever the plugin labels workspace-scoped actions or resources

The plugin should behave as though the workspace identity is a property of the project, not of the runtime session.

### 7. Documentation maintenance contract

This guide must stay in sync with the repo's contributor docs.

Add a gate to `CONTRIBUTING.md` requiring that every PR touching workspace identity, workspace selection, or related plugin/prompt docs either:

- updates this guide, or
- explicitly confirms the guide remains accurate

The gate should also require that behavior changes affecting this contract update any downstream docs in the same PR.

## Suggested implementation shape for a plugin

This guide is intentionally not code, but a plugin built on top of it should follow a simple flow:

1. identify the active work location
2. check whether the location is inside a git repo
3. if yes, use the repo remote name as the UID
4. if no, derive the parent folder name for the project
5. if neither exists, fall back to `root`
6. pass the resolved UID through the rest of the plugin flow unchanged

## Examples

### Example A: work inside a git repo

- Work location: `/workspace/hermes-webui`
- Repo: yes
- Remote name: `origin`
- Resulting UID: `origin`

### Example B: work in a non-git project folder

- Work location: `/workspace/demo-notes`
- Repo: no
- Parent folder holding the project: `workspace`
- Resulting UID: `workspace`

### Example C: no project identity available

- Work location: unknown or not meaningful
- Repo: no
- Project folder identity: none
- Resulting UID: `root`

## Acceptance criteria

- The document reads as a guide for plugin authors, not a code patch note.
- The work-location rule is explicit.
- The nearest enclosing git repo wins.
- Git-based identity uses the repo remote name.
- Non-git identity uses the parent folder name of the project.
- The fallback is the literal string `root`.
- The doc includes the contributor-doc maintenance gate.
- The doc is broad enough to support a plugin that uses this workspace resource without inventing extra heuristics.

## Resolved decisions

1. If a git repository has no configured remote, fall back to the repository directory name.
2. If a git repository has multiple remotes, prefer `origin`.

These decisions keep the contract deterministic without adding extra user-facing configuration.

## Rollout plan

1. Land this guide as the source of truth for workspace-aware plugin behavior.
2. Add the `CONTRIBUTING.md` maintenance gate.
3. Update any downstream prompt/plugin docs to point at this contract.
4. Keep the contract aligned with implementation details if the plugin ever needs a narrower or more explicit remote-selection rule.
