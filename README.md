# mulzatech-terraform-planner

A reusable GitHub Actions workflow that runs `terraform plan` on a pull
request and posts the result as one comment: the plan's own one-line
headline, the full plan output, and a short AI summary at the end. Built for
the [MulzaTech DOOM on Azure](https://github.com/marcelomusza/mulzatech-doom-azure)
project, as a sibling to
[mulzatech-infra-reviewer](https://github.com/marcelomusza/mulzatech-infra-reviewer).

It **only plans.** The workflow never runs `apply`, and the Azure identity it
signs in with has read-only roles, so it couldn't even if asked.

## What the comment looks like

1. **Headline**, copied from Terraform's own `Plan: X to add, Y to change, Z
   to destroy.` line (not written by a model).
2. **Full plan output** in a collapsible block.
3. **Summary**, at the end: what will happen, and anything worth
   double-checking (replacements, role assignments, cost, exposure).

Later pushes to the same PR **edit the same comment**.

## How it works

- `.github/workflows/plan.yml` is the reusable workflow. It checks out the
  PR, signs in to Azure with OIDC (no stored secret), looks up the container
  app's *live* image so the plan shows only the PR's own change, runs
  `terraform init` and `terraform plan -lock=false`, and posts the comment.
- `plan_comment.py` builds the comment. It redacts the subscription, tenant
  and client IDs first, uses a code fence the plan text can't close, never
  silently truncates (it says so), and still posts the plan if the summary
  step fails.
- The summary model is `claude-haiku-5-5` by default (`SUMMARY_MODEL`
  overrides it). The plan is treated as untrusted input, wrapped in per-run
  random tags, because a pull request author controls the Terraform that
  produced it.

## Using it

See the caller workflow in the DOOM repo,
[`terraform-plan.yml`](https://github.com/marcelomusza/mulzatech-doom-azure/blob/main/.github/workflows/terraform-plan.yml).
It needs these repository secrets: `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`,
`AZURE_SUBSCRIPTION_ID`, and optionally `ANTHROPIC_API_KEY` (without it the
comment still posts, just without the summary).

Pin the `uses:` reference and `planner-ref` to the **same full commit SHA**.
This code runs with an Azure login, so updates should be deliberate.

### The Azure identity

Create it by hand (do not manage it from the pipeline it protects):

- An Entra app registration with a **federated credential** for the calling
  repo's `pull_request` events. No client secret.
- **Reader** on the resource group, and **Storage Blob Data Reader** on the
  Terraform state storage account. Nothing else.

## Running the comment script locally

```bash
pip install -r requirements.txt
python plan_comment.py tests/sample_plan.txt
```

Set `ANTHROPIC_API_KEY` first (use a masked prompt), or the comment is built
without the summary. The files in `tests/` are hand-written to resemble
Terraform output; they are not from a real environment.

## Limitations and risks

- **`terraform plan` evaluates the PR's Terraform**, and Terraform can run
  code at plan time (for example the `external` data source). A malicious
  same-repo PR could run code holding the read-only token and read the state.
  Fork PRs are skipped. That is acceptable for a solo repo and should be
  revisited before adding collaborators (a required-reviewer environment
  would add a manual approval).
- **State can contain sensitive attributes**, so the read identity is not
  harmless, only read-only.
- **Redaction is best effort**: it covers the three Azure IDs, any
  `/subscriptions/<guid>` path, and base64 tokens that decode to GUIDs (the
  `azurerm_client_config` data source id is one), not every possible
  identifier. A first version missed the base64 case and leaked IDs into a
  public test comment, which was deleted.
- The summary is advisory and can be wrong. The plan output above it is the
  source of truth.
