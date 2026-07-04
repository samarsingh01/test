# DQ Design Studio — Application Specification

**Document version:** 1.0  
**Status:** Draft  
**Owner:** Data Quality Engineering  

---

## 1. Purpose and Scope

DQ Design Studio is a Streamlit-based web application that provides a structured authoring environment for Data Quality (DQ) rule and population definitions. It replaces manual YAML editing with guided forms, live SQL validation, and GitHub-integrated version control. All definitions are stored as YAML files in a designated GitHub repository and follow the Control Model folder hierarchy.

---

## 2. Clarifications

### Session 2026-07-04

- Q: How should the application handle concurrent edit conflicts when two users modify the same file? → A: Detect version mismatch at commit time and require user to refresh/merge (optimistic locking)
- Q: What is the session duration and token refresh behavior for authentication? → A: 8-hour session with automatic token refresh; save/commit prompts re-auth if refresh fails
- Q: What timeout limits should apply to SQL query execution? → A: 60-second timeout; automatically retry once before failing
- Q: What happens when deleting/deprecating a population referenced by active rules? → A: Allow deletion only if all dependent rules are deprecated/inactive first
- Q: How should population_version_pin behave - strictly enforced or advisory? → A: Pin is advisory only; rule always uses latest population version regardless of pin value

---

## 3. Guiding Principles

- **GitHub is the source of truth.** The application reads from and writes to a GitHub repository. No definition exists authoritatively inside the application itself.
- **YAML is the file format.** All rules and populations are stored as structured YAML files. The application generates and parses YAML; users never write raw YAML.
- **Compile before commit.** SQL queries must be validated (formatted and dry-run) against a live database connection before a check-in is permitted.
- **Every change is traceable.** All commits require a mandatory comment. The application exposes the full commit history per file.
- **Populations are reusable.** One population definition may be referenced by many rules. The application enforces this relationship and surfaces which rules depend on a given population.

---

## 3. Repository Structure

The GitHub repository follows this folder hierarchy:

```
<repo-root>/
  <Control Model>/
    Rules/
      <rule_id>.yaml
    Populations/
      <population_id>.yaml
```

**Example:**

```
dq-rules-repo/
  crm-customer-salesforce/
    Rules/
      NUL-CUST-EMAIL-001.yaml
      NUL-CUST-PHONE-002.yaml
      FMT-CUST-DOB-004.yaml
    Populations/
      POP-CUST-ACTIVE.yaml
```

The `<Control Model>` segment maps to a domain/sub-domain/system combination (e.g. `crm-customer-salesforce`). The application treats this as the top-level organisational unit.

---

## 4. Application Pages

The application has five pages accessible from a persistent left navigation sidebar:

| # | Page | Primary Purpose |
|---|------|----------------|
| 1 | Population Authoring | Create, modify, and view population definitions |
| 2 | Rule Authoring | Create, modify, and view rule definitions |
| 3 | Config | Manage database connections |
| 4 | Export | Search and download rules and populations |
| 5 | (Sidebar) Repo Browser | Browse the GitHub repo tree (accessible from authoring pages) |

---

## 5. Population Authoring Page

### 5.1 Purpose

Allows DQ engineers to create new population definitions, modify existing ones, and view committed population files. A population defines the in-scope dataset against which one or more rules execute.

### 5.2 Entry Points

- **New population** — blank form with auto-suggested `population_id` based on Control Model and descriptor entered by the user.
- **Open existing** — browse the GitHub repository tree (see Section 7) to select a `.yaml` file from the `Populations/` folder of any Control Model.
- **Clone existing** — open an existing population as a starting point; clears the `population_id` and `version` fields.

### 5.3 Form Fields

The form is organised into three sections.

#### Section A — Identity

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `population_id` | Text (read-only after first save) | Yes | Auto-generated as `POP-<DESCRIPTOR>`. Immutable after first commit. |
| `population_name` | Text | Yes | Human-readable display name |
| `domain` | Text | Yes | Top-level domain (e.g. `crm`) |
| `sub_domain` | Text | Yes | Sub-domain (e.g. `customer`) |
| `system` | Text | Yes | System of execution (e.g. `salesforce`) |
| `owner` | Text (email) | Yes | Responsible engineer or team email |
| `tags` | Multi-value tag input | No | Free-form tags; displayed as chips |

