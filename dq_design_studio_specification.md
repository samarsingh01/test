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
- Q: What format should review checklists use in PR descriptions? → A: GitHub task list markdown format (`- [ ]` / `- [x]`) enabling interactive checkboxes
- Q: How should rule deprecation handle downstream impact detection (jobs, dashboards, alerts)? → A: Skip impact analysis entirely; show deprecation form without dependency information
- Q: Is the checklist validation bot part of DQ Design Studio or external? → A: Out of scope for v1.0; teams use existing GitHub Actions or third-party apps
- Q: How should historical data retention be implemented when deprecating rules? → A: Trigger webhook to external data retention service with retention policy
- Q: How should replacement population/rule IDs be used after deprecation? → A: Documentation-only; displayed in warnings and PR description

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
| 3 | Config | Manage database connections, GitHub configuration, review checklists, and data retention webhooks |
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
   - The PR description automatically includes the Population Review Checklist (see Section 5.9).
6. A success banner displays the commit SHA and a link to the PR.

### 5.6 View Mode

When a population is opened read-only (e.g. from the Export page or when the user has no write permission):
- All form fields are non-editable.
- The SQL action toolbar shows **Format** only (no Validate & Run).
- A **Clone to new** button allows opening the population as a new draft.
- The commit history panel is shown (see Section 5.7).
- If the population status is `active`, a **Deprecate** button is available (see Section 5.11).

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

### 5.9 Population Review Checklist

When a pull request is created for a population change, the PR description automatically includes a configurable review checklist using GitHub task list markdown format. Reviewers can check items directly in the PR description by clicking the checkboxes.

#### Default Population Review Checklist

The checklist is rendered in the PR description as GitHub task list markdown:

```markdown
- [ ] **Population Scope Verified**: The `population_description` accurately describes what records are included and excluded
- [ ] **Source Table Exists**: Confirmed that `source.schema` and `source.table` exist in the target database
- [ ] **Primary Key Valid**: The `source.primary_key` column is a valid unique identifier for the source table
...
```

Reviewers check items by clicking directly in the GitHub UI, which updates the markdown from `- [ ]` to `- [x]`.

#### Default Checklist Items

- [ ] **Population Scope Verified**: The `population_description` accurately describes what records are included and excluded
- [ ] **Source Table Exists**: Confirmed that `source.schema` and `source.table` exist in the target database
- [ ] **Primary Key Valid**: The `source.primary_key` column is a valid unique identifier for the source table
- [ ] **Column Selection Appropriate**: All columns in `source.select_cols` are necessary and exist in the source table
- [ ] **Filter Clauses Correct**: All `source.filter` conditions are syntactically correct and produce the intended subset
- [ ] **Partition Column Indexed**: If `source.partition_col` is specified, confirmed it has an index for performance
- [ ] **Query Validated**: The `compiled_population_query` has been successfully validated against a non-production connection
- [ ] **Performance Acceptable**: Query execution time and estimated row count are within acceptable limits
- [ ] **Ownership Confirmed**: The `owner` email is correct and the owner has been notified
- [ ] **Documentation Complete**: Business justification and technical details are clearly documented

#### Checklist Configuration

The checklist is stored in a configuration file `.dq-studio/review-checklists.yaml` in the GitHub repository:

```yaml
population_checklist:
  - id: scope_verified
    label: "Population Scope Verified"
    description: "The population_description accurately describes what records are included and excluded"
    required: true
  - id: source_exists
    label: "Source Table Exists"
    description: "Confirmed that source.schema and source.table exist in the target database"
    required: true
  # Additional items...
```

- Teams can customize the checklist by editing this configuration file.
- Each item can be marked as `required: true` (must be checked) or `required: false` (optional).
- The `required` flag is advisory for v1.0; teams implementing their own validation can use it to determine which items must be checked before merge.

### 5.10 Population Deletion and Deprecation

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

### 5.11 Deprecation Workflow

Both active and draft populations can be deprecated through a structured workflow that ensures proper documentation and impact analysis.

#### Initiating Deprecation

The **Deprecate** button is available:
- In View Mode for active populations (Section 5.6)
- In the authoring form action toolbar for populations in edit mode
- From the Export page via a context menu on population rows

Clicking **Deprecate** opens a modal dialog with the following form:

