---
name: nightly-e2e-triage
description: "Analyze nightly E2E test failures from Jenkins reports, map failing tests to responsible scrum teams, and post threaded Slack summaries tagging the appropriate teams. Guides the operator through the Jenkins agentic-analysis job and automates the Slack notification workflow."
argument-hint: "<Slack thread URL or message timestamp of the nightly report>"
---

# Nightly E2E Triage

End-to-end workflow for triaging nightly E2E test failures. Guides the operator through running the Jenkins agentic-analysis job, parses the results, maps each failing test to the responsible scrum team using the test directory structure and area-to-team mapping, and posts threaded Slack messages in `#team-openshift-ai-dashboard` tagging the relevant teams.

## Arguments

`$ARGUMENTS` -- one of:

- A Slack message URL (e.g., `https://redhat-internal.slack.com/archives/C05SMJ09DD2/p1785322975383129`)
- A Slack thread timestamp (e.g., `1785322975.383129`)
- Empty -- print usage and stop

If no arguments are provided, print:
```
Usage: /nightly-e2e-triage <slack-thread-url-or-timestamp>

Examples:
  /nightly-e2e-triage https://redhat-internal.slack.com/archives/C05SMJ09DD2/p1785322975383129
  /nightly-e2e-triage 1785322975.383129

Provide the Slack message URL or timestamp of the nightly E2E summary report
posted to #team-openshift-ai-dashboard.
```

## Prerequisites

- **Slack MCP** (`slack-rh`) -- connected and authenticated. This skill uses the `slack-rh` MCP server for reading threads and posting messages. If the MCP is not connected, stop immediately with the error in the Error Handling section below.
- **Jenkins report** -- the operator must be able to run the `dashboard-agentic-analysis` Jenkins job (VPN + credentials required) and supply the HTML report when prompted

**IMPORTANT — No Slack messages without explicit approval.** This skill MUST NOT post any Slack messages (via `post_message` or any other Slack write tool) until the user has explicitly approved the content in Step 7. Always present the full analysis first and wait for a clear "yes" / "Y" / "post" confirmation. If the user says "skip", "no", or "edit", do NOT post. This is a hard rule — violating it sends unsanctioned messages to a team-wide channel.

## Constants

| Constant | Value |
|---|---|
| Dashboard Slack channel ID | `C05SMJ09DD2` |
| Dashboard Slack channel name | `#team-openshift-ai-dashboard` |
| E2E test base path | `packages/cypress/cypress/tests/e2e/` |
| Anthony Coughlin Slack handle | `@Anthony Coughlin` |

## Execution

### Step 0: Verify prerequisites

Before doing anything else, verify the `slack-rh` MCP is available by calling `whoami`. If the call fails or the MCP is not connected, stop immediately:
```
Slack MCP (slack-rh) is not connected. This skill requires it for reading
threads and posting messages.

1. Check if Podman is running — slack-rh runs in a container and needs Podman:
   podman machine info
   If not running: podman machine start
2. Run /mcp in Claude Code to check your MCP server connections and reconnect.
```

### Step 1: Parse arguments and read the nightly report message

Extract the thread timestamp from the argument:
- If a full Slack URL: extract the timestamp from the `p` parameter (e.g., `p1785322975383129` -> `1785322975.383129` -- insert `.` before last 6 digits)
- If a bare timestamp: use as-is

Read the parent message using `get_thread` with channel `C05SMJ09DD2` and the extracted timestamp.

**Verify** the parent message looks like a nightly E2E report. It should contain patterns like:
- Build number and link (e.g., `Build ... #NNNN`)
- Failure count (e.g., `FAILURE (30 tests)`)
- Cluster and Git info
- Link to Jenkins job

If the message doesn't look like a nightly report, warn the user and ask to confirm.

Extract from the parent message:
- **Build number** and Jenkins URL
- **Failure count**
- **Branch/version** info
- **Cluster** info

### Step 2: Check for existing analysis in the thread

Read all replies in the thread. Look for:
- Links to `dashboard-agentic-analysis` Jenkins job runs
- Analysis messages already posted (test names, error messages, root cause analysis)
- Team tags already applied

If analysis is already present in the thread, **ask the user** whether to:
1. Use the existing analysis (skip to Step 4)
2. Run a fresh analysis

### Step 3: Request Jenkins agentic-analysis report

