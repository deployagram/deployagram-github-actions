# deployagram-github-actions

Composite GitHub Actions that replace the CI boilerplate needed to use [Deployagram](https://deployagram.com) in a build, following the `actions/cache/save` + `actions/cache/restore` pattern: one repo, one `action.yml` per subdirectory, referenced as `deployagram/deployagram-github-actions/<name>@v1`.

| Action | Calls | Purpose |
|---|---|---|
| [`setup`](setup/action.yml) | Cognito + ECR + Docker | Authenticate, pull images, start the Collector container |
| [`mark-test-run-complete`](mark-test-run-complete/action.yml) | Collector `/run/markComplete/{container}/{testRunId}` | Mark a test run as finished |
| [`flush-to-cloud`](flush-to-cloud/action.yml) | Collector `/run/flushToCloud/{container}` | Push the captured run to Deployagram Cloud |
| [`contract-query`](contract-query/action.yml) | Collector `/run/contractQuery` | Ask the Collector to contract-check the latest run |
| [`contract-check`](contract-check/action.yml) | Deployagram Cloud `/Prod/contract/check/container` | Contract-check a build that's already been flushed |
| [`inform-deploy`](inform-deploy/action.yml) | Deployagram Cloud `/Prod/contract/deploy/container` | Tell Deployagram Cloud a version was deployed |

> **Note on `markTestRunEnd`:** the Collector's combined `/run/markTestRunEnd/{container}/{testRunId}` endpoint (used by the original hand-written workflow steps) is `@Deprecated` server-side — its own doc comment says "let's not mark complete AND flush together." This repo intentionally only exposes the two granular actions (`mark-test-run-complete` + `flush-to-cloud`); use both instead of the deprecated combined call.

## Example pipeline

```yaml
- uses: actions/checkout@v7

- uses: actions/setup-java@v6
  with:
    distribution: 'temurin'
    java-version: '21'
    cache: 'maven'

- uses: deployagram/deployagram-github-actions/setup@v1
  id: deployagram
  with:
    client-id: ${{ secrets.DEPLOYAGRAM_CLIENT_ID }}
    username: ${{ secrets.DEPLOYAGRAM_CLOUD_USERNAME }}
    password: ${{ secrets.DEPLOYAGRAM_CLOUD_PASSWORD }}
    license-key: ${{ secrets.DEPLOYAGRAM_LICENSE_KEY }}
    license: ${{ secrets.DEPLOYAGRAM_LICENSE }}
    # collector-port: '1152'   # optional, see setup/action.yml for all inputs

- name: Build with Maven
  env:
    DEPLOYAGRAM_LICENSE_KEY: ${{ secrets.DEPLOYAGRAM_LICENSE_KEY }}
    DEPLOYAGRAM_LICENSE: ${{ secrets.DEPLOYAGRAM_LICENSE }}
  run: ./mvnw -B clean package

- name: Mark test run complete
  if: ${{ success() }}
  uses: deployagram/deployagram-github-actions/mark-test-run-complete@v1
  with:
    container: BookOverflow
    test-run-id: ${{ github.run_number }}

- name: Contract query against the collector
  uses: deployagram/deployagram-github-actions/contract-query@v1
  with:
    container: BookOverflow
    test-run-id: ${{ github.run_number }}
    environment: Prod
    
- name: Flush test run to cloud
  if: ${{ success() }}
  uses: deployagram/deployagram-github-actions/flush-to-cloud@v1
  with:
    container: BookOverflow

- name: Contract check against Deployagram Cloud
  id: contract-check
  uses: deployagram/deployagram-github-actions/contract-check@v1
  with:
    token: ${{ steps.deployagram.outputs.token }}
    container: BookOverflow
    build-id: ${{ github.run_number }}
    environment: Prod

# Insert step here to really deploy your app to your Prod environment

- name: Inform Deployagram Cloud of deploy to Prod
  if: steps.contract-check.outcome == 'success'
  uses: deployagram/deployagram-github-actions/inform-deploy@v1
  with:
    token: ${{ steps.deployagram.outputs.token }}
    container: BookOverflow
    version: ${{ github.run_number }}
    environment: Prod
```

`deployagram-github-actions/setup` writes `DEPLOYAGRAM_COLLECTOR_URL` (`http://localhost:<collector-port>`) to `GITHUB_ENV`, so `mark-test-run-complete`, `flush-to-cloud`, and `contract-query` all pick up the right port automatically — no need to repeat it if you changed `collector-port` in `setup`. Override per-call with each action's own `collector-url` input if needed.

## Versioning

Tagged like `actions/setup-java`: use `@v1` to track the latest `v1.x.y` release, or pin an exact tag/SHA.

## Development

Each action is a self-contained `action.yml` (composite, `shell: bash` throughout) with no build step — edit in place, tag a release.