#### Deprecation Form

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| Deprecation reason | Dropdown + Text area | Yes | Dropdown options: `Replaced by new population` / `Data source retired` / `Business requirement changed` / `Data quality issues` / `Other`. If "Replaced by new population" selected, an additional "Replacement Population ID" field appears. Text area for detailed explanation (minimum 50 characters). |
| Effective date | Date picker | Yes | Date when the population should be considered deprecated. Can be set to today (immediate) or a future date. |
| Notification list | Multi-value email input | No | Email addresses of stakeholders to notify about the deprecation |
| Replacement population ID | Text (conditional) | Conditional | Required if deprecation reason is "Replaced by new population". Auto-complete from existing active populations. |
| Migration deadline | Date picker (conditional) | Conditional | Required if dependent rules exist. Date by which dependent rules should migrate to an alternative population. |

#### Impact Analysis

Before the deprecation form is shown, the application performs an impact analysis:

1. **Scans all rule files** in the same Control Model to identify dependencies
2. **Categorizes dependent rules** by status:
   - Active rules (status = `active`)
   - Draft rules (status = `draft`)
   - Deprecated rules (status = `deprecated`)
3. **Displays impact summary** in the modal above the form:
   ```
   Impact Analysis
   ⚠️ This population is referenced by:
   - 5 active rules
   - 2 draft rules
   - 1 deprecated rule
   
   View Dependent Rules
   ```
4. **"View Dependent Rules" link** opens a table showing: Rule ID, Rule Name, Status, Owner, Last Updated
5. If no active or draft rules depend on the population, displays: "✓ No active dependencies found. Safe to deprecate."

#### Deprecation Process

On form submission:

1. **YAML Update**: The application updates the population YAML:
   ```yaml
   version:
     status: deprecated
     deprecated_at: <effective_date_UTC>
     deprecated_by: <authenticated_user>
     deprecation_reason: <reason_category>
     deprecation_notes: <detailed_explanation>
     replacement_population_id: <replacement_id>  # if applicable
     migration_deadline: <deadline_date_UTC>  # if applicable
   ```

2. **Commit Creation**: A commit is created with:
   - Message: `chore(deprecate): <population_id> - <reason_category>`
   - Branch: `deprecate/<population_id>-<date>`
   - Body includes the full impact analysis and deprecation details

3. **Pull Request Creation**: If enabled, a PR is opened with:
   - Title: `Deprecate Population: <population_id>`
   - Body includes:
     - Deprecation reason and detailed notes
     - Impact analysis (list of dependent rules)
     - Effective date and migration deadline (if applicable)
     - Replacement population (if applicable): "Recommended replacement: <replacement_population_id>"
     - List of dependent rules that should be migrated
     - Population Review Checklist (modified for deprecation)
   - Labels: `deprecation`, `population`

4. **Notification**: If email addresses were provided in the notification list:
   - Comment is added to the PR mentioning the notification list
   - GitHub will notify those users (if they have repository access)
   - For external stakeholders, a note is added: "Please notify: <email_list>"

5. **Dependent Rules Update** (optional): 
   - If dependent rules exist, a banner in the PR description states: "Action required: Update dependent rules before <migration_deadline>"
   - The PR can include an optional GitHub issue creation to track the migration

#### Post-Deprecation Behavior

- Deprecated populations remain in the repository and are still accessible in View Mode
- The population form displays a prominent deprecation banner showing the reason, effective date, and replacement (if any)
- If a replacement population is specified, the banner includes: "This population has been deprecated. Please use <replacement_population_id> instead."
- Rules referencing deprecated populations show a warning banner during authoring: "Warning: Population <population_id> is deprecated. Consider migrating to <replacement_population_id>." (if replacement specified)
- In the Export page, deprecated populations are shown with a strikethrough and a "Deprecated" badge
- The "Referenced by" count in Export page includes a breakdown: "X active, Y draft, Z deprecated rules"
- Clicking on a deprecated population in Export page shows the deprecation details including replacement recommendation

**Note**: The replacement population ID is documentation-only. The application does not automatically migrate rule references. Users must manually update rules to reference the replacement population.

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

- The PR description automatically includes the Rule Review Checklist (see Section 6.7).

### 6.6 View Mode and Commit History

Same behaviour as Population Authoring (Section 5.6 and 5.7).

- If the rule status is `active`, a **Deprecate** button is available (see Section 6.8).

### 6.7 Rule Review Checklist

When a pull request is created for a rule change, the PR description automatically includes a configurable review checklist using GitHub task list markdown format. Reviewers can check items directly in the PR description by clicking the checkboxes.

#### Default Rule Review Checklist

The checklist is rendered in the PR description as GitHub task list markdown:

```markdown
- [ ] **KDE Reference Correct**: Confirmed that all KDE fields match the enterprise data catalogue
- [ ] **Population Scope Verified**: The referenced population accurately represents the intended scope
- [ ] **Threshold Calibrated**: Thresholds have been calibrated against baseline run on real data
- [ ] **Rationale Complete**: Rule rationale includes reference to governing policy or regulation
...
```

