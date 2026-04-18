---
name: github-monitor-pattern
category: devops
description: Pattern for monitoring GitHub repos for new issues, PRs, and releases using baseline-based diffing. Includes lessons on pagination, type safety, and incremental detection.
---

# GitHub Monitor Pattern

Monitor a GitHub repo for changes (new issues, closed issues, new PRs, merged PRs, new releases) using baseline-based diffing.

## Approach

1. **Fetch all items** via GitHub API (no `since` filter — it causes bugs)
2. **Compare against baseline** stored in JSON (key types must match)
3. **Detect state changes** by comparing current state vs baseline state
4. **Update baseline** after reporting
5. **Run via cron** for periodic checks

## Implementation

### Baseline Structure
```json
{
  "repos": {
    "owner/repo": {
      "issues": { "123": { "state": "open", "title": "..." }, ... },
      "prs": { "456": { "state": "open", "merged": false, "title": "..." }, ... },
      "tags": ["v1.0", "v1.1", ...]
    }
  },
  "last_run": "2026-04-18T02:00:00+00:00"
}
```

### Critical Bug: Type Mismatch
GitHub API returns integer issue/PR numbers. Baseline stores them as strings (JSON keys are always strings). **Always convert to string before comparison:**
```python
iid = issue["number"]          # int from API
iid_str = str(iid)             # convert to string
if iid_str not in known_ids:   # compare string to string
```

### Critical Bug: `since` Parameter
Do NOT use the `since` query parameter for incremental fetching. It returns items updated after the timestamp, which may not include all tracked items. Instead:
- Fetch ALL items every time
- Compare against baseline to detect changes
- The baseline tracks everything, so you only report what changed

### Pagination
GitHub API returns max 100 per page. Handle pagination:
```python
page = 1
while True:
    result = fetch_json(f"{url}?page={page}&per_page=100")
    if not result or len(result) < 100:
        break
    all_items.extend(result)
    page += 1
```

## Detection Logic

### Issues
- **New**: `iid` not in baseline → check state (open = new issue, closed = closed issue)
- **Closed**: in baseline, was "open", now "closed"
- **Reopened**: in baseline, was "closed", now "open"

### PRs
- **New**: `pid` not in baseline → check state (merged = merged, closed = closed, open = new)
- **Merged**: in baseline, was `merged: false`, now `merged: true`

### Releases
- **New**: tag not in baseline tags list

## Cron Setup
```bash
# Run every 6 hours, deliver to chat
cronjob create --name "repo-monitor" --schedule "every 6h" \
  --prompt "Run python3 ~/.hermes/scripts/repo-monitor.py" \
  --deliver origin
```

## Script Location
Store at `~/.hermes/scripts/<name>-monitor.py`
Baseline at `~/.hermes/data/<name>-monitor/baseline.json`

## Lessons Learned
- Always convert API integers to strings before baseline comparison
- Never use `since` parameter — fetch all, diff locally
- Pagination must check `len(result) < 100` to detect last page
- GitHub API rate limit: 60 req/hr unauthenticated, 5000/hr with token
- Use `urllib.request` with `User-Agent` header (GitHub requires it)