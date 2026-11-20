# gamedev-publish-action

Publish an [AI Game Dev](https://ai-game.dev) package to the community registry from CI with **Trusted Publishing**
(OIDC): no publisher token in your secrets. The registry trades the CI job's short-lived OIDC token for a 15-minute
credential that can upload one version of one package, and records where it was built (provenance).

Full guide: <https://ai-game.dev/docs/wiki/app/trusted-publishing>

## 1. Add a trusted publisher

On [ai-game.dev/publisher](https://ai-game.dev/publisher), open your package → **Trusted publishing** and add your
repository, the workflow file (GitHub) or CI config path (GitLab), and an **environment** (with required reviewers) or a
protected **ref**. The workflow path alone can be edited by anyone who can push a branch. A package that was never
uploaded can be set up there too (**Set up a new package**).

## 2. GitHub Actions

```yaml
on:
  push:
    tags: ["v*"]
jobs:
  publish:
    runs-on: ubuntu-latest
    environment: registry              # the environment your trusted publisher requires
    permissions:
      contents: read
      id-token: write                  # lets the job request an OIDC token for ai-game.dev
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci --ignore-scripts && npm run build   # no dependency install scripts beside the token
      - id: publish
        uses: IvanMurzak/gamedev-publish-action@v1
        with:
          path: .
          notes: RELEASE_NOTES.md      # optional; default: the version's section of CHANGELOG.md
```

| Input | Default | What it does |
|---|---|---|
| `path` | `.` | The package folder (with `aigd-package.json`) |
| `notes` | empty | A Markdown file with this version's release notes (≤ 20 KB) |
| `wait` | `true` | Wait for the registry's checks and fail the step on a rejection; `false` returns after the upload |
| `cli-version` | `beta` | The `@baizor/gamedev` version: an exact version (recommended) or a dist-tag. Trusted Publishing needs a CLI newer than 0.32.0, in the `beta` channel until the next stable release |
| `audience` | empty | The OIDC audience; set it only for a non-production registry |
| `node-version` | `22` | The Node.js version that runs the CLI |

Output `result`: the CLI's JSON (`ok`, `id`, `version`, `state`, `integrity`, `auth`, `report`, or `error` on failure).

Pull requests never publish: only branch and tag runs do, and `pull_request_target` runs are refused.

## 3. GitLab CI

```yaml
include:
  - remote: https://raw.githubusercontent.com/IvanMurzak/gamedev-publish-action/v1/gitlab/publish-package.gitlab-ci.yml

publish-package:
  extends: .aigd-publish-package
  before_script:
    - npm ci --ignore-scripts && npm run build
  variables:
    AIGD_PACKAGE_PATH: "."
    AIGD_RELEASE_NOTES: ""             # optional Markdown file
  environment: registry
  rules:
    - if: $CI_COMMIT_TAG
```

The template declares `id_tokens: { AIGD_ID_TOKEN: { aud: ai-game.dev } }`. The registry accepts an ID token issued at
most an hour before the exchange: keep the publish job short, separate from long builds.

## Versions

`v1` moves with compatible fixes; pin a full commit SHA if you need it frozen. Sources of truth: the AI Game Dev App's
`ci/github/publish-package/action.yml` and `ci/gitlab/publish-package.gitlab-ci.yml`.