Reviewers check items by clicking directly in the GitHub UI, which updates the markdown from `- [ ]` to `- [x]`.

#### Default Checklist Items

- [ ] **KDE Reference Correct**: Confirmed that all KDE fields (`kde_id`, `kde_term`, `kde_server`, `kde_database`, `kde_table`, `kde_column`) match the enterprise data catalogue
- [ ] **Population Scope Verified**: The referenced population (`population_ref`) accurately represents the intended scope for this rule
- [ ] **Population Active**: The referenced population has status = `active` and is not deprecated
- [ ] **Target Column Valid**: The `condition.target_col` exists in the referenced population's `select_cols`
- [ ] **Condition Logic Correct**: The `condition.type`, `operator`, and `value` accurately represent the intended data quality check
- [ ] **Threshold Calibrated**: The `fail_if_pct_gt` and `warn_if_pct_gt` thresholds have been calibrated against a baseline run on real data
- [ ] **Rationale Complete**: The `rule_rationale` includes reference to the governing policy, regulation, or business requirement
- [ ] **Severity Appropriate**: The `severity` level matches the business impact of rule failures
- [ ] **Query Validated**: The `compiled_rule_query` has been successfully validated and test-run against non-production data
- [ ] **Performance Acceptable**: Query execution time is within acceptable limits for scheduled execution
- [ ] **Ownership Confirmed**: The `owner` email is correct and the owner has been notified
- [ ] **Documentation Complete**: Technical description and business rationale are clearly documented
- [ ] **Failure Output Configured**: The `output.max_failure_rows` is set to a reasonable limit (typically 1000-5000)

#### Checklist Configuration

The checklist is stored in the same configuration file `.dq-studio/review-checklists.yaml` in the GitHub repository:

```yaml
rule_checklist:
  - id: kde_correct
    label: "KDE Reference Correct"
    description: "Confirmed that all KDE fields match the enterprise data catalogue"
    required: true
  - id: population_verified
    label: "Population Scope Verified"
    description: "The referenced population accurately represents the intended scope"
    required: true
  - id: threshold_calibrated
    label: "Threshold Calibrated"
    description: "Thresholds have been calibrated against baseline run on real data"
    required: true
  - id: rationale_policy
    label: "Rationale Complete"
    description: "Rule rationale includes reference to governing policy or regulation"
    required: true
  # Additional items...
```

- Teams can customize the checklist by editing this configuration file.
- Each item can be marked as `required: true` (must be checked) or `required: false` (optional).
- The `required` flag is advisory for v1.0; teams implementing their own validation can use it to determine which items must be checked before merge.

#### Checklist Enforcement

For both population and rule checklists:
- Checklists use GitHub's native task list markdown syntax (`- [ ]` for unchecked, `- [x]` for checked).
- Reviewers manually check items by clicking checkboxes in the PR description.
- **Automated validation is out of scope for v1.0.** Teams can optionally configure their own enforcement using:
  - GitHub Actions workflows that parse PR descriptions and check for unchecked required items
  - Third-party GitHub Apps like PR Checklist Bot or similar tools
  - GitHub branch protection rules requiring specific reviewers who ensure checklist completion
- The `.dq-studio/review-checklists.yaml` configuration file marks items as `required: true` or `required: false` for teams implementing their own validation logic.
- The Config page Review Checklists tab is provided for convenient checklist management but does not include enforcement capabilities in v1.0.

### 6.8 Deprecation Workflow

Both active and draft rules can be deprecated through a structured workflow that ensures proper documentation and impact analysis.

#### Initiating Deprecation

The **Deprecate** button is available:
- In View Mode for active rules (Section 6.6)
- In the authoring form action toolbar for rules in edit mode
- From the Export page via a context menu on rule rows

Clicking **Deprecate** opens a modal dialog with the following form:

#### Deprecation Form

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| Deprecation reason | Dropdown + Text area | Yes | Dropdown options: `Replaced by new rule` / `Business requirement changed` / `KDE retired` / `Population no longer valid` / `False positive rate too high` / `Other`. If "Replaced by new rule" selected, an additional "Replacement Rule ID" field appears. Text area for detailed explanation (minimum 50 characters). |
| Effective date | Date picker | Yes | Date when the rule should stop executing. Can be set to today (immediate) or a future date. |
| Notification list | Multi-value email input | No | Email addresses of stakeholders to notify about the deprecation. Users should manually identify and list stakeholders who need to be informed. |
| Replacement rule ID | Text (conditional) | Conditional | Required if deprecation reason is "Replaced by new rule". Auto-complete from existing active rules. |
| Historical data retention | Dropdown | Yes | Options: `Retain all historical results` / `Archive results older than 90 days` / `Delete all results`. Default: `Retain all historical results`. This policy is passed to the configured data retention service via webhook. |