#### Section B — Description and Source

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `population_description` | Multi-line text area | Yes | Business and technical scope; documents inclusions and exclusions explicitly |
| `source.schema` | Text | Yes | Database schema name |
| `source.table` | Text | Yes | Source table name |
| `source.primary_key` | Text | Yes | Primary key column |
| `source.select_cols` | Multi-value list input | Yes | Columns to include; displayed as an ordered list with add/remove/reorder controls |
| `source.filter` | Multi-line list input | No | One AND-clause per row; rows are joined at compile time |
| `source.partition_col` | Text | No | Column used for parallel execution partitioning |

#### Section C — Population Query

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `compiled_population_query` | SQL editor (read-only display) | System-managed | Generated by the compile action; never hand-edited |

### 5.4 SQL Actions

The following actions are available in the SQL action toolbar below Section C:

#### Format Query
- Reformats the `compiled_population_query` using a SQL formatter for readability.
- Does not validate against the database.
- Available at any time.

#### Validate & Run
- Requires a database connection to be selected (from the connections defined in Config).
- Executes an `EXPLAIN` statement against the query to validate syntax and resolve table/column references.
- If EXPLAIN succeeds, executes the query with a `LIMIT 100` row cap and displays results in a data table below the editor.
- Query execution has a 60-second timeout. If the timeout is reached, the query is automatically retried once. If the retry also times out, an error message is displayed suggesting query optimization.
- Displays the following metadata alongside results:
  - Estimated row count (from EXPLAIN)
  - Execution time
  - Columns returned
  - Any warnings (e.g. missing index on partition column)
- If EXPLAIN or execution fails, displays a formatted error message with the offending clause highlighted.

#### Connection selector
- Dropdown populated from saved Snowflake connections (Config page).
- Persists the selection for the session.

### 5.5 Check-in

The **Save & Commit** button is enabled only when:
- All required fields are populated.
- The `compiled_population_query` has been successfully validated at least once in the current session.

On click, a modal appears with:

| Field | Type | Required |
|-------|------|----------|
| Commit message | Text area | Yes — minimum 20 characters |
| Change type | Dropdown: `new` / `fix` / `refactor` / `chore` | Yes |
| Linked ticket | Text (e.g. `DQ-1042`) | No |
| Target branch | Text (default: `feat/<population_id>-<date>`) | Yes |
| Open pull request after commit | Toggle (default: on) | — |

On confirm:
1. The YAML is assembled from all form fields.
2. The `compiled_population_query` is written into the YAML.
3. The `version` block is updated: `number` incremented, `compiled_at` set to current UTC timestamp, `compiled_checksum` computed as SHA-256 of the compiled query, `created_by` set from the authenticated user.
4. The file is committed to the target branch.
5. If the PR toggle is on, a pull request is opened with the commit message as the PR title and the change summary as the body.
6. A success banner displays the commit SHA and a link to the PR.

### 5.6 View Mode

When a population is opened read-only (e.g. from the Export page or when the user has no write permission):
- All form fields are non-editable.
- The SQL action toolbar shows **Format** only (no Validate & Run).
- A **Clone to new** button allows opening the population as a new draft.
- The commit history panel is shown (see Section 5.7).

### 5.7 Commit History Panel

A collapsible panel below the form shows the Git log for the file:

| Column | Content |
|--------|---------|
| Version | `version.number` from YAML |
| Date | Commit timestamp |
| Author | Committer name |
| Message | Commit message (first line) |
| Ticket | Linked ticket if present |
| SHA | Short commit SHA (links to GitHub) |

Clicking a row loads that historical version into the form in read-only mode with a **Restore as draft** option.

### 5.8 Population Deletion and Deprecation

The application enforces referential integrity when deleting or deprecating populations:

#### Deletion
- When a user attempts to delete a population file (via GitHub commit), the application first scans all rule files in the same Control Model to identify dependencies.
- If any **active** rules (status = `active`) reference the population, deletion is blocked.
- The user is shown an error modal listing:
  - The number of active dependent rules
  - A table showing: Rule ID, Rule Name, Status, Owner
  - A message: "This population cannot be deleted while active rules depend on it. Please deprecate all dependent rules first, or deprecate this population instead."
- The modal includes buttons: **View dependent rules** (opens Export page filtered to those rules), **Cancel**.
- Deletion is only allowed when all dependent rules have status = `deprecated` or `inactive`.

#### Deprecation
- Users can change a population's status to `deprecated` at any time, regardless of dependencies.
- When a population is deprecated, it remains in the repository and can still be referenced by rules.
- Rules referencing a deprecated population show a warning banner: "This rule references deprecated population {population_id}. Consider updating to use an active population."
- Deprecated populations are excluded from the `population_ref` dropdown for new rules, but existing references are preserved.
- The Export page shows deprecated populations with a visual indicator (grayed out or strikethrough) and they can be filtered using the Status dropdown.