If no analysis exists or the user wants a fresh one:

Tell the user:
```
I need the Jenkins agentic-analysis report for this build. Please:

1. Go to: https://jenkins-csb-rhods-opendatascience.dno.corp.redhat.com/job/components/job/dashboard/job/dashboard-agentic-analysis/
2. Run a new build with the build number from the nightly report: <BUILD_NUMBER>
3. When the job completes, download the "TFA Analysis" HTML report
4. Share the file path or paste the report content here

Note: This Jenkins instance requires VPN access and authentication.
```

Wait for the user to provide the report. Accept:
- A local file path (read it with the Read tool)
- Pasted HTML/text content
- A statement that they want to use the existing thread analysis instead

### Step 4: Extract failing tests

From the Jenkins agentic-analysis report (or existing thread analysis), extract each failing test:

For each failure, capture:
- **Test file name** (e.g., `testWorkbenchCreation.cy.ts`)
- **Test spec path** if available (e.g., `dataScienceProjects/workbenches/testWorkbenchCreation.cy.ts`)
- **Error message** or summary
- **Root cause analysis** if provided
- **Video link** if available
- **Whether the failure is a known blocker** (cross-referenced with existing thread discussion)

### Step 5: Map each failing test to a scrum team

For each failing test, determine the responsible scrum team using this procedure:

#### 5a: Resolve the test file path

Search for the test file in the E2E test directory:

```bash
find packages/cypress/cypress/tests/e2e/ -name "<test-file-name>" -type f
```

The directory structure under `e2e/` maps to feature areas.

#### 5b: Map test directory to dashboard area

Use this test-directory-to-area mapping:

| E2E Test Directory | Dashboard Area |
|---|---|
| `agentOps/` | `dashboard-area-genai` |
| `agentsCatalog/` | `dashboard-area-genai` |
| `applications/` | `dashboard-area-applications` |
| `automl/` | `dashboard-area-automl` |
| `autorag/` | `dashboard-area-autorag` |
| `dashboardNavigation/` | `dashboard-area-infrastructure` |
| `dataScienceProjects/workbenches/` | `dashboard-area-workbenches` |
| `dataScienceProjects/clusterStorage/` | `dashboard-area-cluster-storage` |
| `dataScienceProjects/connections/` | `dashboard-area-connection-types` |
| `dataScienceProjects/models/` | `dashboard-area-model-serving` |
| `dataScienceProjects/` (other, including standalone .cy.ts files) | `dashboard-area-projects` |
| `distributedWorkloadMetrics/` | `dashboard-area-distributed-workloads` |
| `eval-hub/` | `dashboard-area-genai` |
| `featureStore/` | `dashboard-area-feast` |
| `gen-ai/` | `dashboard-area-genai` |
| `gpuaas/` | `dashboard-area-hardware-profiles` |
| `learningResources/` | `dashboard-area-home` |
| `mcpCatalog/` | `dashboard-area-mcp` |
| `mlflowExperiments/` | `dashboard-area-observability` |
| `modelCatalog/` | `dashboard-area-model-catalog` |
| `modelRegistry/` | `dashboard-area-model-registry` |
| `modelsAsAService/` | `dashboard-area-maas` |
| `modelTraining/` | Tangerine (direct — model training: RayJob/TrainJob) |
| `nim/` | `dashboard-area-model-serving` |
| `Pipelines/` | `dashboard-area-pipelines` |
| `promptManagement/` | `dashboard-area-genai` |
| `settings/` | `dashboard-area-cluster-settings` |
| `storageClasses/` | `dashboard-area-storage-classes` |

**Longest prefix wins**: `dataScienceProjects/workbenches/` matches `workbenches` (Razzmatazz), not `projects` (Monarch).

#### 5c: Map area to scrum team

Use the Area-to-Scrum Default Mapping from `jira-project-reference.md`:

