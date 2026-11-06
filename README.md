# ga_github_automation

Shared GitHub Actions automation for Microsoft Fabric repositories in Group Analytics.

This repository contains reusable workflows and shared action logic used by caller repositories to validate pull requests, deploy changed Fabric items to DEV, and run explicit releases to TEST and PROD for any Fabric item type.

## Current scope

This repository covers Fabric item deployment end to end, built on the [`fabric-cicd`](https://microsoft.github.io/fabric-cicd/) Python library:

Included:
- Reusable GitHub Actions workflows for any Fabric caller repository
- Detection of changed Fabric items between two Git commits, for any item type (Report, SemanticModel, Notebook, DataPipeline, Lakehouse, Warehouse, Environment, VariableLibrary, Eventhouse, and everything else `fabric-cicd` supports)
- Selective deployment of changed items to DEV
- Shared Python-based deployment logic for Fabric workspaces
- Release automation for TEST and PROD using explicit YAML configuration
- Explicit, reviewable handling of cross-environment dependencies (Direct Lake lakehouse bindings, cross-workspace report/semantic-model bindings, composite model dependencies) instead of relying on implicit Git-sync side effects
- Optional deployment/validation of non-Fabric-item, environment-scoped config files (e.g. an orchestration framework's per-model YAML) to a lakehouse's `Files/config` folder on OneLake — see [Config file deployment](#config-file-deployment)

Not included yet:
- Full standard support for composite model release scenarios beyond the documented `item_dependencies` mechanism
- Automated Warehouse DDL deployment under service principal auth (see [Warehouse DDL limitation](#warehouse-ddl-limitation) below — this is a Fabric platform constraint, not a gap in this tooling)

## Purpose

The purpose of this repository is to centralize reusable GitHub automation so that multiple caller repositories can use the same deployment and validation logic without duplicating scripts and workflow definitions.

This makes it easier to:
- maintain one shared automation standard across Power BI and engineering Fabric artifacts
- onboard new domains faster
- reduce copy/paste across repositories
- improve consistency in CI/CD behavior

## Design principles

The setup is based on the following principles:

- `main` is the deployable truth (trunk-based; no per-environment branches)
- Pull requests to `main` run validation only
- Deployment to DEV happens only after merge to `main`
- Only changed Fabric items are deployed to DEV
- Releases to TEST and PROD are explicit, not implicit side effects of a branch push
- Workspace IDs are managed centrally in the caller repository
- Every cross-environment value difference (variable values, item IDs, connection endpoints) is a declared, reviewable entry in a release-config file — never a value that changes silently as part of deployment
- Same-workspace dependencies (e.g. a Direct Lake semantic model's lakehouse) are resolved dynamically against the actual deployed target item, so they can never go stale
- Cross-workspace dependencies (e.g. a Report bound to a SemanticModel released separately) are resolved from a central item catalog, since the target item can't be created as a side effect of this item's own deploy
- Composite model scenarios are treated as controlled exceptions until validated

## Repository usage model

This repository is intended to be called from other repositories through reusable GitHub workflows.

Typical caller repositories:
- Power BI PBIP repositories
- Fabric engineering repositories (Notebooks, Data Pipelines, Lakehouses, Warehouses, Environments, Variable Libraries, ...)
- Repositories mixing both

## Expected caller repository config

A caller repository is expected to contain:

- `.github/config/catalog/workspaces.yml` — shared workspace name → GUID catalog (workspace names are matched case-insensitively against `dev_workspace_name`/`target_workspace_name`, but must otherwise match exactly — no partial matches)
- `.github/config/catalog/items.yml` — shared cross-workspace item catalog
- `.github/config/catalog/common-pq-parameters.yml` — shared Power Query parameter values per environment
- one release config file per workspace root in `.github/config/release/` (e.g. `project_A.yml` for a single-workspace repo, or `engineering.yml` / `analytics.yml` for a repo hosting multiple independently-deployed workspaces — see [Workspace layout](#workspace-layout))

All three `catalog/` files are shared across however many workspace roots a repo has — that's the point of splitting them out from the per-root release config. Everything else (target workspace names, `item_dependencies`) is specific to one workspace root's release, hence one file per root rather than one per item or one for the whole repo.

Engineering-artifact-specific folder conventions (native Fabric structures, not specific to this tooling):
- `Variables/<name>.VariableLibrary/valueSets/<env>.json` — per-environment value sets (see [VariableLibrary value sets](#variablelibrary-value-sets) — these must be hand-maintained, not generated via `item_dependencies`)
- `Environments/<name>.Environment/Libraries/CustomLibraries/*.whl`, `Environments/<name>.Environment/Setting/Sparkcompute.yml` — custom libraries and Spark configuration

### Workspace layout

Two repository layouts are supported, controlled by the `workspace_layout` input on `reusable-fabric-pr-validation.yml` / `reusable-fabric-deploy-dev.yml`:

- **`multi-top-level`** (default) — each top-level folder is its own Fabric workspace, with item folders directly one level below it (`<workspace>/<item_name>.<ItemType>/...`). Used by repositories that intentionally host many small, independent PBIP projects/workspaces side by side, all sharing one deployment pipeline.
- **`single-workspace`** — one Fabric-Git-connected workspace maps to one folder, matching what Fabric's own Git integration actually exports: item folders directly under that folder, optionally nested under purely organisational subfolders (Fabric "workspace folders", e.g. `Lakehouses/lh_x.Lakehouse`) that carry no workspace-boundary meaning of their own.

For `single-workspace`, that folder is controlled by `workspace_root` (default `"."`, the whole repo):

- **One workspace per repo** — leave `workspace_root` at its default. This is the typical layout for a repository connected directly to a single Fabric workspace via Git integration (e.g. `ga_cicd_test`).
- **Multiple independently-deployed workspaces per repo** — set `workspace_root` to a top-level folder name (e.g. `"Engineering"`, `"Analytics"`). Detection/deploy is scoped to just that subtree; a change under `Analytics/` never triggers anything scoped to `Engineering`. Give each workspace root its **own caller workflow file**, path-filtered to its own subtree, so each gets an independent trigger, run history, and independent approvals — genuinely separate deployment lifecycles using the same reusable workflows underneath:

  ```yaml
  # .github/workflows/deploy-engineering-dev.yml
  on:
    push:
      branches: [main]
      paths: ["Engineering/**"]
  jobs:
    deploy:
      uses: group-analytics/ga_github_automation/.github/workflows/reusable-fabric-deploy-dev.yml@main
      with:
        workspace_layout: single-workspace
        workspace_root: "Engineering"
        dev_workspace_name: "dev_engineering"
        workspace_catalog_path: .github/config/catalog/workspaces.yml
      secrets: inherit
  ```

  Repeat with `workspace_root: "Analytics"`, its own `dev_workspace_name`, and its own `paths: ["Analytics/**"]` in a second file. Release works the same way — point each workspace root's release workflow at its own `release/<name>.yml`, **and set `workspace_root` there too**: `reusable-fabric-release.yml` also defaults `workspace_root` to `"."` (the repo root), so without it, a release publishes from the whole repo instead of just that root's subtree — `fabric-cicd` then mirrors the root folder itself (e.g. `Engineering/`) as an actual folder *inside* the target Fabric workspace, instead of treating it as the workspace root. By default, `reusable-fabric-release.yml`'s job also gates on a GitHub Environment named after `target_environment` ("test"/"prod"), which every workspace root would share. To give each root its own approvers, set `github_environment_name` explicitly per root (e.g. `"engineering-test"` / `"analytics-test"`), and create those GitHub Environments (with their own required reviewers) under Settings → Environments:

  ```yaml
  # .github/workflows/release-engineering.yml (workflow_dispatch)
  with:
    item_name: "engineering"
    target_environment: ${{ inputs.target_environment }}
    workspace_root: "Engineering"
    github_environment_name: "engineering-${{ inputs.target_environment }}"
    ...
  ```

In every `single-workspace` case, also set `dev_workspace_name` on `reusable-fabric-deploy-dev.yml` to the DEV workspace's name — `project_directory` equals `workspace_root` (not a real workspace name) in this layout, so it can't itself serve as the workspace name the way it does in `multi-top-level`.

### Release config: `item_dependencies`

Each item's release-config YAML declares its cross-environment dependencies explicitly under `item_dependencies`, replacing hardcoded values in the item definition with either a dynamically-resolved value (same workspace) or a catalog lookup (cross workspace):

```yaml
item_name: "Sales Model"

test:
  target_workspace_name: "TEST_ai001_Data"
  item_dependencies:
    # Same-workspace dependency (e.g. Direct Lake lakehouse SQL endpoint).
    # Resolved dynamically against the actual deployed target-environment
    # item — no catalog entry needed, and it can never go stale.
    - dependency_item_name: "Sales_LH"
      dependency_item_type: "Lakehouse"
      scope: same_workspace
      find_value: "37dc8a41-dea9-465d-b528-3e95043b2356"  # DEV-environment value to replace
      attribute: sqlendpointid

    # Same-workspace dependency not tied to a specific named item — e.g. a
    # Direct Lake OneLake connection string embeds the workspace GUID
    # separately from the lakehouse GUID. 'dynamic_variable' overrides the
    # default $items.<Type>.<Name>.$<attribute> derivation.
    - dependency_item_name: "workspace"
      dependency_item_type: "Workspace"
      scope: same_workspace
      find_value: "661a8b10-3aa0-4649-9e14-2afcca6743a9"  # DEV workspace GUID to replace
      dynamic_variable: "$workspace.$id"

    # An $items.<Type> dynamic variable (whether from 'attribute' or an
    # explicit 'dynamic_variable') is automatically scoped to skip every
    # VariableLibrary when generated - see "VariableLibrary value sets"
    # below for why. Optional replace_in_item_type/replace_in_item_name
    # (string or list of strings, passed through to fabric-cicd's own
    # find_replace item_type/item_name filters) narrow this further, to
    # specific item(s) instead of "every item type except VariableLibrary" -
    # rarely needed.

    # Cross-workspace dependency (e.g. a Report bound via byConnection to a
    # SemanticModel released separately — byPath references don't need this
    # at all, fabric-cicd converts byPath to byConnection automatically when
    # both items are in the same workspace). Target values are resolved
    # from items.yml, since the dependency's workspace isn't part of this
    # item's own deploy run. Each find_* is optional — declare only the
    # literal DEV-authored values (from the report's actual byConnection
    # connection string) that differ from the target environment's:
    #   Data Source=powerbi://api.powerbi.com/v1.0/myorg/<workspace name>;
    #   initial catalog="<model name>";...;semanticmodelid=<model guid>
    - dependency_item_name: "Finance Model"
      dependency_item_type: "SemanticModel"
      dependency_environment: "test"
      scope: cross_workspace
      find_workspace_name: "DEV_ai001_Data"
      find_semantic_model_name: "Finance Model"
      find_semantic_model_id: "39e2762b-b77b-4199-8e21-9d298f7dc1a3"
```

`scope: cross_workspace` currently only supports `dependency_item_type: SemanticModel`, matching the two realistic cross-workspace item-reference scenarios (Report → SemanticModel, composite model dependencies). The corresponding `items.yml` entry can optionally set `semantic_model_name` per environment if the target model's display name differs from `dependency_item_name` (the DEV name / catalog key) — it defaults to matching if omitted.

### Power Query parameters: `common-pq-parameters.yml`

`use_common_environment_parameters: true` on an environment section pulls in that environment's `power_query_parameters` from the shared `common-pq-parameters.yml` catalog. Each entry supports two forms:

```yaml
# common-pq-parameters.yml
environments:
  test:
    power_query_parameters:
      # Legacy form: matches the literal placeholder __PARAM__<name>__ - only
      # works if the M-code was authored with that placeholder in place of a
      # real value.
      MyPlaceholderParam: "some TEST value"

      # {find_value, replace_value} form: matches an explicit literal already
      # in the source instead - e.g. the real DEV-authored value Fabric-
      # authored M-code actually contains (a SharePoint site URL, an API
      # base URL, etc.). Same idea as item_dependencies/replacements: the
      # source artifact is never rewritten with a placeholder, only the
      # published copy is parameterized.
      SharePoint_SiteRoot_Path:
        find_value: "https://contoso.sharepoint.com/sites/GroupAnalytics-Infiles-DEV"
        replace_value: "https://contoso.sharepoint.com/sites/GroupAnalytics-Infiles-TEST"
  prod:
    power_query_parameters:
      SharePoint_SiteRoot_Path:
        find_value: "https://contoso.sharepoint.com/sites/GroupAnalytics-Infiles-DEV"
        replace_value: "https://contoso.sharepoint.com/sites/GroupAnalytics-Infiles-PROD"
```

`find_value` is repeated per environment (the DEV literal doesn't change; only `replace_value` does) — the same repetition pattern already used by `item_dependencies` across `test:`/`prod:` sections in a release config. Only parameters that actually need to change per environment belong here; a Power Query parameter whose value is the same everywhere (e.g. a fixed folder path within whichever SharePoint site is selected) doesn't need an entry at all.

### Excluding an item type: `exclude_item_types`

Top-level (not per-environment) `exclude_item_types` removes item type(s) from `item_types_in_scope` entirely for every environment this release config covers:

```yaml
item_name: "lz036_GL_Engineering"
exclude_item_types:
  - "KQLQueryset"

test:
  ...
```

This is the escape hatch for an item type that simply can't publish correctly today — most commonly an upstream `fabric-cicd` bug (see "KQLQueryset same-workspace data source limitation" below) rather than anything this release config could otherwise fix. It's a blunt instrument: every item of that type is skipped for this release config's workspace root, not just the one hitting the problem. Prefer `item_dependencies`/`spark_pool_dependencies` whenever the underlying value can actually be fixed by a declared replacement — reach for `exclude_item_types` only when it can't (`find_replace`/`spark_pool` are both applied server-side during `fabric-cicd`'s own publish step, so they can only fix what happens to run *after* they're applied - see the note on `func_process_file` ordering below).

### Environment Spark pools: `spark_pool_dependencies`

An `Environment` item's `Setting/Sparkcompute.yml` can pin a custom **Workspace**-level Spark instance pool by ID (`instance_pool_id`). That pool only exists in the workspace it was created in — publishing the same `Environment` to a different workspace with `item_dependencies`/`find_replace` won't fix this, because `fabric-cicd` resolves Spark pool remapping as a distinct, structural `spark_pool` section in `parameter.yml`, not a text substitution. Left unmapped, publish fails with something like:

```
Failed to publish Environment 'x': ... SparkSettingsInvalidInstancePool ... 'The current selected pool <guid> is invalid.'
```

Declare it per environment alongside `item_dependencies`:

```yaml
test:
  target_workspace_name: "TEST_ai001_Data"
  spark_pool_dependencies:
    - find_instance_pool_id: "9dfbd45e-ecb5-45cf-855f-43600b85f0ae"  # DEV custom pool GUID
      replace_type: "Capacity"   # or "Workspace"
      replace_name: "My Custom Pool"  # target pool name in the TEST workspace/capacity
      item_name: "cfg_lz036_defaultLib"  # optional - restricts to specific Environment item(s); omit to apply to all
```

`replace_type`/`replace_name` must name a real **custom** pool that already exists in the target workspace or capacity — `fabric-cicd` resolves it via Fabric's [list workspace custom pools API](https://learn.microsoft.com/en-us/rest/api/fabric/spark/custom-pools/list-workspace-custom-pools), which only returns pools an admin explicitly created. **This does not include Fabric's built-in default "Starter Pool"** — it isn't a custom pool, so it will never resolve here (confirmed the hard way: `replace_type: "Capacity"`/`name: "Starter Pool"` fails with `Could not resolve custom Spark pool ... No matching pool found`). If the target environment should just use its own default Starter Pool, the fix isn't `spark_pool_dependencies` at all — remove `instance_pool_id` from the DEV `Environment`'s `Sparkcompute.yml` instead, so no environment pins a custom pool and there's nothing to remap. Only use `spark_pool_dependencies` when a *real, named* custom pool exists in the target workspace/capacity to point at.

### VariableLibrary value sets

`item_dependencies`' `same_workspace` scope resolves a dynamic variable (`$items.<Type>.<Name>.$id`, `$workspace.$id`, etc.) against another item **published earlier in the same run** — it works by literal `find_replace` substitution applied while `fabric-cicd` publishes each item, so the dependency has to already exist (from this run or a prior one) at the moment the referencing item is published.

`fabric-cicd` publishes item types in a fixed, non-configurable order (`constants.SERIAL_ITEM_PUBLISH_ORDER`), and **`VariableLibrary` is always first** — before `Lakehouse` (4th), `Eventhouse` (8th), or any other item type. This means a `VariableLibrary`'s own definition files (`variables.json`, `valueSets/*.json`) can **never** successfully reference another same-workspace item via a dynamic `item_dependencies` entry — the referenced item hasn't been published yet in this run, however early in `SERIAL_ITEM_PUBLISH_ORDER` it otherwise is. Attempting it fails at publish time with an error like:

```
Failed to publish VariableLibrary 'x': Item type 'Lakehouse' is invalid or not found in deployed items
```

This is a `fabric-cicd` platform constraint, not something a `config.yml`/`parameter.yml` setting can work around. (The reverse direction is fine: an item type that publishes *after* `VariableLibrary` — e.g. `DataPipeline`, 19th — can safely use `item_dependencies` to reference a `Lakehouse`/`Eventhouse`/etc., since those already exist in the run's item registry by then.)

Practical consequence: a `VariableLibrary`'s per-environment value sets (`valueSets/TEST.json`, `valueSets/PROD.json`, ...) must contain **real, hand-entered values** for anything that references another Fabric item by object ID (Lakehouse/Eventhouse/Warehouse IDs, etc.) — not the DEV-environment literal, and not a dynamic variable. Since those target IDs don't exist until the referenced item has been published to that environment at least once, bootstrapping a new environment is a two-step process:

**This isn't only about the value set files.** `VariableLibrary`'s base `variables.json` legitimately needs to keep its real DEV literal values (that's what DEV deploy actually publishes — DEV deploy never generates a `parameter.yml`/`find_replace` at all, so it's unaffected by any of this). But `find_replace` scans **every** published file by default, so if any `item_dependencies` entry's `find_value` also happens to match a literal in `variables.json` (very likely, if it's the same DEV GUID `variables.json` legitimately stores), a TEST/PROD release would try to apply that substitution to `variables.json` too and hit the exact same failure — even after fixing the value set files.

Because of this, `prepare-fabric-release-config` **automatically excludes every `VariableLibrary` from any `$items.<Type>` dynamic-variable replacement** (`attribute`-derived or explicit `dynamic_variable`) — it's never something that can legitimately resolve there, for any item type, not just the specific one a given dependency happens to reference, so there's no reason to make each caller repo enumerate an item-type whitelist that has to grow every time a new artifact type needs the same substitution. You don't need to configure anything for this; `replace_in_item_type`/`replace_in_item_name` exist only if you want *narrower* scoping than "every item type except VariableLibrary" (e.g. for precision, or if the same literal happens to also appear somewhere else it shouldn't be touched).

1. First release to the new environment: put in any placeholder value for the not-yet-known ID(s) (anything that isn't a `find_value` used elsewhere in that environment's `item_dependencies` — otherwise the substitution could unexpectedly touch the VariableLibrary too, e.g. if another item happens to share that literal for an unrelated reason). Let the release run to completion so the referenced items get created in the target workspace.
2. Look up their real object IDs (from the Fabric portal — each item's GUID is in its URL — or the Fabric API), hand-enter them into the value set, and re-release. These IDs are stable once assigned, so this is a one-time correction per environment, not a per-release maintenance burden.

## Config file deployment

Some caller repositories keep a folder of environment-scoped, non-Fabric-item YAML config (e.g. an orchestration framework's per-model config, consumed by a notebook at runtime from a lakehouse's `Files/config` folder rather than published as a Fabric item). `reusable-fabric-deploy-dev.yml`, `reusable-fabric-release.yml`, and `reusable-fabric-pr-validation.yml` can deploy/validate this alongside the normal Fabric item flow, gated entirely behind one opt-in input:

- `config_files_path` — path to a folder structured as `<config_files_path>/<env>/*` (e.g. `Engineering/ConfigFiles/environments`, with one subfolder per environment). The subfolder is matched against `target_environment` **case-insensitively**, so it doesn't have to share the caller's `target_environment` casing convention (see "Environment keys" below) — e.g. lowercase `test`/`prod` folders work fine even when `target_environment` is `TEST`/`PROD`. Leave `config_files_path` unset (the default) to skip this feature entirely; every other input below is only read when this is set.
- `config_validator_script_path` — a caller-repo Python script invoked once per file as `python <path> <file> --strict`. Optional; deployment still happens without it, just unvalidated.
- `config_schema_path` — path to a JSON schema file used for validation. Passed to the validation script via `--schema` argument. If omitted, the validation script uses its default schema location (e.g., `config-schema.json` in the same directory as the script). This allows different caller repos to enforce different config patterns instead of sharing a single schema — each project can maintain its own schema tailored to its specific needs.
- `variable_library_path`, `lakehouse_variable_name` (default `bronze_lakehouseId`), `lakehouse_display_name` — see lakehouse resolution below.

On DEV deploy, config files under `<config_files_path>/dev/` are deployed whenever that folder has changes, independently of whether any Fabric item also changed in the same push (a config-only commit still deploys — it isn't gated on the Fabric item `deploy` job having actually run, only on it not having *failed*). On release, `<config_files_path>/<target_environment>/` deploys after the Fabric item release succeeds, gated by the same GitHub Environment approval. PR validation runs the validator (if configured) over every environment folder's YAML files whenever anything under `config_files_path` changed in the PR.

### Config-only release: `reusable-config-files-release.yml`

`reusable-fabric-release.yml`'s config deployment always runs *after* that same run's Fabric item release, in the same job — by design, so a release never leaves config out of sync with the items it configures. But that means it can't be used to push *just* a config change to TEST/PROD while `main` also has Fabric item work in progress that isn't ready to release yet — releasing would take those items along with it.

`reusable-config-files-release.yml` is a separate, standalone workflow for exactly that case: it deploys config files to one environment and touches no Fabric item at all. Same `config_files_path`/`config_validator_script_path`/`config_schema_path`/`variable_library_path`/`lakehouse_variable_name`/`lakehouse_display_name`/`target_environment` inputs as above, plus:

- `release_config_path` — the same per-workspace-root release config file `reusable-fabric-release.yml` already uses (e.g. `.github/config/release/engineering.yml`). Only its `<target_environment>: target_workspace_name` entry is read; no separate workspace-name input to maintain, and no dependency on that file's `item_dependencies` or anything else Fabric-item-specific.
- `github_environment_name` — defaults to `target_environment`; point it at the same GitHub Environment `reusable-fabric-release.yml` uses (e.g. `"engineering-prod"`) if a config-only PROD push should require the same approvers as a full release.

Typical caller wiring - a `workflow_dispatch` mirroring the existing release workflow, just calling this instead:

```yaml
# .github/workflows/engineering-config-release.yml
on:
  workflow_dispatch:
    inputs:
      target_environment:
        required: true
        type: choice
        options: [TEST, PROD]
jobs:
  deploy-config:
    uses: group-analytics/ga_github_automation/.github/workflows/reusable-config-files-release.yml@main
    with:
      config_files_path: "Engineering/ConfigFiles/environments"
      config_validator_script_path: "Engineering/ConfigFiles/scripts/validate_config.py"
      config_schema_path: "Engineering/ConfigFiles/scripts/config-schema.json"
      workspace_catalog_path: .github/config/catalog/workspaces.yml
      release_config_path: .github/config/release/engineering.yml
      target_environment: ${{ inputs.target_environment }}
      variable_library_path: "Engineering/Variables/vl_lz036_01.VariableLibrary"
      lakehouse_display_name: "lh_lz036_bronze_01"
      github_environment_name: "engineering-${{ inputs.target_environment }}"
    secrets: inherit
```

### Target lakehouse resolution

No separate GUID catalog is maintained for this. The target lakehouse ID is resolved in order:

1. **VariableLibrary value** — `variable_library_path/valueSets/<target_environment>.json`'s `variableOverrides` entry named `lakehouse_variable_name`; if absent, `variable_library_path/variables.json`'s base `variables` entry of the same name (this is also what `dev` uses, since it has no value set of its own — see "VariableLibrary value sets" above). If found, it's then **verified** with a Fabric API call confirming that ID still exists in the target workspace as a `Lakehouse` named `lakehouse_display_name`.
2. **Fabric API fallback** — if step 1 found nothing, or found a value that failed verification, `deploy-config-files` instead lists the target workspace's `Lakehouse` items via the Fabric REST API and matches by `lakehouse_display_name`.

The verification step means this needs no special-casing for *how* a not-yet-bootstrapped environment's placeholder is written (empty, missing override, or any fake-but-well-formed GUID convention, e.g. the repeated-digit placeholders used elsewhere in this README's bootstrap examples) — whatever's there, if it doesn't actually resolve to the named Lakehouse in that workspace, resolution falls through to the by-name lookup automatically. It's also self-healing across a lakehouse ID rotating for any reason, not just first bootstrap.

## Environment keys

`target_environment` (`test`/`prod` in a release `workflow_dispatch`, `dev` internally for DEV deploy) is caller-defined free text — this repo's own scripts never hardcode a specific casing for it (`publish.skip`/`unpublish.skip` are set as plain booleans in the generated `config.yml`, not per-environment dicts, specifically to avoid that). Pick whatever convention fits your repo, e.g. lowercase `test`/`prod` (as in `ga_cicd_test`) or uppercase `TEST`/`PROD` (as in `ga_lz036`, to match an existing uppercase `VariableLibrary` value-set convention).

Whatever you pick, it must be used **consistently within that repo**, because the exact same string is reused in several places that do compare it case-sensitively:

- the release-config's per-environment YAML section key (`test:`/`TEST:`)
- the `items.yml` / `common-pq-parameters.yml` per-environment lookup key
- `deploy_with_config()`'s `environment` argument, which `fabric-cicd` itself matches case-sensitively wherever it needs an environment-specific value — most notably selecting a `VariableLibrary`'s active value set against its `valueSetsOrder` entries
- by default, the GitHub Environment name used for release approval gating (override via `github_environment_name` if you want this to differ)

A mismatch anywhere in that list — e.g. `target_environment: "TEST"` but a release-config section still keyed `test:` — fails at a specific, identifiable step (`Missing target_workspace_name in 'TEST'`, or a `VariableLibrary` silently falling back to `"Default value set"`), not necessarily at the first place you'd look.

Example:

- `dependency_environment: "prod"` means:
  - look up the semantic model under the `prod` section in `items.yml`
  - get its `workspace_name`
  - resolve `workspace_id` from `workspaces.yml`

## Reusable workflows

This repository provides:

- `reusable-fabric-pr-validation.yml`
- `reusable-fabric-deploy-dev.yml`
- `reusable-fabric-release.yml`

## Prerequisites

### Azure Active Directory & service principal

1. Create an Azure AD App Registration and generate a client secret (or configure federated credentials for OIDC — see [Authentication](#authentication) below)
2. Grant the following Microsoft Fabric API permissions (with admin consent):
   - `https://api.fabric.microsoft.com/Item.ReadWrite.All`
   - `https://api.fabric.microsoft.com/Workspace.ReadWrite.All`
3. Add the service principal to each target Fabric workspace with **Admin** role
4. Add the service principal to every Fabric connection referenced by deployed artifacts (data/source connections used by pipelines, notebooks, lakehouses, warehouses, or other items)

### Cross-workspace binding permissions

Any item that references another item in a workspace this repo *doesn't itself deploy to* (`scope: cross_workspace` in `item_dependencies`, e.g. a Report bound via `byConnection` to a SemanticModel released separately — see [items.yml](#release-config-item_dependencies)) needs the service principal to have access on that **external** workspace too — separate from, and in addition to, the Admin role on this repo's own target workspaces above.

**Viewer access is not consistently enough.** It got a first publish of the item past `PowerBIEntityNotFound` (the SP could resolve/import the connection), but a *later* republish of the same, otherwise-unchanged item failed with `The executing identity is not authorized to call POST on '.../updateDefinition'` — Fabric's `updateDefinition` check apparently enforces a higher bar than the initial import did. This is directionally consistent with Microsoft's own live-connection docs (*"Only users with Build permission for a semantic model can connect to a published semantic model by using the Power BI service live connection"*), though what actually resolved it here was coarser than that: granting the service principal the **Contributor** role on the entire external workspace (not just Build permission on the one dataset, which wasn't tried in isolation). If you hit `updateDefinition`/"not authorized" errors on a cross-workspace-bound item that published fine before, this is the confirmed fix: upgrade the service principal's access on the external workspace from Viewer to Contributor.

### Authentication

Both service principal client-secret and federated identity (OIDC) authentication are supported, chosen automatically based on whether `AZURE_CLIENT_SECRET` is provided:
- If `AZURE_CLIENT_SECRET` is set, the workflows and Python scripts use `ClientSecretCredential`
- If it's omitted, the workflow runs `azure/login@v2` (OIDC) and the Python scripts fall back to `DefaultAzureCredential()`, which picks up the resulting Azure CLI session

**To use OIDC**, the *calling* workflow (in your caller repo) must declare `permissions: id-token: write` itself — the reusable workflows here deliberately don't request it unconditionally, since that permission is validated against what the caller's repo/org allows before any step runs, and many orgs/GitHub Enterprise instances restrict `id-token` to `none` by default. Requesting it unconditionally here would break every caller using client-secret auth on such an org, not just ones that want OIDC. If you're using a client secret only, no `permissions:` block is needed in your caller workflow.

### Cross-repo access — no PAT needed

Caller repos reference both this repo's reusable workflows (`uses: group-analytics/ga_github_automation/.github/workflows/reusable-fabric-*.yml@ref`) and its composite actions (`uses: group-analytics/ga_github_automation/actions/*@ref`) directly — neither of these `actions/checkout`s this repository or requires a token. Both are resolved natively by GitHub against the ref given, governed entirely by **this repo's own Settings → Actions → General → Access** setting (allow access from repositories in the `group-analytics` organization, or list specific repos).

If a caller gets "workflow was not found" or "Can't find 'action.yml' ... for action", check that Access setting first — it's almost always the cause, not a token/permissions problem on the caller's side.

### Testing an unmerged branch of this repo

Composite action references inside these reusable workflows (e.g. `uses: group-analytics/ga_github_automation/actions/detect-changed-fabric-items@main`) don't automatically follow the ref a caller used to invoke the reusable workflow itself — `uses:` doesn't support expressions, so the ref has to be a static string. Normally that's `@main`. To test unmerged changes to this repo end to end, temporarily edit the `@main` refs in the relevant `reusable-fabric-*.yml` file(s) to your branch (e.g. `@feature/GA-491` — see the `ACTIONS_REF` comment near the top of each file), then revert to `@main` before merging.

### Warehouse DDL limitation

**Important:** Deployments that include Warehouse DDL operations (`CREATE TABLE`, `ALTER TABLE`, `CREATE VIEW`, etc.) via service principal authentication will fail due to insufficient SQL permissions. This is a Fabric platform permission constraint, not a gap in this tooling.

**Impact:**
- ✅ Supported: Warehouse DML operations (INSERT, UPDATE, DELETE)
- ❌ Blocked: DDL operations (CREATE TABLE, ALTER TABLE, etc.), initial warehouse schema setup

**Workarounds:**
1. Perform DDL operations manually in the Fabric portal, then use CI/CD for subsequent DML
2. Use CI/CD for non-DDL artifacts and deploy warehouse schema changes manually
3. Use user-principal authentication instead of a service principal (not recommended for production CI/CD)

### Known fabric-cicd bugs: automatic exclude-and-retry

`deploy_with_config()` aborts its *entire* run on any single item type's publish failure (confirmed against `fabric-cicd`'s `publish.py`: one try/except wraps the whole multi-item-type loop) — so a currently-unfixable bug affecting one item type would otherwise silently prevent every item queued after it from ever being attempted too, not just fail that one item. Both `deploy-fabric-item` (release) and `deploy-fabric-item-dev` (DEV) handle this the same way: try a normal deploy first: if it fails with the exact, confirmed error signature of a known upstream `fabric-cicd` bug (see below), exclude that item type from `item_types_in_scope` and automatically retry the rest of the deployment — logged both as a `::warning::` and in the run's Step Summary, so a skip is always visible, never silent. A failure that doesn't match a known signature still fails the run normally.

This needs no configuration in a caller repo and requires no code change to handle a future occurrence of the *same* known bug elsewhere — but it only ever protects against the specific bugs listed below, matched by a distinctive substring of the raw Fabric API error. A different, unrecognized failure always fails the run visibly, exactly as before.

### KQLDatabase move-without-parent limitation

**Important:** Re-publishing an *existing* `KQLDatabase` whose parent `Eventhouse` sits inside a workspace folder fails, even when it's already correctly placed there — a `fabric-cicd` bug ([issue #629](https://github.com/microsoft/fabric-cicd/issues/629), closed **"not planned"** — no fix expected), confirmed present in `1.3.0`. First-time creation on an empty workspace is unaffected — this only bites on the second and subsequent publishes of the same item once it already exists:

```
Failed to publish KQLDatabase 'x': Unhandled error occurred calling POST on '.../items/{id}/move'. Message: The child item cannot be moved without its parent item.
```

Because this is conditional on the item already existing — unlike the `KQLQueryset` bug below, which is unconditional — it can't be handled with a permanent `exclude_item_types` entry without also blocking the item's legitimate first-time creation in a not-yet-deployed environment (e.g. PROD before its first release). This is exactly what the automatic exclude-and-retry mechanism above is for: no config needed, and it doesn't care whether this is the first deploy or the hundredth.

### KQLQueryset same-workspace data source limitation

**Important:** A `KQLQueryset` whose data source references a `KQLDatabase` in the same workspace fails to publish via `fabric-cicd` — this is an upstream `fabric-cicd` bug ([issue #1105](https://github.com/microsoft/fabric-cicd/issues/1105)), confirmed present through the latest released version (`1.3.0` as of writing), not a gap in this tooling.

**Root cause:** Fabric's own Git export blanks the data source's `clusterUri` and swaps `databaseItemId` for a logical ID when it detects a same-workspace reference (intending it to auto-bind, the same idea as Lakehouse/Notebook logical-ID binding elsewhere). `fabric-cicd`'s resolution for this specific case is broken: it fails to find the just-published `KQLDatabase` in its own item registry, even when that `KQLDatabase` published successfully moments earlier in the same run. Publish fails with:

```
Failed to publish KQLQueryset 'x': Cannot find the KQL Database source with name 'None' as it is not yet deployed.
```

**`find_replace`/`item_dependencies` cannot fix this, even in principle.** `fabric-cicd` only attempts this resolution when a data source's `clusterUri` is exactly `""`, which looks like a natural `find_replace` target (replace the whole `"clusterUri": ""` literal with a placeholder) — but confirmed against `fabric-cicd`'s own `_publish_item` source, the per-item processing order is:

```python
file.contents = func_process_file(self, item, file)   # KQLQueryset's clusterUri resolution — crashes here
file.contents = self._replace_logical_ids(file.contents)
file.contents = self._replace_parameters(file, item)  # find_replace — never reached
```

(`VariableLibrary` is the only item type that skips `func_process_file` entirely, which is why the equivalent-looking fix above works for it and not for this.) The crash happens on the file's raw, unmodified content before `find_replace` is ever applied — there is no `parameter.yml` configuration that can reach it in time.

**Handled automatically** by the exclude-and-retry mechanism above, in both release and DEV deploy — nothing to configure. Unlike the `KQLDatabase` bug above, this one is **unconditional** (it fails on first-time creation too, not just re-publishing), so it's worth *also* declaring `exclude_item_types: ["KQLQueryset"]` (see above) in the release config once you know it applies: doing so skips the doomed first attempt entirely instead of paying for a failing publish-then-retry cycle every release. The `KQLQueryset` simply won't be part of the release until this is fixed upstream — recreate/update it manually in the target workspace's Fabric UI if needed there in the meantime. There's no available workaround that avoids either hand-editing the source artifact (ruled out — every value difference in this pipeline is a declared, reviewable config, never a hand-edited source file) or skipping the item type until `fabric-cicd` fixes this.

## Important notes

- Workspace IDs are resolved from the caller repo `workspaces.yml`
- Cross-workspace item IDs are resolved from the caller repo `items.yml`
- Same-workspace dependencies are resolved dynamically at deploy time via `fabric-cicd` dynamic variables — no catalog entry needed
- New items may start with empty IDs and be filled in later
- Composite models using DirectQuery to remote Power BI semantic models are not a standard release scenario yet
- Keep release config files small and explicit
