# WordPress.com Composer Deployment Design

**Status:** Approved  
**Date:** 27 July 2026  
**Repository:** `henryperkins/mcp-adapter`  
**Production branch:** `trunk`  
**Target site:** `hperkins.blog`

## Problem

The WordPress.com GitHub Deployment currently deploys the MCP Adapter repository source without a build step. The repository intentionally excludes `vendor/`, while current `trunk` requires the Jetpack Autoloader at `vendor/autoload_packages.php`. WordPress can therefore mark MCP Adapter active while its bootstrap returns before initializing the plugin.

Production will continue following `trunk`, including changes made after the v0.5.0 release. The deployment must build the exact production dependency set locked by `composer.lock` before WordPress.com receives the artifact.

## Goals

1. Install locked production Composer dependencies on every deployment.
2. Generate both the standard Composer autoloader and the Jetpack Autoloader.
3. Give the Jetpack Autoloader the plugin's declared version rather than an inferred `dev-trunk` version.
4. Fail before deployment when dependency or artifact requirements are not satisfied.
5. Deploy only the plugin runtime surface.
6. Keep generated dependencies out of Git.

## Non-goals

- Pin production to the latest stable release.
- Commit `vendor/`.
- Modify MCP Adapter runtime behavior.
- Add the full PHP or WordPress test suite to the deployment workflow.
- Change Flavor Agent, HPerkins Tokens, or any database-owned site content.
- Publish changes to the upstream `WordPress/mcp-adapter` repository.

## Approaches considered

### 1. Direct locked Composer build — selected

Install production dependencies from `composer.lock`, validate the generated autoloaders, and upload an allowlisted runtime artifact. This is the smallest workflow that satisfies the deployment contract.

### 2. Recreate the upstream npm release package

Run the upstream Node-based `plugin-zip` process, extract the ZIP, and upload its contents. This closely mirrors the release workflow but adds Node installation, npm dependencies, ZIP creation, and extraction without providing runtime value for this deployment.

### 3. Commit generated dependencies

Commit `vendor/` so the basic deployment contains dependencies. This avoids a build step but creates generated-file churn, increases repository size, and makes dependency provenance easier to drift from `composer.lock`.

## Selected workflow

Add `.github/workflows/wpcom.yml` to the fork.

### Trigger and permissions

- Run on every push to `trunk`.
- Support `workflow_dispatch` for an explicit rebuild or recovery deployment.
- Grant only `contents: read`.
- Use a concurrency group for WordPress.com deployments and cancel superseded in-progress runs.
- Pin third-party actions to immutable commit SHAs.

### PHP and Composer environment

- Use PHP 8.3 and Composer 2, matching the upstream release build environment.
- Read the version declared by `WP_MCP_VERSION` in `mcp-adapter.php`.
- Fail if the declared version cannot be found or normalized.
- Export that value as `COMPOSER_ROOT_VERSION` for the production install.

The current declared version is `0.5.0`. Deriving it from source prevents the workflow from silently retaining a stale hard-coded value when upstream changes the plugin version. It also avoids Composer's inferred `dev-trunk`, which the Jetpack Autoloader treats as always newest and could incorrectly outrank a genuinely newer bundled MCP Adapter.

### Dependency build

Run:

```sh
composer validate --strict --no-check-publish
composer install --no-dev --prefer-dist --no-interaction --no-progress --optimize-autoloader
composer check-platform-reqs --no-dev
```

`composer install` must consume the committed lock file. The workflow must not run `composer update`.

### Build gates

Before creating the artifact, fail unless all of the following are true:

- `vendor/autoload.php` is readable.
- `vendor/autoload_packages.php` is readable.
- `mcp-adapter.php` exists.
- `includes/` exists.
- Composer reports that production platform requirements are satisfied.

A failed gate produces no `wpcom` artifact and therefore no WordPress.com deployment.

### Artifact contract

Upload an artifact named exactly `wpcom` with a one-day retention period.

Use a positive runtime allowlist:

- `mcp-adapter.php`
- `includes/`
- `vendor/`
- `readme.txt`
- `README.md`
- `LICENSE.md`
- `CHANGELOG.md`

Do not include tests, development tooling, repository metadata, GitHub workflows, Node dependencies, Composer manifests, or Composer lock files in the production artifact.

## WordPress.com activation

After the workflow is merged:

1. Confirm that the MCP Adapter GitHub Deployment targets `henryperkins/mcp-adapter` and branch `trunk`.
2. Switch or confirm the deployment configuration is in advanced mode.
3. Select the new `wpcom.yml` workflow.
4. Run the workflow manually once.
5. Confirm WordPress.com consumes the successful `wpcom` artifact.

The HPerkins Tokens theme deployment remains independent and unchanged.

## Verification

A deployment is accepted only when:

1. The GitHub workflow completes successfully.
2. The downloadable artifact contains the runtime allowlist and both autoloader files.
3. MCP Adapter remains active on hperkins.blog.
4. The missing Composer autoloader notice is absent.
5. `wp mcp-adapter list` completes successfully.
6. The default MCP Adapter server is registered.
7. Flavor Agent's MCP server is registered when Flavor Agent is active.
8. A basic authenticated MCP discovery request succeeds.

## Failure handling and rollback

- Dependency, validation, or artifact failures stop before upload.
- Concurrency prevents an older run from overwriting a newer trunk deployment.
- A failed production verification is handled by redeploying the last known-good commit.
- If MCP Adapter prevents administrative recovery, deactivate it before restoring the last known-good artifact or the official stable release package.
- This change has no database migration and does not mutate site content, so rollback is limited to plugin files and activation state.

## Security and reproducibility

- Dependencies come from the committed `composer.lock`.
- The workflow has read-only repository permissions.
- No deployment secret is required by the workflow.
- Third-party actions are SHA-pinned.
- Production dependencies are regenerated for each artifact and are never treated as source.
- The artifact is retained for one day, matching WordPress.com's guidance.

## Acceptance criteria

- A push to `trunk` produces one successful `wpcom` artifact.
- The artifact contains `vendor/autoload.php` and `vendor/autoload_packages.php`.
- The artifact contains no `tests/`, `.github/`, `node_modules/`, or `vendor/bin/`.
- Activating the deployed plugin produces no missing-autoloader notice.
- MCP Adapter and Flavor Agent MCP discovery work on hperkins.blog.
- `vendor/` remains untracked in Git.
