# GitHub Copilot → Azure DevOps → working calculator

A presenter-ready demo kit: turn requirements into Azure DevOps Boards work items, then use those work items as the bounded specification for GitHub Copilot implementation.

**This kit authors instructions, not a running app.** No Azure DevOps account was connected, no work items were created, and no application was implemented or tested while preparing it. The two real repository agent skills live in their own `SKILL.md` files. They require tools exposed by your Copilot client; a skill does not install tools or grant permissions.

## Start here

1. Extract the ZIP into a fresh Git repository root, or merge its contents into an existing local checkout. Preserve the hidden `.github/` and `.vscode/` directories. The archive has no enclosing folder.
2. Read [calculator requirements](docs/calculator-requirements.md).
3. Complete the setup checklist below.
4. Open [copy/paste demo prompts](docs/demo-prompts.md) and run them in order in **VS Code, GitHub Copilot Chat, Agent mode**.
5. Inspect [the integrated skill report](docs/skill-report.html) for authoring checks and the limits of the offline smoke tests. It is not a live Azure DevOps certification.

## Intended repository layout

```text
.
├── README.md
├── .github/
│   └── skills/
│       ├── ado-create-work-items/
│       │   └── SKILL.md
│       └── ado-implement-work-item/
│           └── SKILL.md
├── .vscode/
│   └── mcp.json
└── docs/
    ├── calculator-requirements.md
    ├── demo-prompts.md
    ├── work-item-map.json
    └── skill-report.html
```

Application files such as `package.json`, `src/`, and tests are created later by Copilot during the demo, not supplied by this kit. Copy the repository skills into this exact layout; do not flatten or rename either `SKILL.md`.

## Chosen demo defaults

These are choices for a compact demo, **not facts about your organization**:

- VS Code + GitHub Copilot Agent mode; Azure DevOps Boards for planning; local Git checkout for coding.
- New app: React + TypeScript + Vite, with Vitest and React Testing Library. Prefer existing repository language, package manager, and test conventions if there is already an app.
- V1: simple calculator. V2: scientific extension of the same engine and UI, with V1 regressions preserved.
- Two version Feature parents and six implementable stories: three for each version. Tasks are optional, not extra backlog noise.
- No backend, authentication, database, paid service, or cloud deployment.
- Stable demo key: `copilot-calculator-demo`. Keep this key on repeat runs; choose another key deliberately for an independent demo.

## Architecture to explain aloud

```text
Calculator requirements (version + stable requirement IDs + acceptance criteria)
    → ado-create-work-items skill
    → Azure DevOps MCP: metadata/query → approved create/link → read-back
    → Azure DevOps Boards: version parent → implementable stories
    → docs/work-item-map.json: actual returned IDs and URLs
    → ado-implement-work-item skill: exact selected item + acceptance criteria
    → Copilot file/terminal tools: isolated calculation engine + UI + tests
    → observed validation results and human review
    → optional, separately authorized Azure DevOps comment/state transition
```

The Azure DevOps MCP server supplies work-item context and Boards actions. Copilot's repository and terminal tools do the programming. Neither the MCP server nor a `SKILL.md` is a magic code-generation API.

## Setup checklist

- [ ] A local Git repository is open at its root in VS Code, with a clean or clearly understood working tree. Use a disposable demo branch; do not overwrite unrelated work.
- [ ] GitHub Copilot is available to your account and VS Code offers Agent mode and repository agent skills. Verify that both skill descriptions are visible/discoverable in your client. Discovery is based on descriptions and prompts; filename placement alone is not proof it triggered.
- [ ] You have an Azure DevOps organization and project, project membership, and permission to query, create, edit, and link the intended work items.
- [ ] For the primary remote MCP setup, the organization is Microsoft Entra-backed. Standalone Microsoft Account-backed organizations are not supported by the documented remote server.
- [ ] Replace `<ADO_ORGANIZATION>` in [.vscode/mcp.json](.vscode/mcp.json) with the organization slug, not a project name or full URL. Replace `<ADO_PROJECT>` in prompts with your actual project name.
- [ ] Start the configured MCP server in VS Code and complete authentication through the client. Do not put tokens, passwords, client secrets, or credentials in the config, repository, prompts, or work items.
- [ ] Check actual exposed tool names and schemas, project process, work-item types, fields, link relations, and allowed state transitions using the preflight prompt. Have a prepared project selected before the live demo.
- [ ] Permit read tools during preflight. Approve creation/link tools only when running the explicit create prompt. Preview-only prompts never authorize writes.
- [ ] Node.js and your repository's package manager are ready for the later app build; React/Vite setup and package installation happen only during implementation with your consent. The kit itself installed nothing.
- [ ] Rehearse authentication, item creation, and test commands before presenting. Network, permission, generated-code, and package-install latency are not predictable.

### Remote MCP: primary configuration

The sample uses the remote HTTP endpoint and the `wit` toolset:

```json
{
  "servers": {
    "ado-remote-mcp": {
      "type": "http",
      "url": "https://mcp.dev.azure.com/<ADO_ORGANIZATION>",
      "headers": { "X-MCP-Toolsets": "wit" }
    }
  },
  "inputs": []
}
```

`wit` is the remote work-item toolset; core tools remain available unless separately filtered. This is intentionally **not read-only**, since the approved creation part of the demo needs writes. Restrict permissions and approve tools in the client instead of assuming the skill grants access.

### Local MCP: documented fallback, not a second simultaneous server

If you need the supported local path, replace the remote config with this alternative. Local prerequisites include **Node.js 20+**. This command uses the current unpinned package; review the package/version your demo actually uses. Do not select an old version merely to preserve obsolete tool names.

