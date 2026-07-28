# WordPress.com Composer Deployment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build locked production Composer dependencies on every `trunk` deployment and configure hperkins.blog to consume the resulting WordPress.com artifact.

**Architecture:** A single GitHub Actions workflow derives the plugin version from `WP_MCP_VERSION`, installs the production dependency graph from `composer.lock`, gates on both Composer and Jetpack autoloaders, and uploads a minimal artifact named `wpcom`. WordPress.com GitHub Deployments remains attached to `trunk` but uses advanced mode to consume this workflow artifact.

**Tech Stack:** GitHub Actions, PHP 8.3, Composer 2, `actions/checkout`, `shivammathur/setup-php`, `actions/upload-artifact`, WordPress.com GitHub Deployments.

## Global Constraints

- Production continues following branch `trunk`; do not pin to v0.5.0.
- Install only dependencies locked by `composer.lock`; never run `composer update`.
- Set `COMPOSER_ROOT_VERSION` from `WP_MCP_VERSION` before installation.
- Do not commit `vendor/`.
- The deployment artifact must be named exactly `wpcom` and retained for one day.
- The artifact must include only plugin runtime files and must exclude `vendor/bin/`.
- Do not modify `henryperkins/hperkins-tokens`, Flavor Agent, or database-owned content.

---

### Task 1: Add the production deployment workflow

**Files:**
- Create: `.github/workflows/wpcom.yml`

**Interfaces:**
- Consumes: `composer.lock`, `composer.json`, and the `WP_MCP_VERSION` constant in `mcp-adapter.php`.
- Produces: one GitHub Actions artifact named `wpcom` containing the deployable plugin root.

- [ ] **Step 1: Confirm the workflow does not already exist**

Fetch `.github/workflows/wpcom.yml` from `trunk` and require a GitHub 404 response. If it exists, fetch its blob SHA and replace it intentionally instead of attempting a create operation.

- [ ] **Step 2: Create the workflow**

Create `.github/workflows/wpcom.yml` with this exact content:

```yaml
name: Publish Website

on:
  push:
    branches:
      - trunk
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: wpcom-mcp-adapter-production
  cancel-in-progress: true

jobs:
  Publish-Website:
    name: Publish Website
    runs-on: ubuntu-24.04
    timeout-minutes: 10

    steps:
      - name: Checkout repository
        uses: actions/checkout@9c091bb21b7c1c1d1991bb908d89e4e9dddfe3e0 # v7.0.0
        with:
          persist-credentials: false
          show-progress: ${{ runner.debug == '1' && 'true' || 'false' }}

      - name: Set up PHP
        uses: shivammathur/setup-php@f3e473d116dcccaddc5834248c87452386958240 # v2.37.2
        with:
          php-version: '8.3'
          coverage: none
          tools: composer:v2

      - name: Resolve plugin version
        shell: bash
        run: |
          set -euo pipefail
          version="$(php -r '$source = file_get_contents("mcp-adapter.php"); if (false === $source || 1 !== preg_match("/define\\(\\s*\\x27WP_MCP_VERSION\\x27\\s*,\\s*\\x27([^\\x27]+)\\x27\\s*\\)/", $source, $matches)) { fwrite(STDERR, "Unable to read WP_MCP_VERSION.\\n"); exit(1); } echo $matches[1];')"

          if [[ ! "$version" =~ ^[0-9]+\.[0-9]+\.[0-9]+(-[0-9A-Za-z.-]+)?(\+[0-9A-Za-z.-]+)?$ ]]; then
            echo "WP_MCP_VERSION is not a normalized semantic version: $version" >&2
            exit 1
          fi

          printf 'COMPOSER_ROOT_VERSION=%s\n' "$version" >> "$GITHUB_ENV"
          echo "Using COMPOSER_ROOT_VERSION=$version"

      - name: Validate Composer metadata
        run: composer validate --strict --no-check-publish

      - name: Install locked production dependencies
        run: composer install --no-dev --prefer-dist --no-interaction --no-progress --optimize-autoloader

      - name: Check production platform requirements
        run: composer check-platform-reqs --no-dev

      - name: Verify deployment inputs
        shell: bash
        run: |
          set -euo pipefail
          test -f mcp-adapter.php
          test -d includes
          test -r vendor/autoload.php
          test -r vendor/autoload_packages.php

      - name: Upload deployment artifact
        uses: actions/upload-artifact@043fb46d1a93c77aae656e7c1c64a875d1fc6a0a # v7.0.1
        with:
          name: wpcom
          path: |
            mcp-adapter.php
            includes/
            vendor/
            readme.txt
            README.md
            LICENSE.md
            CHANGELOG.md
            !vendor/bin
            !vendor/bin/**
          if-no-files-found: error
          retention-days: 1
```

