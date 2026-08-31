# 💚 GitHub Cloud Runner Base Image (custom images)

## 💛 What is it?
A GitHub-hosted runner boots from a **base VM image**. "Setting up your own base image" means baking your toolchain into a **custom image** so jobs start fast, instead of installing dependencies from scratch on every run.
The first-party way to do this on GitHub's cloud is **Custom images for GitHub-hosted larger runners**. It is **Generally Available** (GA since 2026-03-26). You start from a GitHub base image, run a special workflow that **snapshots** the runner into a versioned image, then create runners that boot from it.
> Important scope: this works only with **larger runners** (a GitHub Team or Enterprise Cloud feature), not the free `ubuntu-latest` fleet.
## 💛 Why do we need it?
- **Faster cold start.** Heavy dependencies are already on the image, so a job does not spend minutes installing them every run.
- **Consistency.** Every job gets the exact same environment, pinned and reproducible.
- **Hardening.** You control precisely what is on the image, including internal certs and private tooling.
### 🤍 Real-world use case
Your CI installs a large SDK and system packages that take 4 minutes every job. You bake them into a custom image once, and jobs now start in seconds. You rebuild the image weekly so it stays patched.
## 💛 How it works (snapshot, not Packer)
GitHub does not expose Packer for this. The mechanism is a **snapshot workflow** (the `snapshot` keyword) that captures a successful runner's state into an image. Three phases:
### 🤍 Step 1: create an image-generation runner
In **Org or Enterprise settings, Actions, Runners, New GitHub-hosted runner**:
- Pick the platform (Linux x64, Linux ARM64, or Windows x64) that matches the image you want.
- Pick the **base image** to build on (a GitHub-owned clean or tooled image, or an existing custom image for layered builds).
- Tick **"Enable this runner to generate custom images."**
- Assign it to a **dedicated runner group** (GitHub recommends a separate group for image generation, as a security boundary).
### 🤍 Step 2: generate the image with a snapshot workflow
Run a workflow on that generation runner using the `snapshot` keyword. The image is produced **only if the job succeeds**.
Simple form (creates the image or adds a new version):
```yaml
jobs:
  build:
    runs-on: my-image-generation-runner
    snapshot: my-custom-image
    steps:
      - run: |
          sudo apt-get update
          sudo apt-get install -y my-heavy-toolchain
```
Controlled form (version pin plus a condition):
```yaml
jobs:
  build:
    runs-on: my-image-generation-runner
    snapshot:
      if: ${{ ! startsWith(github.ref, 'refs/tags/') }}
      image-name: my-custom-image
      version: 2.*
    steps:
      - run: ./install-tools.sh
```
Run this on a **schedule (for example weekly)** so the baked image stays fresh with security patches and tool updates.
### 🤍 Step 3: create a runner that uses the image
New GitHub-hosted runner again:
- Same platform as the generation runner.
- Under the **Custom** image tab, select your image and choose **Latest** or a pinned version.
- Choose a runner **size whose storage is at least the image size**.
- Assign a runner group.
Then target it from any workflow just by name:
```yaml
jobs:
  build:
    runs-on: my-custom-runner
```
### 🤍 Versioning
- The first build is **1.0.0**; later builds auto-increment the **minor** version (1.1.0, 1.2.0, and so on).
- `version: 2.*` keeps that major and auto-increments the minor. Patch versions are not supported.
- The most recent successful build is always tagged **latest**.
## 💛 Constraints and cost
- **Larger runners only** (Team or Enterprise Cloud). The free standard fleet cannot use custom images.
- **Platforms**: Linux x64, Linux ARM64, Windows x64. macOS custom images are not supported.
- **Billing**: jobs cost the same per-minute rate as the underlying larger runner, and **image storage is billed separately** through Actions storage, so frequent rebuilds grow cost.
- Enabling it needs an org or enterprise owner (or the CI/CD Admin role).
## 💛 Bake vs install vs cache (the real decision)
The reason to build an image at all is start-up speed. Three ways to get dependencies ready, often combined:
- **Bake into the image** (custom image above). Fastest cold start, most consistent. Costs: image storage, a build and versioning pipeline, and staleness risk (rebuild weekly). Best for heavy, slow-changing toolchains (system packages, SDKs, certs).
- **Install per-job** (`actions/setup-node`, `setup-python`, or a package install step). Zero image maintenance and always fresh, but pays install time on every run and depends on upstream registries being up. Fine for light or fast-changing deps.
- **Cache** (`actions/cache`). Middle ground: install once, restore from cache afterward. Still spends restore time and is subject to cache eviction and key misses. Good for language dependency trees (npm, pip, Go modules).
Common pattern: **bake the stable heavy layer** into the image, then **cache the fast-moving dependency layer** per job.
## 💛 Gotcha
- **Custom images are larger-runners-only.** The free `ubuntu-latest` fleet cannot use them. If you are on the standard fleet, your options are per-job install and caching.
- **It is snapshot-based, not a Dockerfile or Packer template.** You run a workflow with the `snapshot` keyword, and it captures the successful runner state. Do not go looking for an image build file.
- **Rebuild on a schedule.** A baked image goes stale as patches and tool versions land. A weekly snapshot workflow keeps it current.
- **runner-images Packer output will not boot on GitHub-hosted runners.** That path produces Azure images for self-hosting only. This is the most common point of confusion.
- **Runner storage must be at least the image size**, or the runner cannot use the image.
## 💛 References
- GitHub Docs: Use custom images (larger runners): https://docs.github.com/en/actions/how-tos/manage-runners/larger-runners/use-custom-images
- GitHub Docs: Manage larger runners: https://docs.github.com/en/actions/how-tos/manage-runners/larger-runners/manage-larger-runners
- GitHub Changelog: custom images GA (2026-03-26): https://github.blog/changelog/2026-03-26-custom-images-for-github-hosted-runners-are-now-generally-available/
- actions/runner-images (standard image source): https://github.com/actions/runner-images
- GitHub Docs: Actions Runner Controller (ARC): https://docs.github.com/en/actions/concepts/runners/actions-runner-controller