The form is organised into four sections.

#### Section A — Identity

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `rule_id` | Text (read-only after first save) | Yes | Auto-generated as `<TYPE>-<DOMAIN>-<SUBJECT>-<SEQ>`. Validated against regex `^[A-Z]{2,4}-[A-Z]{2,6}-[A-Z]{2,8}-[0-9]{3}$`. Immutable after first commit. |
| `rule_name` | Text | Yes | Human-readable display name |
| `domain` | Text | Yes | |
| `sub_domain` | Text | Yes | |
| `system` | Text | Yes | |
| `owner` | Text (email) | Yes | |
| `tags` | Multi-value tag input | No | |
| `dq_dimension` | Dropdown | Yes | `Completeness` / `Accuracy` / `Consistency` / `Timeliness` / `Uniqueness` / `Validity` |
| `severity` | Dropdown | Yes | `critical` / `high` / `medium` / `low` |

#### Section B — Description and Rationale

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `rule_description` | Multi-line text area | Yes | Technical description: what is checked and what a failure means in data terms |
| `rule_rationale` | Multi-line text area | Yes | Business rationale: why it matters, which workflows are affected, which policy mandates it |

#### Section C — KDE (Key Data Element)

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `kde.kde_id` | Text | Yes | Foreign key to enterprise data catalogue (e.g. `KDE-CRM-0042`) |
| `kde.kde_term` | Text | Yes | Business name from data catalogue |
| `kde.kde_server` | Text | Yes | Physical server of the authoritative data element |
| `kde.kde_database` | Text | Yes | Database name |
| `kde.kde_table` | Text | Yes | Table name |
| `kde.kde_column` | Text | Yes | Column name |

#### Section D — Population and Condition

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `population_ref` | Searchable dropdown | Yes | Lists all active populations from the same Control Model. Selecting one shows the population description and its `compiled_population_query` in a read-only preview panel. |
| `population_version_pin` | Text | No | Advisory field for documentation purposes. Allows users to record which version was used during rule authoring, but the rule execution always uses the latest active version of the referenced population. Displayed in the UI as "Reference version (advisory)" with a tooltip explaining the behavior. |
| `condition.type` | Dropdown | Yes | `null_check` / `format_check` / `referential` / `range_check` / `statistical` / `custom_sql` |
| `condition.target_col` | Text | Yes | Must be present in the referenced population's `select_cols` — validated on blur |
| `condition.operator` | Dropdown | Yes | Options filtered by `condition.type`: `IS NULL`, `IS NOT NULL`, `REGEX`, `NOT REGEX`, `BETWEEN`, `NOT BETWEEN`, `NOT IN`, `IN` |
| `condition.value` | Text / multi-value | Conditional | Hidden for `IS NULL` / `IS NOT NULL`. Pattern string for `REGEX`. `[min, max]` pair for `BETWEEN`. |
| `condition.threshold.fail_if_pct_gt` | Number (0.0–100.0) | Yes | Default `0.0` |
| `condition.threshold.warn_if_pct_gt` | Number (0.0–100.0) | Yes | Default `0.0` |
| `condition.output.include_failures` | Toggle | Yes | Default on |
| `condition.output.max_failure_rows` | Number | Yes | Default `1000` |

#### Section E — Rule Query

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `compiled_rule_query` | SQL editor (read-only display) | System-managed | Generated by the compile action; inlines the population query as a subquery |

### 6.4 SQL Actions

Same pattern as Population Authoring (Format, Validate & Run, Connection selector) with one addition:

#### Compile Rule
- Resolves `population_ref` to fetch the referenced population's latest active version and its `compiled_population_query`.
- If a `population_version_pin` is specified, it is recorded in the rule YAML for documentation purposes but does not affect which population version is retrieved (always uses latest).
- A warning banner is displayed if `population_version_pin` is populated: "Note: Version pin is advisory only. This rule will execute against the latest active version of the population."
- Assembles the full rule SQL: population as a subquery wrapped by the condition's WHERE clause.
- Runs EXPLAIN against the selected connection.
- Writes the result into `compiled_rule_query` on success.
- This step must succeed before Save & Commit is enabled.

#### Validate & Run
- Runs `compiled_rule_query` with `LIMIT 100`.
- Displays the failure rows in a data table with the following columns:
  - All columns from the population's `select_cols`
  - A `failure_reason` column (derived from the condition type)
- Displays summary metrics:
  - Total population rows (from EXPLAIN)
  - Failure count
  - Failure percentage
  - Pass / Fail verdict against threshold

### 6.5 Check-in

Same modal and process as Population Authoring (Section 5.5) with one addition:

