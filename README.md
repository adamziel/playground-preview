# WordPress Playground Preview Reusable Workflow

This repository ships a [`workflow_call`](https://docs.github.com/actions/using-workflows/reusing-workflows#calling-a-reusable-workflow) workflow that adds a **WordPress Playground preview button** to pull requests. You can either keep the preview surfaced inline at the bottom of the PR description or manage a dedicated PR comment that stays up to date as the pull request evolves.

The workflow automatically builds a Playground blueprint from the pull request metadata and repository contents. Consumers can supply their own blueprint JSON, override the rendered content through templates, and inject custom variables for complex setups.

> **Status** – Designed for reuse from other repositories. Publish this repo (or copy the workflow) and reference the workflow path from your consuming workflows.

---

## Quick Start

Create a workflow in the consuming repository (for example, `.github/workflows/wp-playground-preview.yml`):

```yaml
name: PR Playground Preview

on:
  pull_request:
    types: [opened, synchronize, reopened, edited]

jobs:
  preview:
    uses: Automattic/preview-button/.github/workflows/playground-preview.yml@main
    with:
      preview-mode: description
      plugin-path: packages/my-plugin
```

This will:

- Append a managed block at the bottom of the PR body that contains a graphical “Open WordPress Playground Preview” button.
- Generate a Playground blueprint that copies `packages/my-plugin` from the pull request branch into `/wp-content/plugins/…` and activates the plugin.
- Re-run automatically whenever the pull request changes, ensuring the button and blueprint stay in sync.

Switch `preview-mode` to `comment` to manage a reusable comment instead of editing the PR body.

---

## Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `preview-mode` | `description` updates the PR body. `comment` manages a comment. | `description` |
| `playground-host` | Base Playground host URL used to compose the preview link. Trailing slashes are trimmed. | `https://playground.wordpress.net` |
| `blueprint` | Inline JSON blueprint template. Can contain `{{PLACEHOLDER}}` tokens that resolve at runtime. Overrides automatic blueprint generation. | *(auto-generated)* |
| `plugin-path` | Path (relative to repo root) to a plugin directory that should be copied into the Playground container and activated. | |
| `theme-path` | Path to a theme directory inside the repo to copy + activate. | |
| `wp-content-path` | Path to a `wp-content` directory inside the repo to import wholesale. | |
| `blueprint-variables` | JSON object containing additional variables for blueprint interpolation. | `{}` |
| `template-variables` | JSON object containing additional variables for template interpolation (description/comment). | `{}` |
| `description-template` | Custom markdown/HTML for description mode. Can reference placeholders. | Built-in template with graphical button |
| `comment-template` | Custom markdown/HTML for comment mode. Can reference placeholders. | Built-in comment with button and metadata |
| `description-marker-start` | Hidden marker used to begin the managed block inside the PR body. | `<!-- wp-playground-preview:start -->` |
| `description-marker-end` | Hidden marker used to end the managed block inside the PR body. | `<!-- wp-playground-preview:end -->` |
| `comment-identifier` | Hidden marker inserted into the managed comment so the workflow can detect/update it. | `<!-- wp-playground-preview-comment -->` |

### Blueprint shortcuts

Supplying any of `plugin-path`, `theme-path`, or `wp-content-path` generates a default blueprint that:

- Downloads the pull request branch archive from GitHub (`https://codeload.github.com/<owner>/<repo>/zip/refs/heads/<branch>`).
- Copies the referenced directory (within the extracted archive) into the appropriate location inside the Playground filesystem.
- Activates the plugin/theme via `wp-cli` if applicable.

Provide multiple paths to combine behaviours (for example, copy a plugin and a theme in one blueprint). If you set `plugin-path: .` within a repo that is itself a plugin, the entire repository root is copied into `/wp-content/plugins/{{REPO_SLUG}}` before activation.

If you pass a `blueprint` input, the shortcuts are ignored and the provided JSON is used after interpolation.

---

## Available placeholders

Placeholders resolve inside both the blueprint and the templates (case-insensitive). They can appear multiple times and are replaced prior to validation.

| Placeholder | Value |
|-------------|-------|
| `{{PR_NUMBER}}` | Numeric pull request number. |
| `{{PR_TITLE}}` | Pull request title. |
| `{{PR_HEAD_REF}}` | Source branch name (e.g. `feature/add-preview`). |
| `{{PR_HEAD_SHA}}` | Head commit SHA. |
| `{{PR_BASE_REF}}` | Base branch name. |
| `{{REPO_OWNER}}` | Repository owner. |
| `{{REPO_NAME}}` | Repository name. |
| `{{REPO_FULL_NAME}}` | `<owner>/<repo>` concatenated. |
| `{{REPO_ARCHIVE_URL}}` | Codeload URL pointing to the pull request branch archive. |
| `{{REPO_ARCHIVE_ROOT}}` | Archive root folder name (as used within the downloaded ZIP). |
| `{{REPO_SLUG}}` | Sanitised repository slug (lowercase alphanumeric + hyphen). |
| `{{PLUGIN_PATH}}`, `{{THEME_PATH}}`, `{{WP_CONTENT_PATH}}` | Paths supplied through the shortcut inputs. |
| `{{PLUGIN_SLUG}}`, `{{THEME_SLUG}}` | Slugified directory name derived from the provided path. |
| `{{PLAYGROUND_HOST}}` | Sanitised Playground host. |
| `{{PLAYGROUND_URL}}` | Fully constructed preview URL containing the inlined blueprint. |
| `{{PLAYGROUND_BLUEPRINT_JSON}}` | The final JSON blueprint (post-substitution). |
| `{{PLAYGROUND_BLUEPRINT_DATA_URL}}` | `data:application/json,…` URL that is appended to the Playground link. |
| `{{PLAYGROUND_BUTTON}}` / `{{PLAYGROUND_BUTTON_HTML}}` | The rendered graphical button HTML. |

Add custom tokens by passing JSON via `blueprint-variables` or `template-variables`. Keys are uppercased automatically, so `{"sitePath":"/var/www"}` becomes available as `{{SITEPATH}}`.

---

## Templates

- **Description mode** uses a button-first block anchored between the configured start/end markers. Removing the block from the PR body is fine—the next workflow run reinstates it.
- **Comment mode** posts a comment that begins with the configured `comment-identifier`. If someone deletes the comment, a subsequent run recreates it.

To switch from the default graphical button to a simple link, override the template, for example:

```yaml
with:
  preview-mode: comment
  comment-template: |
    🔗 [Open playground preview]({{PLAYGROUND_URL}})
    _(applies to branch {{PR_HEAD_REF}})_
```

---

## Custom blueprints

Provide your own blueprint JSON via the `blueprint` input. Multiline strings are supported with YAML block scalars. Placeholders are replaced **before** the workflow validates the JSON, allowing you to keep dynamic tokens inline.

```yaml
with:
  blueprint: |
    {
      "preferredVersions": { "php": "8.3", "wp": "nightly" },
      "steps": [
        {
          "step": "importRemoteZip",
          "url": "{{REPO_ARCHIVE_URL}}",
          "path": "/tmp/repo.zip"
        },
        {
          "step": "copyFromZip",
          "zipFile": "/tmp/repo.zip",
          "source": "{{REPO_ARCHIVE_ROOT}}/packages/block-plugin",
          "destination": "/wp-content/plugins/block-plugin"
        },
        {
          "step": "wp-cli",
          "command": "plugin activate block-plugin"
        }
      ]
    }
```

If your calling workflow pre-computes values (for example, a preview site title), serialize them into JSON and pass them in:

```yaml
    - name: Derive preview variables
      id: vars
      run: |
        title="Preview for PR ${{ github.event.pull_request.number }}"
        json=$(printf '{"site_title":"%s"}' "$title")
        echo "json=$json" >> "$GITHUB_OUTPUT"

    - name: Playground preview
      uses: Automattic/preview-button/.github/workflows/playground-preview.yml@main
      with:
        template-variables: ${{ steps.vars.outputs.json }}
```

---

## Outputs

| Output | Description |
|--------|-------------|
| `preview-url` | Final Playground URL (with inlined blueprint). |
| `blueprint-json` | Rendered blueprint JSON string. |
| `rendered-description` | Content injected into the PR description block. |
| `rendered-comment` | Content used for the managed comment. |
| `mode` | The active mode (`description` or `comment`). |
| `comment-id` | ID of the managed comment (blank in description mode). |

Reuse these outputs to feed subsequent jobs (for example, sharing the preview URL in Slack).

---

## Permissions & tokens

The workflow requires:

- `pull-requests: write` to edit the PR body.
- `issues: write` to post/update comments.
- `contents: read` to fetch repository metadata.

By default it leverages the caller workflow’s `GITHUB_TOKEN`. Supply a different token by exposing it as `github-token` in the `workflow_call` invocation.

---

## Caveats & notes

- The automatically generated blueprint assumes WordPress Playground supports the `importRemoteZip` and `copyFromZip` steps (available in current Playground builds). Provide a custom blueprint if you need older behaviour.
- Large repositories can create sizable data URLs. If you approach Playground’s URL length limits, host the blueprint JSON elsewhere and set `playground-host` to include a `blueprint-url=` parameter that points to it.
- Prefer running this reusable workflow on `pull_request` events with the `synchronize` type so the preview updates after every push.

Happy previewing!
