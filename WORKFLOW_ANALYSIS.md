# Workflow Analysis

This document analyzes the `.github/workflows/deploy.yml` file used to automatically test and deploy the TechFlow Solutions website.

## 1. What triggers this workflow to run?

The workflow is triggered by two events, defined in the `on:` section:
- A **push** to the `main` branch
- A **pull request** targeting the `main` branch

This means the workflow runs both when changes are merged directly into `main`, and whenever someone opens or updates a pull request aimed at `main` (which lets the team catch problems before merging, not just after).

## 2. What are the four main steps this workflow performs?

The four main steps happen in the `build-and-test` job:
1. **Checkout code** — downloads the repository's files so the workflow can access them
2. **Validate HTML** — runs an HTML validator against the site's files to catch markup errors
3. **Check links** — scans for broken links in the project
4. **Upload artifact** — packages the site's files and uploads them so the `deploy` job can use them

(A fifth step, **Deploy to GitHub Pages**, happens afterward in the separate `deploy` job, but only once the four steps above succeed.)

## 3. What does the "Checkout code" step do and why is it necessary?

The `Checkout code` step (using `actions/checkout@v4`) downloads a copy of the repository's code into the GitHub Actions virtual machine that runs the workflow. It's necessary because GitHub Actions runners start out empty — they don't automatically have access to the repository's files. Every later step (validating HTML, checking links, uploading the site) depends on the code actually being present first, so this step has to run before anything else can work.

## 4. What is the purpose of the environment configuration?

The `environment` block in the `deploy` job:
```yaml
environment:
  name: github-pages
  url: ${{ steps.deployment.outputs.page_url }}
```
tells GitHub that this job deploys to the `github-pages` environment, and it captures the live URL of the deployed site as an output. This lets GitHub track deployments to that environment (showing deployment history and status directly in the repo), and it's also what allows the deployment to use the `pages: write` and `id-token: write` permissions needed to actually publish to GitHub Pages.

## 5. How does this automated deployment improve reliability compared to manual deployment?

With manual deployment, a developer might forget a step, skip testing before pushing live, or deploy code that still has broken HTML or dead links — all human errors that are easy to make when repeating the same steps by hand. This workflow removes that risk by always running the same steps in the same order every time: it validates the HTML, checks links, and only deploys if the `build-and-test` job succeeds (`deploy` has a `needs: build-and-test` dependency). This means broken code can never accidentally reach the live site, deployments happen consistently regardless of who pushes the code or what time it is, and there is a clear, reviewable log of every deployment attempt in the Actions tab.

## 6. What would happen if you pushed code to a different branch (not main)?

Nothing would deploy. The `on:` section only triggers this workflow for pushes to `main` (or pull requests targeting `main`), so pushing to any other branch (e.g. a `feature/...` branch) wouldn't even trigger the workflow to run at all as a direct push event.

The one exception is if that branch is the source of an open pull request into `main` — in that case, the `pull_request` trigger would run the `build-and-test` job (validation and link checking) so the team can see if the changes are safe *before* merging. However, even then, the `deploy` job explicitly checks `if: github.event_name == 'push' && github.ref == 'refs/heads/main'`, so it would still refuse to deploy — deployment only ever happens after code actually lands on `main` via a push (including a merge).