**Note**: Version 1.0 does not include automated downstream impact detection. Users are responsible for manually identifying and notifying affected stakeholders (job owners, dashboard maintainers, alert recipients) before deprecating a rule.

**Note**: Version 1.0 does not include automated downstream impact detection. Users are responsible for manually identifying and notifying affected stakeholders (job owners, dashboard maintainers, alert recipients) before deprecating a rule.

#### Deprecation Process

On form submission:

1. **YAML Update**: The application updates the rule YAML:
   ```yaml
   version:
     status: deprecated
     deprecated_at: <effective_date_UTC>
     deprecated_by: <authenticated_user>
     deprecation_reason: <reason_category>
     deprecation_notes: <detailed_explanation>
     replacement_rule_id: <replacement_id>  # if applicable
   execution:
     enabled: false
     disabled_at: <effective_date_UTC>
   ```

2. **Commit Creation**: A commit is created with:
   - Message: `chore(deprecate): <rule_id> - <reason_category>`
   - Branch: `deprecate/<rule_id>-<date>`
   - Body includes the full impact analysis and deprecation details

3. **Pull Request Creation**: If enabled, a PR is opened with:
   - Title: `Deprecate Rule: <rule_id> - <rule_name>`
   - Body includes:
     - Deprecation reason and detailed notes
     - Effective date
     - Replacement rule (if applicable): "Recommended replacement: <replacement_rule_id>"
     - Historical data retention policy
     - Notification list (stakeholders to inform)
     - Rule Review Checklist (modified for deprecation)
   - Labels: `deprecation`, `rule`

4. **Notification**: If email addresses were provided in the notification list:
   - Comment is added to the PR mentioning the notification list
   - GitHub will notify those users (if they have repository access)
   - For external stakeholders, a note is added: "Please notify: <email_list>"

5. **Success Confirmation**: A success banner displays the commit SHA and a link to the PR, along with a reminder: "Remember to update any downstream jobs, dashboards, or alerts that reference this rule."

6. **Data Retention Webhook** (if configured):
   - If a data retention webhook URL is configured in the Config page, the application triggers a POST request to the webhook endpoint with the following payload:
   ```json
   {
     "event": "rule_deprecated",
     "rule_id": "<rule_id>",
     "rule_name": "<rule_name>",
     "effective_date": "<YYYY-MM-DD>",
     "retention_policy": "retain_all|archive_90_days|delete_all",
     "deprecated_by": "<user_email>",
     "timestamp": "<UTC_timestamp>"
   }
   ```
   - The external data retention service can then execute the appropriate data lifecycle actions based on the retention policy.
   - If the webhook call fails, a warning banner is displayed but the deprecation PR still succeeds (data retention is decoupled from the deprecation workflow).

#### Post-Deprecation Behavior

- Deprecated rules remain in the repository and are still accessible in View Mode
- The rule form displays a prominent deprecation banner showing the reason, effective date, and replacement (if any)
- If a replacement rule is specified, the banner includes: "This rule has been deprecated. Please use <replacement_rule_id> instead."
- Deprecated rules are excluded from:
   - Scheduled execution (unless explicitly overridden)
   - New dashboard configurations
   - Alert rule selection dropdowns
- In the Export page, deprecated rules are shown with a strikethrough and a "Deprecated" badge
- The Export page can filter to show: `Active only`, `Include deprecated`, or `Deprecated only`
- When viewing a deprecated rule in the Export page, the deprecation details are displayed including the replacement recommendation

**Note**: The replacement rule ID is documentation-only. The application does not automatically update downstream systems (jobs, dashboards, alerts). Users must manually update these systems to reference the replacement rule.

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

Manages Snowflake database connections that are used by the SQL validation actions in both authoring pages, GitHub repository configuration, review checklist settings, and data retention webhook configuration.

The page is organized into three tabs: **GitHub**, **Snowflake Connections**, and **Review Checklists**, and **Data Retention**.

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

### 8.5 Review Checklists Configuration

The Config page includes a **Review Checklists** tab for managing the population and rule review checklists.

#### Checklist Editor

Displays two sections: **Population Review Checklist** and **Rule Review Checklist**. Each section shows:

