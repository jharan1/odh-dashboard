# Team Slack Reference

Slack usergroup IDs and channels for each dashboard scrum team. Used by the nightly E2E handler to tag the correct team in threaded Slack messages.

**Usergroup mention format**: `<!subteam^ID>` in Slack API calls (the `post_message` tool handles `@` mentions differently — use the subteam syntax in the message text).

**Important**: Slack usergroup IDs can change over time. If a mention doesn't resolve to a team name in Slack, ask the operator to verify the correct handle.

## Scrum Team Handles

| Team | Slack Usergroup ID | Mention Syntax | Working Group Channel |
|---|---|---|---|
| Monarch | `S0ABKEG0C14` | `<!subteam^S0ABKEG0C14>` | `wg-dashboard-monarch` (C07BFD5J4CB) |
| Crimson | `S09A6NJHBU1` | `<!subteam^S09A6NJHBU1>` | `wg-dashboard-crimson` (C099MEPGF43) |
| Razzmatazz | `S06FFRUSFH8` | `<!subteam^S06FFRUSFH8>` | `wg-dashboard-razzmatazz` |
| Zaffre | `S07CFUVMXBM` | `<!subteam^S07CFUVMXBM>` | `wg-dashboard-zaffre` (C069KSM8T9N) |
| Green | `S07BJDHQR2R` | `<!subteam^S07BJDHQR2R>` | `wg-dashboard-green` |
| Tangerine | `S0AG2A9KP5W` | `<!subteam^S0AG2A9KP5W>` | `wg-dashboard-tangerine` (C09PD4PF58W) |
| Purple | `S07EBN8NY1L` | `<!subteam^S07EBN8NY1L>` | `wg-dashboard-purple` (C0A9QDP09J9) |
| Onyx | `S0B5BJW6T8S` | `<!subteam^S0B5BJW6T8S>` | `wg-dashboard-onyx` |

## Other Relevant Handles

| Handle | Slack Usergroup ID | Mention Syntax | Notes |
|---|---|---|---|
| Dashboard Managers | `S07JPAFE0KY` | `<!subteam^S07JPAFE0KY>` | Management/leads group — use for escalations |
| Quality / E2E | `S08AZ980ER0` | `<!subteam^S08AZ980ER0>` | Quality/E2E testing group — secondary contact for test infrastructure issues |

## Fallback: OWNERS_ALIASES

If a Slack usergroup handle doesn't resolve or you need to identify individual contacts, check the `OWNERS_ALIASES` file at the repo root. It maps code areas to GitHub reviewer/approver groups with current team members. It is maintained by the teams themselves and stays up to date as people move.