- [ ] **Step 3: Validate the workflow statically**

Parse the file as YAML and assert the deployment contract directly from its text:

```bash
python - <<'PY'
from pathlib import Path
import yaml

path = Path('.github/workflows/wpcom.yml')
text = path.read_text()
yaml.compose(text)

required = (
    'branches:\n      - trunk',
    'workflow_dispatch:',
    'permissions:\n  contents: read',
    'COMPOSER_ROOT_VERSION',
    'composer validate --strict --no-check-publish',
    'composer install --no-dev --prefer-dist --no-interaction --no-progress --optimize-autoloader',
    'composer check-platform-reqs --no-dev',
    'test -r vendor/autoload.php',
    'test -r vendor/autoload_packages.php',
    'name: wpcom',
    'retention-days: 1',
    '!vendor/bin/**',
)
for marker in required:
    assert marker in text, marker
assert 'composer update' not in text
print('wpcom workflow contract: PASS')
PY
```

Expected: `wpcom workflow contract: PASS`.

- [ ] **Step 4: Commit the workflow to `trunk`**

Commit only `.github/workflows/wpcom.yml` with message:

```text
ci: build Composer dependencies for WordPress.com
```

The push itself must start the `Publish Website` workflow.

- [ ] **Step 5: Verify the GitHub Actions build and artifact**

Require the workflow job to complete successfully. Download the `wpcom` artifact and verify:

```bash
unzip -l wpcom.zip | grep -E 'vendor/autoload\.php|vendor/autoload_packages\.php|mcp-adapter\.php'
if unzip -Z1 wpcom.zip | grep -Eq '(^|/)(tests|node_modules|vendor/bin|\.github)(/|$)'; then
  echo 'Artifact contains an excluded path.' >&2
  exit 1
fi
```

Expected: both autoloader files and `mcp-adapter.php` are listed, and the exclusion check exits successfully.

### Task 2: Point WordPress.com at the advanced workflow

**Files:**
- Modify: WordPress.com GitHub Deployment configuration for `hperkins.blog` only.

**Interfaces:**
- Consumes: successful `Publish Website` runs from `henryperkins/mcp-adapter`, branch `trunk`, workflow `.github/workflows/wpcom.yml`, artifact `wpcom`.
- Produces: deployed MCP Adapter runtime files under the existing WordPress.com plugin deployment.

- [ ] **Step 1: Open the existing MCP Adapter GitHub Deployment**

In the WordPress.com dashboard for `hperkins.blog`, select the deployment connected to `henryperkins/mcp-adapter`. Confirm the repository is `henryperkins/mcp-adapter` and the deployment branch is `trunk`; do not edit the HPerkins Tokens theme deployment.

- [ ] **Step 2: Enable advanced deployment**

Switch the MCP Adapter deployment to advanced mode and select `.github/workflows/wpcom.yml` / `Publish Website`. Save the deployment settings without changing the repository or branch.

- [ ] **Step 3: Confirm WordPress.com consumes the successful artifact**

Wait for the GitHub push-triggered run and WordPress.com deployment to complete. In the WordPress.com activity log, require a successful GitHub deployment event for MCP Adapter after the workflow commit time.

- [ ] **Step 4: Verify plugin health**

Confirm through WordPress.com plugin state and site diagnostics that:

- MCP Adapter remains active.
- The missing Composer autoloader notice is absent.
- `wp mcp-adapter list` completes successfully.
- The default MCP Adapter server is registered.
- Flavor Agent's MCP server is registered while Flavor Agent is active.
- An authenticated MCP discovery request succeeds.

- [ ] **Step 5: Stop safely on any failed acceptance check**

Do not deactivate unrelated plugins or modify site content. If the new artifact fails, restore the last known-good deployment or install the official stable MCP Adapter release package, then report the exact failed gate and observed evidence.