- A table of checklist items with columns: Order (drag-to-reorder), Label, Description, Required (toggle), Actions (Edit, Delete)
- An **Add Item** button to create new checklist items
- A **Preview** button to see how the checklist will render in PR descriptions
- A **Reset to Defaults** button to restore the original checklist configuration

#### Add/Edit Checklist Item Form

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| Item ID | Text | Yes | Unique identifier (e.g., `kde_correct`, `scope_verified`). Auto-generated from label if not provided. |
| Label | Text | Yes | Short title displayed in the checklist (e.g., "KDE Reference Correct") |
| Description | Text area | Yes | Detailed explanation of what the reviewer should verify |
| Required | Toggle | Yes | If enabled, this item must be checked before PR can be merged |
| Order | Number | Yes | Display order in the checklist (auto-assigned, can be reordered via drag-and-drop) |

#### Checklist Validation Settings

The Config page Review Checklists tab provides information about implementing optional validation:

- **Enforcement Guidance** section with links to:
  - Example GitHub Actions workflows for checklist validation
  - Recommended third-party GitHub Apps
  - Documentation on configuring branch protection rules
- **Webhook URL** (informational field): Display-only field showing where teams can configure their validation service webhook if implemented
- Note: "Automated checklist validation is not included in v1.0. Configure external tools using the `.dq-studio/review-checklists.yaml` file to identify required vs optional items."

Changes to checklists are committed to `.dq-studio/review-checklists.yaml` in the repository and versioned like other configuration.

### 8.6 Data Retention Webhook Configuration

The Config page includes a **Data Retention** tab for configuring the optional webhook that is triggered when rules are deprecated.

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| Enable data retention webhook | Toggle | Yes | When enabled, deprecation events trigger a webhook call |
| Webhook URL | Text | Conditional | Required if enabled. HTTPS endpoint of the data retention service |
| Authentication method | Dropdown | Conditional | Options: `None` / `Bearer token` / `API key header` / `Basic auth`. Required if enabled. |
| Bearer token / API key | Password input | Conditional | Required if authentication method uses token or API key |
| API key header name | Text | Conditional | Required if "API key header" selected (e.g., `X-API-Key`) |
| Username | Text | Conditional | Required if "Basic auth" selected |
| Password | Password input | Conditional | Required if "Basic auth" selected |
| Timeout (seconds) | Number | Yes | Default: 10. Maximum time to wait for webhook response |
| Retry on failure | Toggle | Yes | Default: off. If enabled, retry up to 3 times with exponential backoff |

#### Test Webhook

- **Test** button sends a sample deprecation payload to the configured endpoint
- Displays the HTTP response status and body
- Helps verify the webhook configuration before actual use

**Security Note**: Webhook credentials are stored encrypted in the same secure storage as database credentials (Streamlit secrets or secrets manager).

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

## 12. Review Workflow

### 12.1 Pull Request Review Process

The application integrates with GitHub's pull request workflow:

1. **Author commits changes** using the Save & Commit button in either authoring page
2. **PR created automatically** with:
   - Conventional commit message as title
   - Summary table of changes
   - Appropriate review checklist (Population or Rule)
3. **Reviewers receive notification** via GitHub
4. **Reviewers complete checklist** by ticking items in the PR description
5. **Checklist validation** (if enabled):
   - GitHub status check verifies all required items are checked
   - PR cannot be merged until validation passes
6. **PR approved and merged** following standard GitHub workflow
7. **Changes deployed** to the main branch and become active

### 12.2 Checklist Management

**Viewing current checklists**: The Config page includes a **Review Checklists** tab showing the current configuration for both population and rule checklists.

**Editing checklists**: Users with admin access can:
- Add, remove, or reorder checklist items
- Mark items as required or optional
- Edit item labels and descriptions
- Preview how the checklist will appear in PR descriptions

**Version control**: The `.dq-studio/review-checklists.yaml` file is version-controlled, so checklist changes are auditable and can be rolled back if needed.

---

## 13. Out of Scope (Version 1.0)

The following are explicitly excluded from this version:

- Rule execution scheduling (handled by the DQ Scheduler service)
- Alerting configuration per rule
- Bulk rule update UI (future version)
- Support for database engines other than Snowflake
- Role-based access control beyond GitHub repository permissions
- Inline diff view when opening a file from a non-main branch (commit diff is viewable on GitHub via the PR link)
- **Automated checklist validation** - Teams must configure GitHub Actions, third-party apps, or manual review processes to enforce checklist completion
- Email notifications for checklist reminders (handled by GitHub notifications)
- Automated downstream impact detection for rule deprecation (detection of jobs, dashboards, alerts requires manual identification or external integration)
- Automatic removal of deprecated rules from downstream systems (manual intervention required)
