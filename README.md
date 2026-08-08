# DevSecOps PR pipeline (ARC + Kubernetes)

Reusable GitHub Actions workflow pattern for **application repositories**. This
repo holds the pipeline definition and a **reference copy** of CI RBAC — not an
application under test.

Consumers bring their own app (`Dockerfile`, HTTP service). Cluster bootstrap
(including ARC + canonical RBAC) lives in
[`azure-rancher-k3s-lab`](../azure-rancher-k3s-lab).

On each pull request to `main` the workflow runs:

1. **SAST** — Trivy filesystem scan (`vuln` + `secret`)
2. **Build** — Kaniko Job in `ci-builds` → `ghcr.io/<owner>/<repo>:<sha>`
3. **DAST** — Ephemeral deploy of that image + OWASP ZAP baseline, then teardown

Findings show up in the **Actions job logs**.

## Repository layout

| Path | Purpose |
|------|---------|
| `.github/workflows/devsecops.yml` | PR pipeline (SAST → build → DAST) |
| `platform/ci-builds/rbac.yaml` | Reference only — lab repo is source of truth |

## Requirements

### Application repository (code under test)

- Root **`Dockerfile`** that builds a runnable image
- HTTP service on **port 3000** with healthy **`GET /`** (DAST probe + ZAP target)
- GitHub Packages (**GHCR**) enabled for push/pull

### Cluster / infra (lab repo)

From `azure-rancher-k3s-lab`, after Terraform + k3s + Rancher:

```bash
cp ansible/vars/arc.example.yml ansible/vars/arc.local.yml
# set arc_github_config_url
export ARC_GITHUB_TOKEN=ghp_...
ansible-playbook ansible/playbooks/install-arc.yml -i ansible/inventory.ini \
  -e @ansible/vars/arc.local.yml
```

That registers scale set **`arc-runner-set`** and applies `ci-builds` RBAC.
Workflows must use `runs-on: arc-runner-set`.

## How to use

1. Bring up the lab cluster and run `install-arc.yml`.
2. Copy `devsecops.yml` into the **application** repository.
3. Open a PR against `main` and watch **DevSecOps PR Pipeline**.
