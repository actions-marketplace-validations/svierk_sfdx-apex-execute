# 📜 SFDX Apex Execute

This repository implements a GitHub composite action for executing **anonymous Apex** against any kind of Salesforce org from within CI/CD automations - the piece almost every pipeline needs at some point: **post-deploy scripts**, **data migrations**, **cache invalidation**, feature-flag toggles or one-off fixes.

Scripts are executed **in the given order**, the `System.debug` output is surfaced directly in the workflow log, and a failing script fails the step with its compile problem or runtime exception instead of a silent green build.

## Usage

### Run post-deploy scripts after a deployment

```yaml
# Least-privilege token: this pipeline only reads the repository.
permissions:
  contents: read

jobs:
  deployment:
    name: Deployment
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v7.0.1
        with:
          persist-credentials: false

      - name: Install SF CLI
        uses: svierk/sfdx-cli-setup@v1.1.2

      - name: Salesforce Org Login
        uses: svierk/sfdx-login@v1.4.2
        with:
          client-id: ${{ secrets.SFDX_CONSUMER_KEY }}
          jwt-secret-key: ${{ secrets.SFDX_JWT_SECRET_KEY }}
          username: ${{ secrets.SFDX_USERNAME }}
          alias: awesome-org

      - name: Deploy Metadata
        uses: svierk/sfdx-deploy@v1.2.2
        with:
          source-dir: force-app
          target-org: awesome-org

      - name: Post-Deploy Scripts
        uses: svierk/sfdx-apex-execute@v1.0.0
        with:
          file-path: './scripts/apex/post-deploy/*.apex'
          target-org: awesome-org
```

The glob is expanded and executed in alphabetical order, so a folder of numbered scripts (`01-seed-settings.apex`, `02-migrate-accounts.apex`, ...) runs exactly in the order you named them.

### Run a single script or an inline snippet

```yaml
      - name: Data Migration
        uses: svierk/sfdx-apex-execute@v1.0.0
        with:
          file-path: './scripts/apex/migrate-accounts.apex'
          target-org: awesome-org

      - name: Invalidate Platform Cache
        uses: svierk/sfdx-apex-execute@v1.0.0
        with:
          apex-code: |
            Cache.Org.getPartition('local.Config').removeAll();
            System.debug('Org cache partition cleared');
          target-org: awesome-org
```

### Seed a scratch org and keep the debug logs

```yaml
      - name: Seed Scratch Org
        id: seed
        uses: svierk/sfdx-apex-execute@v1.0.0
        with:
          file-path: './scripts/apex/setup/*.apex,./scripts/apex/seed-data.apex'
          target-org: ${{ steps.scratch-org.outputs.username }}
          fail-fast: false
          log-output: full
          output-dir: ./logs/apex

      - name: Upload Apex Logs
        if: always()
        uses: actions/upload-artifact@v7.0.1
        with:
          name: apex-logs
          path: ./logs/apex
```

With `fail-fast: false` every script is executed even if an earlier one failed, and the step fails afterwards with the complete picture - useful for seeding, where you want to see all problems in one run rather than one per pipeline attempt.

The following actions were also used in the example workflows to create the prerequisites for the Apex execution:

- [SFDX CLI Setup](https://github.com/svierk/sfdx-cli-setup) | Installs the Salesforce CLI and related plugins
- [SFDX Login](https://github.com/svierk/sfdx-login) | Handles Salesforce login via SFDX Auth URL or the JWT flow
- [SFDX Deploy](https://github.com/svierk/sfdx-deploy) | Deploys Salesforce metadata to a target org

Of course, the Apex execute action can be used flexibly and the respective approach can vary.

## Inputs

| Name           | Required | Default | Description                                                                                                                                                                                                                                    |
| -------------- | -------- | ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `file-path`    | no\*     |         | Comma-separated list of files containing the anonymous Apex to execute, in the given order. Each entry may also be a glob pattern (e.g. `./scripts/apex/*.apex`), expanded and executed in alphabetical order. An entry matching no file fails the step. |
| `apex-code`    | no\*     |         | Anonymous Apex to execute inline. If `file-path` is set as well, the files are executed first and the inline snippet last.                                                                                                                        |
| `target-org`   | no       |         | Username or alias of the target org. Not required if the default org is set.                                                                                                                                                                     |
| `api-version`  | no       |         | Override the api version used for api requests, e.g. `59.0`.                                                                                                                                                                                     |
| `fail-fast`    | no       | `true`  | Stop at the first failing script. Set to `false` to execute all scripts and fail the step afterwards.                                                                                                                                             |
| `log-output`   | no       | `debug` | How much of the debug log to print for a **successful** execution: `debug` (only the `System.debug` output), `full` (the complete log in a collapsible group) or `none`. A failing execution always prints the error and the complete log.        |
| `output-dir`   | no       |         | Directory to write the debug log of each execution to, e.g. `./logs/apex` to upload them as a build artifact. Disabled when empty.                                                                                                                |
| `step-summary` | no       | `true`  | Write a result section to the GitHub Actions [job summary](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#adding-a-job-summary). Set to `false` to avoid collisions with a custom workflow summary.       |

\* At least one of `file-path` and `apex-code` is required - the step fails if neither is provided.

## Outputs

| Name            | Description                                                        |
| --------------- | ------------------------------------------------------------------ |
| `success`       | `true` when every executed script succeeded, otherwise `false`.    |
| `scripts-run`   | Number of scripts that were executed.                              |
| `failed-script` | The first script that failed, empty if all succeeded.              |

## Writing scripts for CI

Anonymous Apex in a pipeline runs unattended, so a few habits are worth adopting:

- **Make the scripts idempotent.** A deployment can be re-run, a workflow can be retried. Guard the changes (`if (![SELECT COUNT() FROM Setting__c] > 0) { ... }`) instead of assuming a clean org.
- **Mind the governor limits.** Anonymous Apex runs synchronously and is bound by the normal limits - 10.000 DML rows, 50.000 query rows. For larger volumes, kick off a Batch Apex job from the script, or use [sfdx-data-import](https://github.com/svierk/sfdx-data-import) for pure record loads.
- **Log what happened.** `System.debug` output is printed straight into the workflow log with the default `log-output: debug`, which is what makes a post-deploy script auditable after the fact.
- **Fail loudly.** Throw an exception (or `System.assert`) when a precondition is not met. The step then fails with the exception message and the full debug log, instead of reporting a green deployment on a half-migrated org.
- **Keep secrets out of the scripts.** If a script needs a credential, read it from a protected Custom Setting or Named Credential in the org - never interpolate a GitHub secret into the Apex source.

## Security & versioning

Every `uses:` reference in the snippets above is **pinned to an exact release tag**, e.g. `svierk/sfdx-apex-execute@v1.0.0`. Do the same in your own pipelines:

- **Never reference a mutable ref** such as `@main` or `@v1`. It runs whatever code sits behind that branch/tag at run time - with access to your org credentials - so a compromised or rewritten ref would run unnoticed.
- **Good - pin to an exact release tag** (`@v1.0.0`). Readable, concrete, and bumped through reviewed pull requests.
- **Strictest - pin to a full-length commit SHA** (`@a1b2c3d…`) with the version as a trailing comment. A SHA can never be re-pointed by the publisher; the cost is readability. Worth it for actions from publishers you don't control.
- **Enable [Dependabot](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/keeping-your-actions-up-to-date-with-dependabot) for `github-actions`** in the repository that holds your pipelines, so those pins are bumped for you instead of silently ageing.

This applies to **all** actions your workflow references - this action as well as `actions/*` and any other third-party action.

Anonymous Apex runs arbitrary code against a real org, so the rules around the calling workflow matter even more than usual:

- **Validate pull requests with `pull_request`, never `pull_request_target`** - the latter runs with the base repository's secrets, which would let a fork execute *its own* Apex against your org. This is the single most important rule for this action.
- **Gate production behind a [GitHub Environment](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)** with required reviewers, so a script that touches production data is approved by a human first.
- **Secrets travel as secrets** - hand credentials into the inputs of [sfdx-login](https://github.com/svierk/sfdx-login) as shown above. Never interpolate a secret into a `run:` script via `${{ ... }}` - that allows command injection and can leak the value into the log. This action follows the same rule internally: every input is exposed to the shell as an **environment variable** and referenced as `"$FILE_PATH"`, `"$TARGET_ORG"` and so on, and the `apex-code` input is written to a file rather than interpolated into a command line.
- **Least-privilege `GITHUB_TOKEN`** - declare a `permissions:` block granting only what the job needs. Executing Apex needs no write access at all, so `contents: read` is enough.
- **`persist-credentials: false` on checkout** - the token is not written to `.git/config`, so later steps (SF CLI, third-party actions) cannot reuse it.

## References

The Apex execution used by this GitHub composite action can be found in the Salesforce CLI Command Reference here:

- [apex run](https://developer.salesforce.com/docs/platform/salesforce-cli-reference/guide/cli_reference_apex_run.html) | Execute anonymous Apex code from the command line or a local file
- [Anonymous Blocks](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_anonymous_block.htm) | Apex Developer Guide

## Releases

Latest release notes can be found on the [release page](https://github.com/svierk/sfdx-apex-execute/releases).

## License

The scripts and documentation in this project are released under the [MIT License](https://github.com/svierk/sfdx-apex-execute/blob/main/LICENSE).