```json
{
  "servers": {
    "ado-local-mcp": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@azure-devops/mcp", "<ADO_ORGANIZATION>", "-d", "core", "work", "work-items"]
    }
  },
  "inputs": []
}
```

Remote **toolsets** (`wit`) and local **domains** (`core`, `work`, `work-items`) are different configuration mechanisms. Do not copy one into the other. The local process starts/downloads the package when you start it; authoring this kit did not run it. Follow the official local authentication instructions and use the client sign-in flow, never embed secrets. Local fallback is not a promise of access when organization policy or permissions deny it.

### Current documented tool examples — always discover at runtime

Microsoft consolidated the tool surface; earlier tutorials may show names that are no longer current. The current local TOOLSET documents these families:

| Capability | Documented tool family | Documented actions |
|---|---|---|
| Read items / comments / type | `wit_work_item` | `get`, `get_batch`, `list_comments`, `get_type` |
| Create / edit / add child | `wit_work_item_write` | `create`, `update`, `update_batch`, `add_child` |
| Query | `wit_query` | `wiql`, `get`, `get_results` |
| Link items or artifacts | `wit_work_item_link_write` | `link`, `unlink`, `link_to_pull_request`, `add_artifact_link` |
| Comment | `wit_work_item_comment_write` | `add`, `update` |
| Backlogs | `wit_backlog` | `list`, `list_work_items`, `reorder` |

The local TOOLSET also documents `mcp_ado_core_list_projects` and `mcp_ado_core_list_project_teams`; remote documentation shows `core_list_projects`. These are examples, **not universal callable names**. Client prefixes, actions, and input parameters must be taken from the actual exposed descriptions and schemas. No skill in this kit assumes a JSON payload or falls back to invented REST requests when a capability is unavailable.

## Backlog and process mapping

Use metadata from your actual project before deciding types or fields:

| Process, if verified | Implementable item | Version parent |
|---|---|---|
| Agile | User Story | Feature, if supported |
| Scrum | Product Backlog Item | Feature, if supported |
| CMMI | Requirement | Feature, if supported |
| Basic | Issue | Supported Issue parent if hierarchy is allowed; otherwise standalone Issues |

Basic does not have Feature by default. Never create an unsupported Feature just to match the picture. For custom processes, use verified types and relations. Acceptance criteria go in a supported acceptance-criteria field, or in a clearly headed section of the supported description field. Preserve all Given/When/Then criteria in either case.

The traceability map starts deliberately empty. IDs and URLs appear only after successful MCP responses and read-back. Keep it with the repository, do not invent a completed map before the demo. Where item queries cannot discover pre-existing matches, creation is blocked unless the user supplies verified metadata and trustworthy existing-item evidence; the agent must not guess and risk duplicates.

## Suggested rehearsal and presentation flow

**Rehearsal:** complete setup and all preflight/preview steps. Test the entire flow on a disposable project or clearly tagged demo backlog, with actual approvals. Keep actual item IDs handy. Make a screenshot or local backup only of non-sensitive demo material. Never present precreated items as newly created if you use a fallback.

**Short live flow (roughly 12–20 minutes; generation may take longer):**

1. **1–2 min:** show requirements and the two `SKILL.md` descriptions. Explain read tools versus write tools.
2. **2–3 min:** preflight, then V1 preview. Show version parent, three stories, acceptance criteria, and repeatability keys before approving anything.
3. **2–3 min:** create V1, then show actual Boards items and the populated map.
4. **4–8 min:** implement the first actual V1 story, inspect tests and diff, then continue the remaining V1 stories if time allows. The first story is engine-only; a working UI requires the other V1 stories.
5. **2–3 min:** show V2 preview/create and a bounded V2 implementation request. Run real V1 regression checks before claiming V2 success.
6. **1 min:** repeat the V1 create prompt to demonstrate reuse, or show the separate optional writeback approval.

For a reliably short talk, implement just one story live and explain that the remaining prompts complete the app. Do not call a partially implemented V1 a finished calculator. For a complete end-to-end rehearsal, use all prompts and allow additional time.

## Definition of done for the demo application

- Selected acceptance criteria have implementation and test coverage, with requirement IDs visible in test names or a test map.
- Engine tests, UI tests, type checks, lint (if configured), and production build have actually run, or are explicitly reported as not run with reasons.
- Keyboard behavior and accessibility checks include manual inspection; automated tests alone are not a blanket accessibility guarantee.
- V2 changes preserve the V1 suite. Unselected work items remain out of scope.
- Diff is reviewed. No automatic commit, push, pull request, Azure DevOps state change, or completion claim.

## Offline validation limits

The delivered skill report covers static authoring checks and isolated smoke tests of routing, boundary handling, and safe blocking when inputs/tools are unavailable. It does **not** prove successful authentication, project metadata discovery, work-item CRUD/linking, Copilot client discovery, code generation, or calculator execution in your environment. The numeric writing-quality rubric is advisory and not a GitHub/Microsoft certification. Rehearse the live path with your actual project and client.

## Official references

- [GitHub: Add repository agent skills](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/customize-cloud-agent/add-skills) — placement, frontmatter, skill discovery, and support.
- [Microsoft Azure DevOps MCP repository](https://github.com/microsoft/azure-devops-mcp) — setup, remote recommendation, local fallback, and tool consolidation notes.
- [Microsoft Learn: Remote Azure DevOps MCP server](https://learn.microsoft.com/en-us/azure/devops/mcp-server/remote-mcp-server?view=azure-devops) — organization/authentication requirements and remote toolsets.
- [Microsoft: current local MCP TOOLSET](https://github.com/microsoft/azure-devops-mcp/blob/main/docs/TOOLSET.md) — current tool families and actions; inspect your live schemas because the surface evolves.
