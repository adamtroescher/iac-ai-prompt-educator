---
name: python-package-security
description: >
  Activates when Python packages are added to dependency files
  (requirements.txt, pyproject.toml, setup.cfg, setup.py, Pipfile).
  For each new package: searches NVD and OSV for CVEs, identifies the most
  recent release that is at least 4 weeks old with no known vulnerabilities,
  pins that version in the dependency file, and regenerates the lock file.
when_to_use: >
  Auto-trigger when requirements.txt, requirements*.in, pyproject.toml,
  setup.cfg, setup.py, or Pipfile is edited to add new package dependencies.
  Also invoke manually via /python-package-security or when the user asks to
  audit, check, or pin Python package versions for security.
allowed-tools:
  - WebFetch
  - WebSearch
  - Bash
  - Read
  - Edit
paths:
  - "**requirements*.txt"
  - "**requirements*.in"
  - "**/pyproject.toml"
  - "**/setup.py"
  - "**/setup.cfg"
  - "**/Pipfile"
---

# Python Package Security — Skill Instructions

You are performing a security-aware version pinning workflow for newly added Python packages. Follow every step below in order. Do not skip steps.

---

## Step 1 — Identify Newly Added Packages

Determine which packages were just added. Use the approach that fits the situation:

- **If triggered by a file edit in this session**: diff the before/after of the dependency file to extract package names that were added (ignore version specifiers, extras, and comments).
- **If invoked manually**: ask the user which packages to audit, or read the dependency file and audit all unpinned packages.

For each identified package name, proceed through Steps 2–5.

---

## Step 2 — Fetch All Available Versions from PyPI

For each package, fetch its full release history from PyPI:

```
GET https://pypi.org/pypi/<package-name>/json
```

From the response, build a list of `(version, upload_date)` pairs. Use the earliest `upload_time` entry within each version's file list as the release date. Parse dates as ISO 8601 (e.g. `"2025-01-15T10:30:00.000000Z"`).

**Filter the candidate list:**
- Keep only versions whose upload date is **at least 28 days before today** (the "4-week soak" rule — new releases often carry undiscovered CVEs).
- Exclude pre-release versions (alpha, beta, rc, dev suffixes) unless the current pinned version is also a pre-release.
- Sort remaining candidates descending by version (newest first).

---

## Step 3 — Check Each Candidate Version for Known CVEs

For each candidate version (newest first), query two sources:

### 3a. OSV (Open Source Vulnerabilities) — primary source

```
POST https://api.osv.dev/v1/query
Content-Type: application/json

{
  "package": { "name": "<package-name>", "ecosystem": "PyPI" },
  "version": "<candidate-version>"
}
```

If the response contains a non-empty `"vulns"` array, this version has known vulnerabilities — skip it and try the next candidate.

### 3b. NVD (National Vulnerability Database) — secondary source

Search for package-level CVEs to catch any not yet in OSV:

```
GET https://services.nvd.nist.gov/rest/json/cves/2.0?keywordSearch=<package-name>&noRejected
```

For each CVE returned, check if its affected version ranges include the candidate version. The `configurations` → `nodes` → `cpeMatch` fields contain `versionStartIncluding`, `versionEndExcluding`, and `versionEndIncluding`. If the candidate version falls within any affected range, skip it.

When evaluating NVD results, reference the CVE detail page format for context:
`https://nvd.nist.gov/vuln/detail/<CVE-ID>`

### 3c. Selection rule

Accept the **first candidate version that passes both checks** (no OSV vulns, not in any NVD affected range). This is the "known good" version.

If all candidates fail, report this to the user with the CVE list and ask them to decide whether to proceed or remove the package.

---

## Step 4 — Pin the Version in the Dependency File

Update the dependency file to pin the selected version using exact pinning (`==`):

| File type | Example before | Example after |
|---|---|---|
| `requirements.txt` / `requirements*.in` | `langchain` or `langchain>=0.1` | `langchain==<good-version>` |
| `pyproject.toml` (PEP 621 `[project]`) | `"langchain"` or `"langchain>=0.1"` | `"langchain==<good-version>"` |
| `pyproject.toml` (Poetry `[tool.poetry.dependencies]`) | `langchain = "*"` or `langchain = "^0.1"` | `langchain = { version = "==<good-version>", ...}` |
| `setup.cfg` `[options] install_requires` | `langchain` | `langchain==<good-version>` |
| `Pipfile` `[packages]` | `langchain = "*"` | `langchain = "==<good-version>"` |

Preserve all existing formatting, comments, and other entries in the file.

---

## Step 5 — Regenerate the Lock File

Detect which lock toolchain the project uses and regenerate:

| Lock file present | Command to run |
|---|---|
| `uv.lock` | `uv lock` |
| `poetry.lock` | `poetry lock --no-update` |
| `Pipfile.lock` | `pipenv lock` |
| `requirements.txt` generated from `requirements.in` (pip-tools) | `pip-compile requirements.in -o requirements.txt` |
| No lock file (bare `requirements.txt`) | `pip install --dry-run -r requirements.txt` to validate; no lock file to regenerate |

If the lock command fails, report the error verbatim to the user and do not proceed silently.

---

## Step 6 — Report to the User

After processing all packages, output a summary table:

```
Package        | Pinned Version | Released       | CVEs Found | Action
-------------- | -------------- | -------------- | ---------- | ------
langchain      | 0.3.14         | 2025-01-10     | 0          | Pinned, lock updated
requests       | 2.31.0         | 2023-05-22     | 0          | Pinned, lock updated
some-pkg       | —              | —              | 3 (see below) | Blocked — awaiting your decision
```

For any blocked package, list each CVE with its NVD link:
- [CVE-2025-XXXX](https://nvd.nist.gov/vuln/detail/CVE-2025-XXXX) — brief description

---

## Operational Notes

- **Do not downgrade packages that were already pinned.** Only act on newly added or previously unpinned packages.
- **The 4-week rule is non-negotiable.** Do not select a version released less than 28 days ago, even if it has no listed CVEs — the window for CVE disclosure may not have closed.
- **OSV is the primary authority.** NVD is used as a supplementary check because NVD indexing can lag days to weeks behind disclosure.
- **Never remove a package** without explicit user instruction. If no safe version exists, report the finding and wait.
- **Preserve existing pinned versions** for packages not being audited in this run.