- The commit message is pre-populated with `<change_type>(<rule_id>): <user-entered summary>` following conventional commit format.
- The PR body includes a summary table:

| Field | Value |
|-------|-------|
| Rule ID | `NUL-CUST-EMAIL-001` |
| Population | `POP-CUST-ACTIVE v2.1` |
| Condition | `EMAIL_ADDRESS IS NULL` |
| Threshold | fail > 0.0% |
| Compile status | ✓ Valid |
| Warnings | 1 (index warning on REGION_CODE) |

### 6.6 View Mode and Commit History

Same behaviour as Population Authoring (Section 5.6 and 5.7).

---

## 7. Repo Browser (Shared Component)

The Repo Browser is a sidebar panel accessible from both authoring pages via an **Open from repo** button.

### 7.1 Structure

- **Repository selector** — dropdown of configured repositories (from Config page). Defaults to the repository set in Config.
- **Branch selector** — lists all branches. Default: `main`.
- **File tree** — collapsible tree showing the `<Control Model>` folders, with `Rules/` and `Populations/` sub-folders. Clicking a `.yaml` file loads it into the authoring form.
- **Search box** — filters the file tree by rule ID or population ID substring.

### 7.2 Behaviour

- The file tree is fetched from the GitHub API on open and cached for the session (refresh button available).
- Opening a file from a non-main branch loads it in draft mode; the branch name is pre-populated in the check-in modal.
- A file opened from main is shown in read-only view mode unless the user creates a new branch (button available in the check-in modal).

---

## 8. Config Page

### 8.1 Purpose

Manages Snowflake database connections that are used by the SQL validation actions in both authoring pages.

### 8.2 GitHub Configuration

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| GitHub repository URL | Text | Yes | e.g. `https://github.com/acme/dq-rules-repo` |
| Personal access token | Password input | Yes | Stored encrypted; scopes required: `repo`, `pull_requests` |
| Default branch | Text | Yes | Default: `main` |

Test connection button — verifies token has access to the repository and displays the repo name and current default branch on success.

### 8.3 Snowflake Connection Management

Connections are listed in a table with columns: Name, Account, Database, Schema, Warehouse, Status (active / inactive), Actions (Edit, Test, Delete).

#### Add / Edit Connection Form

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| Connection name | Text | Yes | Unique display name (e.g. `dev-crm`, `prod-finance`) |
| Account identifier | Text | Yes | Snowflake account locator (e.g. `xy12345.eu-west-1`) |
| Username | Text | Yes | |
| Authentication | Dropdown | Yes | `Password` / `Key pair` / `SSO` |
| Password | Password input | Conditional | Required if authentication = Password |
| Private key path | Text | Conditional | Required if authentication = Key pair |
| Role | Text | No | Default role for the connection |
| Warehouse | Text | Yes | Snowflake virtual warehouse |
| Database | Text | Yes | Default database |
| Schema | Text | No | Default schema |
| Environment | Dropdown | Yes | `development` / `staging` / `production`. Production connections are read-only in authoring pages — they can be used to validate SQL (EXPLAIN only) but Validate & Run (full execution) is blocked. |

#### Test Connection
- Executes `SELECT CURRENT_VERSION()` against Snowflake.
- Displays Snowflake version, current role, and warehouse on success.
- Displays the full error message on failure.

### 8.4 Security Notes

- Credentials are stored in Streamlit secrets or an environment-configured secrets manager (e.g. AWS Secrets Manager). They are never written to the GitHub repository or logged.
- Production connections are always labelled and blocked from full query execution to prevent accidental large reads.

---

## 9. Export Page

### 9.1 Purpose

Allows any user (including read-only users) to search, browse, and download rule and population YAML files from the GitHub repository.

### 9.2 Search and Filter

| Filter | Type | Notes |
|--------|------|-------|
| Type | Radio: `Rules` / `Populations` / `Both` | |
| Control Model | Dropdown (multi-select) | Lists all `<Control Model>` folders found in the repo |
| Domain | Text | Substring match against `domain` field in YAML |
| Status | Dropdown: `active` / `draft` / `deprecated` / `all` | Filters on `version.status` |
| Owner | Text | Substring match against `owner` field |
| Tag | Text | Matches any tag in the `tags` list |
| Free text | Text | Matches against `rule_name`, `population_name`, `rule_description`, `population_description` |

Results are displayed in a table:

**Rules table columns:** Rule ID, Rule Name, Domain, Sub-domain, Category, DQ Dimension, Severity, Owner, Population Ref, Version, Status, Last Updated

