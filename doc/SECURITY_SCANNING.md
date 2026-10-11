# Security Scanning with Strix

This repository runs an AI-driven security scan (pentest) on every pull
request using [Strix](https://docs.strix.ai). The scan is defined in
`.github/workflows/security.yml`.

On each PR, Strix tests **only the files changed in that PR**, validates
findings with real proof-of-concept exploits (not static-analysis guesses),
and **fails the build** when it confirms a vulnerability — blocking insecure
code before it merges. Findings are also uploaded to the repository's
**Security → Code scanning** tab as SARIF.

---

## Required setup (one time)

The workflow needs two repository secrets. Add them in GitHub under
**Settings → Secrets and variables → Actions → New repository secret**:

| Secret        | Value                                                        |
| ------------- | ------------------------------------------------------------ |
| `STRIX_LLM`   | The model id, e.g. `openai/gpt-5.4` or `anthropic/claude-...` |
| `LLM_API_KEY` | The API key for that provider                                |

Without both secrets the job fails at the scan step. Do not commit these
values to the repository — they belong only in GitHub Actions secrets.

See the list of supported providers: https://docs.strix.ai/llm-providers/overview

---

## How the workflow works

Key flags in `.github/workflows/security.yml`:

- `--scope-mode diff --diff-base "origin/${{ github.base_ref }}"` — scans only
  the diff against the PR's **actual base branch** (this repo's default is
  `9.0`, not `main`). `fetch-depth: 0` on checkout is required so git can
  resolve that diff.
- `--scan-mode quick` — fast, PR-sized scan.
- `--max-budget 10` — hard cap on LLM spend per run (raise it for large diffs).
- **"Fail unless the scan completed" step** — a run that exhausts its budget can
  exit `0` while leaving `run.json` status `stopped`. This step reads that
  status and fails the job so the scan never "fails open" (passes silently
  without finishing). If you see it fail with a `stopped` status, raise
  `--max-budget`.

### Exit codes

| Code | Meaning                                                  |
| ---- | -------------------------------------------------------- |
| `0`  | No validated vulnerabilities in what was analyzed        |
| `2`  | Vulnerabilities found — **fails the job**                |
| `1`  | Setup error                                              |

Add `--fail-on high` to the scan command to fail only on high/critical
findings (lower-severity ones are still reported).

---

## Scan modes

| Mode       | Use for                                    | Suggested trigger          |
| ---------- | ------------------------------------------ | -------------------------- |
| `quick`    | Per-PR, diff-scoped (what this repo uses)  | `on: pull_request`         |
| `standard` | Broader coverage                           | Nightly schedule           |
| `deep`     | Thorough audit                             | Release candidates         |

To add a nightly `standard` scan, create a second workflow with
`on: schedule` and `--scan-mode standard` (drop the `--scope-mode diff` flags
so it scans the whole target, not just a diff).

---

## Running it locally

```bash
curl -sSL https://strix.ai/install | bash   # needs Docker running
export STRIX_LLM="openai/gpt-5.4"
export LLM_API_KEY="your-provider-key"
strix -t ./ --scan-mode quick
```

Results are written to `strix_runs/<run>/` (`run.json`, `findings.sarif`).

---

## Alternatives

- **Managed platform (no CI maintenance).** Instead of running the CLI in a
  runner, install the Strix GitHub App and enable PR reviews at
  https://app.strix.ai — no workflow file, no Docker, no LLM key. Lowest-effort
  path if you just want PR gating. See
  https://docs.strix.ai/integrations/github-actions.
- **Strix as an MCP client.** Strix can connect to *your* MCP servers (database,
  infrastructure, issue tracker) via `~/.strix/mcp-servers.json` so the agent
  tests with real context instead of inferring it from the outside. See
  https://docs.strix.ai/integrations/mcp.

---

## Note on this codebase

This is Odoo 9.0 (Python 2, end-of-life). The per-PR scan only covers what each
PR changes — it does not audit the existing history, which predates many modern
security practices. For a full picture, run a scheduled `standard`/`deep` scan
or the managed platform against the whole target.