| Dashboard Area | Default Team |
|---|---|
| `dashboard-area-applications` | Monarch |
| `dashboard-area-infrastructure` | Monarch |
| `dashboard-area-home` | Monarch |
| `dashboard-area-projects` | Monarch |
| `dashboard-area-cluster-settings` | Monarch |
| `dashboard-area-observability` | Monarch |
| `dashboard-area-workbenches` | Razzmatazz |
| `dashboard-area-pipelines` | Razzmatazz |
| `dashboard-area-storage-classes` | Razzmatazz |
| `dashboard-area-hardware-profiles` | Razzmatazz |
| `dashboard-area-model-serving` | Zaffre |
| `dashboard-area-connection-types` | Zaffre |
| `dashboard-area-maas` | Onyx |
| `dashboard-area-genai` | Crimson |
| `dashboard-area-model-catalog` | Green |
| `dashboard-area-model-registry` | Green |
| `dashboard-area-distributed-workloads` | Green |
| `dashboard-area-mcp` | Green |
| `dashboard-area-feast` | Tangerine |
| `dashboard-area-automl` | Purple |
| `dashboard-area-autorag` | Purple |

#### 5d: Resolve Slack contact for the team

Use the team Slack reference from [`team-slack-reference.md`](team-slack-reference.md).

If no Slack handle can be resolved for a team, flag it:
```
WARNING: Could not find Slack handle for team <TEAM>. You may need to manually
tag the right people for: <test-file-name>
```

### Step 6: Group failures by team

Group all failing tests by their resolved scrum team. For each team, compile:
- List of failing test files
- Brief error summary for each
- Video links where available
- Root cause hypothesis if available from the analysis

Also create an "unresolved" group for tests that couldn't be mapped to a team.

### Step 7: Present the analysis for review

Before posting to Slack, present the full analysis to the user:

```
## Nightly E2E Analysis — Build #<NUMBER>

### <TEAM_NAME> (<N> failures)
- `<testFile.cy.ts>` — <brief error summary>
- `<testFile2.cy.ts>` — <brief error summary>
Slack contact: <handle/channel>

### <TEAM_NAME_2> (<N> failures)
...

### Unresolved (<N> failures)
- `<testFile.cy.ts>` — <brief error summary>
  WARNING: Could not determine responsible team

---
Ready to post to #team-openshift-ai-dashboard thread?
[Y] Post all  |  [E] Edit assignments  |  [S] Skip posting
```

Wait for the user to confirm or edit team assignments before posting.

### Step 8: Post Slack messages

Post threaded replies to the original nightly report message in `#team-openshift-ai-dashboard` (channel `C05SMJ09DD2`).

**Post one message per team**, threaded on the original report message. Format:

```
<TEAM_SLACK_HANDLE> — <N> failing test(s) in tonight's nightly run:

• `<testFileName.cy.ts>` — <brief error summary>
  Video: <link if available>

• `<testFileName2.cy.ts>` — <brief error summary>
  Video: <link if available>

<Root cause note if available from the analysis>

cc @Anthony Coughlin
```

**Rules for posting:**
- Always thread on the original report message (use `thread_ts`)
- Always cc Anthony Coughlin in each message
- Keep messages concise — one message per team, not per test
- Include video links when available from the Jenkins report
- If root cause analysis is available, include a brief summary
- If a test's failure is confirmed as a known blocker already discussed in the thread, note it as `(known blocker — see above)`

**For unresolved tests**, post a single message:

```
The following test failures could not be automatically mapped to a team:

• `<testFileName.cy.ts>` — <brief error summary>

Manual assignment needed.

cc @Anthony Coughlin
```

### Step 9: Summary

After posting, provide a summary:

```
## Nightly E2E Triage — Complete

Posted <N> threaded messages for Build #<NUMBER>:
- <TEAM>: <N> failures (posted)
- <TEAM>: <N> failures (posted)
- Unresolved: <N> failures (posted, needs manual assignment)

Thread: <link to original Slack message>
```

## Error Handling

### Slack MCP not available
```
Slack MCP (slack-rh) is not connected. This skill requires it for reading
threads and posting messages.

1. Check if Podman is running — slack-rh runs in a container and needs Podman:
   podman machine info
   If not running: podman machine start
2. Run /mcp in Claude Code to check your MCP server connections and reconnect.
```

### Thread not found
```
Could not find a message at timestamp <TS> in #team-openshift-ai-dashboard.
Please verify the Slack URL or timestamp is correct.
```

### Test file not found in repo
If a test file from the report cannot be found in the repository:
```
WARNING: Test file '<name>' not found in packages/cypress/cypress/tests/e2e/.
It may have been renamed, moved, or removed. Marking as unresolved.
```

### No failures found
If the report shows zero failures:
```
No test failures found in this build. Nothing to post.
```