**Populations table columns:** Population ID, Population Name, Domain, Sub-domain, System, Owner, Referenced by (count), Version, Status, Last Updated

### 9.3 Download Options

| Option | Description |
|--------|-------------|
| Download selected as YAML | Downloads selected rows as individual `.yaml` files in a `.zip` archive |
| Download selected as CSV | Flattens selected YAML metadata fields into a CSV (excludes compiled queries) |
| Download all matching | Same as above but for all search results |
| Copy rule ID list | Copies the list of rule IDs from the current results to the clipboard |

### 9.4 Quick View

Clicking any row in the results table opens a read-only side panel showing the full YAML content rendered as a formatted display (same as View Mode in the authoring pages), with a button to open the file in the Rule or Population Authoring page.

---

## 10. Cross-cutting Concerns

### 10.1 Authentication

- The application uses GitHub OAuth or a GitHub personal access token for authentication.
- Users without write access to the repository can use all View, Browse, and Export functions but cannot commit changes.
- Write access is required to Save & Commit on either authoring page.

#### Session Management
- Sessions are valid for 8 hours from the time of authentication.
- The application automatically attempts to refresh the GitHub token in the background every 4 hours.
- If a token refresh fails (e.g., token revoked, network issue), the user is notified with a non-blocking banner allowing them to continue read-only work.
- When attempting to Save & Commit with an expired or invalid token, the user is prompted to re-authenticate before the commit can proceed.
- Unsaved form data is preserved in session state during re-authentication flows to prevent data loss.
- Users can manually log out at any time, which clears all session state including unsaved form data (with confirmation prompt if unsaved changes exist).

### 10.2 Session State

- The selected database connection persists across page navigation within a session.
- Unsaved form edits trigger a warning dialog if the user navigates away.
- The Repo Browser remembers the last selected repository and branch within a session.

### 10.2.1 Concurrent Edit Conflict Resolution

The application implements optimistic locking to handle concurrent edits:
- When a file is opened, the application records the current commit SHA of the file from GitHub.
- At commit time, before creating the commit, the application checks if the file's HEAD commit SHA matches the recorded SHA.
- If the SHAs match, the commit proceeds normally.
- If the SHAs differ (indicating another user has committed changes), the commit is blocked and the user is presented with:
  - A notification that the file has been modified by another user
  - A diff view showing the conflicting changes
  - Options to: (1) Refresh and manually merge changes, or (2) Discard local edits and reload
- Users are encouraged to work in feature branches to minimize conflicts on shared files.

### 10.3 YAML Assembly

The application assembles YAML files programmatically using a strict field order that matches the specification. Fields not edited by users (version block, compiled queries) are written last. The output YAML is UTF-8 encoded, uses 2-space indentation, and block scalar style (`|`) for multi-line text fields.

### 10.4 Validation Rules (Client-side, before Compile)

| Rule | Description |
|------|-------------|
| `rule_id` format | Must match `^[A-Z]{2,4}-[A-Z]{2,6}-[A-Z]{2,8}-[0-9]{3}$` |
| `population_id` format | Must match `^POP-[A-Z0-9-]{2,20}$` |
| `condition.target_col` | Must appear in the referenced population's `select_cols` |
| `condition.value` | Required and validated by format for `REGEX`, `BETWEEN`, `NOT IN` |
| `threshold` | `warn_if_pct_gt` must be ≤ `fail_if_pct_gt` or both 0.0 |
| Commit message | Minimum 20 characters |
| Email fields | Standard email format |

### 10.5 Error Handling

- GitHub API errors (rate limit, auth failure, network) are surfaced as inline banners with a retry button.
- Snowflake connection failures show the full error message in an expandable panel.
- YAML parse errors on file open are shown with the offending line highlighted.

---

## 11. Technology Stack

| Component | Technology |
|-----------|------------|
| Application framework | Streamlit |
| GitHub integration | PyGitHub (`PyGithub` library) |
| Snowflake connectivity | `snowflake-connector-python` |
| SQL formatting | `sqlfluff` |
| YAML parsing and generation | `PyYAML` (with `ruamel.yaml` for round-trip preservation) |
| Secrets management | Streamlit Secrets / AWS Secrets Manager |
| Session state | Streamlit `st.session_state` |

---

## 12. Out of Scope (Version 1.0)

The following are explicitly excluded from this version:

- Rule execution scheduling (handled by the DQ Scheduler service)
- Alerting configuration per rule
- Bulk rule update UI (future version)
- Support for database engines other than Snowflake
- Role-based access control beyond GitHub repository permissions
- Inline diff view when opening a file from a non-main branch (commit diff is viewable on GitHub via the PR link)
