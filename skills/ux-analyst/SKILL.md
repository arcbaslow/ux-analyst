---
name: ux-analyst
description: "UI/UX audit of a site from Microsoft Clarity: ranks pages by friction with the Clarity MCP, reads click maps, scroll maps, rage and dead clicks and Copilot heatmap summaries in the Clarity UI through a browser tool, compares them with the live page, and writes per-page findings (element, problem, evidence, fix, priority). Use for Clarity, heatmap, click map, scroll map, UX audit, rage or dead clicks, where users get stuck, and before redesigning a page that already has Clarity data."
argument-hint: "[site or Clarity project] [optional: pages or funnel step]"
---

Three to five pages per run, every finding backed by Clarity numbers. Clarity has no heatmap API: the MCP answers dashboard questions, click and scroll maps are read in the Clarity UI.

## Before the first call
- Project ID: from the user, the project's notes or `list-projects`. Pass it as `projectId` on every MCP call; never call `select-project`, it changes the default for the whole sign-in. Project rules in the repo's `CLAUDE.md` or `AGENTS.md` override this skill.
- PII in URLs: if the site puts personal data in paths or query strings (emails, phones, order or user IDs), Clarity already stores it. On such a site: filter recordings and heatmaps by path prefix or path regex only, never list raw URLs, never open recordings, read only the side-panel text and the totals under the map, never the snapshot's text, and state in the report that the data set contains those URLs.
- Check the project's masking mode first. In Balanced (the default) Clarity hides digits but not letters, so click-list labels and the snapshot can show customer names and delivery addresses. Name such elements by selector, never copy their text, and report it as a finding.
- Tools: the official `microsoft-clarity` MCP (OAuth sign-in) and a browser tool that can open the Clarity UI with the user's sign-in (Claude in Chrome or similar). Clarity MCP not installed: point the user to its setup (https://github.com/microsoft/clarity-mcp-server) and stop. Installed during this session but its tools are missing: ask for a new session. No access to the project: stop and say so.
- Clarity settings, smart events and the tag manager are read-only here.

## Window
One window for everything: the last 30 whole days in the site's main time zone, ending yesterday. Custom range in the Clarity UI, the same dates in each MCP query, converted to UTC for `list-session-recordings`. If a source returns a shorter span, say so in the report. When a redesign falls inside the window (step 6), read the maps for the full window and again for the days since the change.

The Clarity UI date field does not take typed text; pick dates in its calendar once, then reuse the heatmap URL it produces: `date=Custom&start=<ms>&end=<ms>`, where `start` is 00:00 of the first day and `end` is 23:59:59.999 of the last day in the site's time zone, in epoch ms. The calendar builds these in the browser's time zone; when it differs from the site's, compute the values yourself. Other parameters: `heatmapDeviceType` 0 mobile, 2 desktop, `heatmapType` 0 all taps, 1 scroll, 3 dead, 4 rage, 7 conversion, and `URL=2;6;<regex>` for a path regex. A heatmap counts at most 100 000 page views; above that, say it is a sample.

## Run
1. **Rank.** `query-analytics-dashboard`, one metric per query, dates spelled out: sessions by URL, then per page and device the counts of sessions and of sessions with a dead click, rage click and quickback on that page, and average scroll depth. Compute the rates yourself: asked for a percentage, the MCP returns different numbers for the same page from query to query. Score = business weight of the page (product card, cart, checkout, lead form over content pages) × friction rate per session, never raw counts. Pick 3-5 pages, one line on why each.
2. **Funnel judge.** The page's step conversion from the product analytics source (GA4, Amplitude, Mixpanel or similar), compared with the same step in the previous 30 days, or with pages of the same type. A page ranked bad that converts as well as its peers is reported as such. Name the unit and source in every column. No funnel source: say so and skip the column.
3. **Click map**, per page, mobile then desktop: the side-panel click list (element selector, clicks, %), top 10, then dead and rage taps from the Tap drop-down. The list reads as text; each item's full selector sits in its `title` attribute, and the page views and tap totals are under the map. Converters: the «Conversion» view gives purchases / sessions for the page and the conversion rate of sessions that tapped each page area. If it shows no purchases, the report says there is no conversion segment; if Clarity's purchase count is not reconciled with the funnel source, the report says so.
4. **Scroll map.** The panel holds a «Scroll data» table, visitors reached per 5% of depth: take the 50% and 25% reach lines from it. Place the key CTA against them by its offset on the live page divided by the page height (step 6).
5. **Copilot.** Run «Summarise heatmaps» and treat it as a hypothesis to check against the numbers, never as evidence: it can miss the largest dead-tap cluster on the page.
6. **Live page.** Open the page at 1440 px and at 375 px, compare its selectors with the panel's and take the CTA offset for step 4. The Clarity snapshot can render without CSS or catch a loading screen or modal, so trust selectors, not positions on it. A redesign inside the window invalidates the map: when the live selectors differ from the panel's, read the panel for single days to date the change.
7. **Recordings** (not on PII sites). `list-session-recordings` by path with `deadClickPresent` or `rageClickPresent`; watch one or two when the cause is unclear.
8. **Findings.** Each one gives the element (selector), what users do, why it hurts the step, the evidence (numbers, device, window), the fix and a priority from P1 to P3.

If the panel text has no click list, stop and ask. The fallback is the CSV download, which needs a yes for each file.

## Output
Write the report where the user keeps project notes; with no such place, a Markdown file `ux-audit-<date>.md` in the working directory. Write it in the user's language. The report never holds query strings, recording or player links, user or session IDs, or personal data; recording links go in chat only. Screenshots stay in the conversation. Chat gets the top three findings, one line each.

```markdown
### Clarity UX audit, <date>

| Parameter | Value |
|---|---|
| Window | <dates, time zone> |
| Clarity project | `<id>` |
| Funnel source | <GA4 / Amplitude / none, unit> |
| Conversion segment | <yes / no> |
| Recordings | <N> watched / not opened (PII in URLs) |

| Page | Clarity sessions | Dead, sessions (%) | Rage, sessions (%) | Quickback, sessions (%) | Avg scroll | Step conversion, <unit> | Comparison | Why picked |
|---|---|---|---|---|---|---|---|---|

#### <page path>
| # | Element | What happens | Evidence | Fix | Priority |
|---|---|---|---|---|---|

#### Not checked
```
