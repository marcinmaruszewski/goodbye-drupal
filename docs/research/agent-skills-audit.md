# Agent skills audit

Date: 2026-08-29  
Scope: Next.js/React, NestJS/Fastify/GraphQL, shadcn/ui/Tailwind/accessibility,
OpenTofu/Terraform, GCP/Cloud Run, GitHub Actions, testing, and safe parallel
Git worktrees.  
Method: inspect the repository lock file and checked-in skills, query the
skills.sh index, then read the actual upstream `SKILL.md`, supporting files,
license, repository status, and recent commit for serious candidates. Install
counts were treated only as discovery signals, never as quality evidence.

No skill was installed during this audit. The one `npx skills add ... --list`
command used for shadcn/ui only enumerated the repository's skills.

## Decision

Keep the repository-owned set small and reproducible. Adopt three existing
skills when the application skeleton is created, add one GCP skill when cloud
architecture work starts, and create project-specific skills for the seams
where generic ecosystem guidance is either incomplete or unsafe.

### Adopt with the application skeleton

| Package | Why it earns repository ownership | Important boundary |
|---|---|---|
| [`vercel-labs/agent-skills@vercel-react-best-practices`](https://skills.sh/vercel-labs/agent-skills/vercel-react-best-practices) ([upstream](https://github.com/vercel-labs/agent-skills/tree/main/skills/react-best-practices)) | First-party Vercel performance guidance; active on 2026-08-28; progressive rule files; no shell mutation instructions. | Performance guidance, not general Next.js correctness. The skill declares MIT, although GitHub does not detect a root repository license; retain the skill's license metadata when vendoring. |
| [`mcollina/skills@fastify-best-practices`](https://skills.sh/mcollina/skills/fastify-best-practices) ([upstream](https://github.com/mcollina/skills/tree/main/skills/fastify)) | Maintained in Matteo Collina's Node/Fastify skill collection; active on 2026-08-17; MIT; focused references for plugins, schemas, lifecycle, logging, security, testing, and deployment. | It teaches native Fastify. A project skill must state where Nest's adapter and GraphQL driver supersede native routes/plugins. |
| `shadcn-ui/ui@shadcn` ([upstream](https://github.com/shadcn-ui/ui/tree/main/skills/shadcn)) | The first-party shadcn/ui repository exposes this exact package/skill through `npx skills add shadcn-ui/ui --list`; active on 2026-08-26; MIT; project-aware via `components.json`; uses docs/dry-run/diff before updates and requires approval before overwrite. | Not indexed by skills.sh search on the audit date, so use the upstream identifier. CLI `add` mutates source by design; normal change authorization still applies. |

### Adopt when GCP architecture work starts

| Package | Why | Boundary |
|---|---|---|
| [`google/skills@google-cloud-solution-architecture`](https://skills.sh/google/skills/google-cloud-solution-architecture) ([upstream](https://github.com/google/skills/tree/main/skills/cloud/google-cloud-solution-architecture)) | First-party Google, Apache-2.0, active on 2026-08-28. It requires requirements discovery, approval between phases, citations to current Google documentation, and explicit permission before dry runs or live deployment. | It is deliberately heavyweight and expects Google Developer Knowledge MCP or equivalent official-doc access. Trigger it for architecture decisions, not routine edits. |

### Do not add a separate Next.js best-practices skill

The machine-global `next-best-practices` is sourced from
`vercel-labs/next-skills`, but that repository now says the skill was retired:
Next.js 16.3+ supplies version-matched bundled documentation and writes agent
rules when `next dev` runs. The old local copy can still run but no longer
receives updates. Use the framework's version-matched mechanism and keep
`vercel-react-best-practices` only for its distinct performance scope. See the
[move notice](https://github.com/vercel-labs/next-skills) and the current
[Next.js skills directory](https://github.com/vercel/next.js/tree/canary/skills).

## What the repository owns today

`skills-lock.json` pins 25 skills, all from
[`mattpocock/skills`](https://github.com/mattpocock/skills), and corresponding
directories are checked into `.agents/skills/`:

`ask-matt`, `code-review`, `codebase-design`, `diagnosing-bugs`,
`domain-modeling`, `grill-me`, `grill-with-docs`, `grilling`, `handoff`,
`implement`, `improve-codebase-architecture`, `prototype`, `research`,
`resolving-merge-conflicts`, `setup-matt-pocock-skills`, `tdd`, `teach`,
`to-questionnaire`, `to-spec`, `to-tickets`, `triage`, `wait-what`,
`wayfinder`, `wizard`, and `writing-for-agents`.

These cover planning, teaching, product/domain discussion, implementation
process, diagnosis, TDD, review, and issue workflow. They do **not** encode the
chosen application stack, cloud platform, or parallel-worktree operating model.

The current machine also exposes `next-best-practices` and
`vercel-react-best-practices` globally. Global availability is not repository
ownership: another contributor or CI agent cannot rely on it, its version is
not pinned by this repository, and in the Next.js case it is already retired
upstream. If a skill affects expected repository behavior, pin it in
`skills-lock.json` and make it available through `.agents/skills/` rather than
depending on a developer's home directory.

## Domain audit

### Next.js and React

- **Adopt:** `vercel-labs/agent-skills@vercel-react-best-practices`. Its trigger
  is broad but accurate for React/Next performance work, its 70 rules are
  progressively disclosed, and it has no deployment or destructive workflow.
- **Reject as stale:** the globally installed
  `vercel-labs/next-skills@next-best-practices`. Upstream explicitly moved away
  from a standalone skill to version-matched Next.js docs.
- **Defer:** the workflow-specific skills currently under
  [`vercel/next.js`](https://github.com/vercel/next.js/tree/canary/skills), such
  as Cache Components adoption/optimization, until the application actually
  chooses those features.

### NestJS, Fastify, and GraphQL

- **Adopt:** `mcollina/skills@fastify-best-practices` for Fastify-native
  behavior. The upstream is active, MIT, narrowly triggered, and separates
  detailed topics into rule files.
- **Do not adopt yet:**
  [`giuseppe-trisciuoglio/developer-kit@nestjs-best-practices`](https://skills.sh/giuseppe-trisciuoglio/developer-kit/nestjs-best-practices)
  ([upstream](https://github.com/giuseppe-trisciuoglio/developer-kit/tree/main/plugins/developer-kit-typescript/skills/nestjs-best-practices))
  is active and MIT, but it hard-selects Drizzle, `class-validator`, and Jest
  conventions while containing no Fastify or GraphQL guidance. It is a useful
  generic Nest reference, not this stack's contract.
- **Representative reject:**
  [`ejirocodes/agent-skills@nestjs-best-practices`](https://skills.sh/ejirocodes/agent-skills/nestjs-best-practices)
  ([upstream](https://github.com/ejirocodes/agent-skills/tree/main/nestjs/skills/nestjs-best-practices))
  is MIT and recent enough, but its quick reference mixes Express-specific
  response handling, treats `forwardRef` as a normal fix, only mentions Fastify
  as a breaking-change bullet, and assumes Apollo in its GraphQL material.
- **Gap:** no audited candidate owns the exact NestJS + Fastify adapter + chosen
  GraphQL driver/schema model. Generic GraphQL results were third-party and
  detached from Nest/Fastify. This seam needs a project skill after the driver,
  schema ownership, validation, persistence, and error model are decided.

### shadcn/ui, Tailwind, and accessibility

- **Adopt:** first-party `shadcn-ui/ui@shadcn`. It reads the actual project
  configuration and current component docs, distinguishes Radix/Base UI,
  enforces semantic tokens and accessible composition, previews updates, and
  protects local changes from blind overwrite.
- **Reject:**
  [`secondsky/claude-skills@tailwind-v4-shadcn`](https://skills.sh/secondsky/claude-skills/tailwind-v4-shadcn)
  ([upstream](https://github.com/secondsky/claude-skills/tree/main/plugins/tailwind-v4-shadcn/skills/tailwind-v4-shadcn))
  is a Vite-specific setup despite this repository choosing Next.js. It mandates
  Vite plugins and file deletion, carries dated dependency/version assumptions,
  and has almost no accessibility coverage.
- **Defer:**
  [`mattbx/shadcn-skills@shadcn-component-review`](https://skills.sh/mattbx/shadcn-skills/shadcn-component-review)
  and
  [`mattbx/shadcn-skills@shadcn-component-discovery`](https://skills.sh/mattbx/shadcn-skills/shadcn-component-discovery)
  are thoughtful MIT review/discovery tools,
  but their proactive triggers cover nearly every UI change and discovery can
  introduce unaudited community registries. Start with the first-party skill;
  reconsider these only if a measured review/discovery gap appears.
- **No separate Tailwind skill:** first-party shadcn project context already
  handles the relevant Tailwind version, semantic tokens, spacing, and theming.
  Add project accessibility gates (lint plus browser checks) rather than another
  overlapping style prompt.

### OpenTofu and Terraform

No existing candidate should be repository-owned unchanged.

- [`hashicorp/agent-skills@terraform-style-guide`](https://skills.sh/hashicorp/agent-skills/terraform-style-guide)
  ([upstream](https://github.com/hashicorp/agent-skills/tree/main/plugins/terraform/skills/terraform-style-guide))
  is the most authoritative style source (MPL-2.0, active on 2026-08-26), but it
  hardcodes Terraform CLI/version examples (`>= 1.14`) and does not claim
  OpenTofu compatibility. Its companion `terraform-test` includes apply-mode
  tests that create real resources.
- [`terramate-io/agent-skills@terraform-best-practices`](https://skills.sh/terramate-io/agent-skills/terraform-best-practices)
  ([upstream](https://github.com/terramate-io/agent-skills/tree/main/skills/terraform-best-practices))
  names OpenTofu and is MIT, but the detailed rules are Terraform/AWS-heavy and
  contain `apply -auto-approve`, `destroy`, `force-unlock`, and `rm -rf
  .terraform` recipes without a project authorization boundary.
- [`josiahsiegel/claude-plugin-marketplace@opentofu-guide`](https://skills.sh/josiahsiegel/claude-plugin-marketplace/opentofu-guide)
  ([upstream](https://github.com/JosiahSiegel/claude-plugin-marketplace/tree/main/plugins/terraform-master/skills/opentofu-guide))
  is MIT but internally anchored to OpenTofu 1.8-era claims while its own
  references mention newer releases. It includes curl-to-shell/sudo, state
  migration, plaintext backup, direct apply, and `-auto-approve` workflows and
  overstates compatibility as universal.

Create a narrow, project-owned OpenTofu/GCP skill grounded in current OpenTofu
and Google provider documentation. It should allow formatting, validation,
linting, tests, and plan generation by default; require explicit authority for
apply/destroy/import/state/IAM/API-enablement operations; mandate GitHub OIDC/WIF
rather than long-lived keys; and encode the selected GCS state, Cloud Run,
Secret Manager, migration job, naming, labels, budgets, and region conventions.

### GCP and Cloud Run

- **Adopt later:** `google/skills@google-cloud-solution-architecture`, described
  above, for explicit architecture work.
- **Do not install for routine use:**
  [`google/skills@cloud-run-basics`](https://skills.sh/google/skills/cloud-run-basics)
  ([upstream](https://github.com/google/skills/tree/main/skills/cloud/cloud-run-basics))
  is authoritative, Apache-2.0, and fresh, but its broad trigger says it
  “manages” Cloud Run and its main path directly enables APIs, changes IAM,
  deploys with `--allow-unauthenticated`, creates/executes jobs, and deploys
  worker pools without an explicit approval gate. Use its official references
  while authoring the safe project skill, not its imperative workflow unchanged.

### GitHub Actions

- **Defer:**
  [`github/awesome-copilot@github-actions-efficiency`](https://skills.sh/github/awesome-copilot/github-actions-efficiency)
  ([upstream](https://github.com/github/awesome-copilot/tree/main/skills/github-actions-efficiency))
  is MIT and current, but GitHub describes the repository as community
  contributed. The skill optimizes runner cost/latency rather than establishing
  correctness, supply-chain security, permissions, OIDC, or deployment
  promotion. It also recommends validating through a live test push without an
  explicit approval step. Revisit once workflows have measurable history.
- **Gap:** the OpenTofu/GCP project skill should own WIF and deployment safety;
  the project testing skill should own required checks. A tiny CI conventions
  reference can then encode action SHA/version policy, least-privilege
  `permissions`, concurrency, caching, and fork/secret boundaries.

### Testing

- **Keep:** the repository already owns `tdd`, which governs the test-first
  feedback loop and discourages mock-heavy implementation tests.
- **Reject as generic:**
  [`secondsky/claude-skills@vitest-testing`](https://skills.sh/secondsky/claude-skills/vitest-testing)
  is MIT but recommends Bun, global APIs, fixed 80% coverage, and one-test-file
  per source file independent of the repository's eventual package manager and
  architecture.
- **Defer Playwright add-ons:** the audited third-party Playwright E2E and a11y
  skills were broad recipe collections rather than first-party test strategy.
  Microsoft's indexed `microsoft/playwright@playwright-test-results` is for
  interpreting results, not authoring this application's tests.
- **Gap:** after test tools are chosen, write a project skill defining Nest
  unit/integration boundaries, GraphQL contract tests, Next component tests,
  Playwright user journeys, accessibility checks, database isolation, and which
  suites run locally versus in CI. Compose it with the existing `tdd` process.

### Safe parallel Git worktrees

No ecosystem candidate matches this repository's safety and tracker workflow.

- [`thebeardedbearsas/claude-craft@parallel-worktrees`](https://skills.sh/thebeardedbearsas/claude-craft/parallel-worktrees)
  is MIT but only 38 lines, delegates to a relative rule outside the skill, and
  provides no dirty-tree, branch, secret, port, service, claim, or recovery
  guardrails.
- [`davidondrej/skills@git-worktree`](https://skills.sh/davidondrej/skills/git-worktree)
  is MIT and detailed, but disabled from automatic invocation, Cursor-specific,
  instructs agents to copy secret files, assumes only `main` is pushed, and
  includes removal/branch-deletion commands that do not fit issue branches and
  research handoffs.

Create a project-owned worktree skill plus a small bootstrap script. It should:

1. map one claimed issue to one branch, worktree, and agent;
2. verify the exact repository/common-dir, base revision, branch uniqueness,
   clean target, and explicit destination before creating anything;
3. never copy secrets by default, and document an allowlisted local mechanism;
4. give each worktree deterministic Compose project names, ports, caches, and
   test databases;
5. keep installs/build output local to the worktree while reusing only safe
   shared caches;
6. require commit plus pushed context pointer before handoff, and review before
   integration;
7. remove a worktree/branch only after verifying it is clean and its commits
   are reachable from an approved integration point; and
8. serialize integration and infrastructure apply operations even when coding
   work is parallel.

## Minimal rollout

1. Pin and vendor the React, Fastify, and first-party shadcn skills when their
   respective packages appear in the repository. Do not rely on global copies.
2. Let Next.js generate version-matched agent documentation; remove any future
   temptation to pin the retired Next skill.
3. Create the safe worktree skill before enabling routine parallel agent work.
4. Create the backend-conventions skill immediately after the GraphQL driver,
   schema model, validation, database, and error boundaries are decided.
5. Add Google's solution-architecture skill and create the safe OpenTofu/GCP
   skill when GCP design begins.
6. Create the test-strategy skill once test runners and package manager are
   selected; evaluate GitHub Actions efficiency only after CI has real runs.

This sequence keeps external authority close to its source while reserving
project skills for decisions that are unique to this repository and for
operations where an over-broad generic skill could mutate Git, cloud, state, or
credentials.
