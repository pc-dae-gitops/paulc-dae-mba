CONFIDENTIALITY: PUBLIC
STATUS: DRAFT - UNREVIEWED

# paulc-dae-mba K8s Cluster Configuration

This repository contains the configuration for a K8s cluster on Paul Carlton's MacBook, created from the [mac-k8s-template](https://github.com/pc-dae-gitops/mac-k8s-template) repository. The scripts and shared configuration are in the [mac-k8s](https://github.com/pc-dae-gitops/mac-k8s) repository, which must be cloned alongside this repository.

See the [mac-k8s-template README](https://github.com/pc-dae-gitops/mac-k8s-template#readme) for full details of the setup, kind options and deployed components.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) (required for local cluster)
- [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) (required by various scripts)
- [Flux](https://fluxcd.io/docs/installation/) (required by various scripts)
- [vault cli](https://www.vaultproject.io/docs/install) (required by various scripts)
- [jq](https://stedolan.github.io/jq/download/) (required by various scripts)
- [yq](https://mikefarah.gitbook.io/yq/) v4 (required by various scripts)
- [envsubst](https://www.gnu.org/software/gettext/manual/html_node/envsubst-Invocation.html) (required by various scripts, `brew install gettext`)
- [openssl](https://www.openssl.org/source/) (required to generate cluster certificate)
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) (optional, required for the `--kind` option, `brew install kind`)
- [Helm](https://helm.sh/docs/intro/install/) (optional, required for kind clusters using the Cilium or Calico CNI, as configured in this repository)
- [direnv](https://direnv.net/docs/installation.html) (optional)

## Configuration

The `.envrc` file configures this cluster. It is committed to this public repository, so it must never contain tokens or passwords.

This repository is configured to use a kind cluster:

| Setting | Value |
| --- | --- |
| `KIND_K8S_VERSION` | `v1.34.0` |
| `KIND_CNI` | `cilium` |
| `KIND_EXTRAS` | `ca metrics registry mirror audit` |
| `local_dns` | `kubernetes.local.internal` |

## Secrets

Set the following in your bash profile, e.g. `~/.bash_profile`, not in `.envrc`.

```bash
export GITHUB_TOKEN_GITOPS_WRITE=...
export GITHUB_TOKEN_GITOPS_READ=...
export DOCKERHUB_CREDS=...
```

- `GITHUB_TOKEN_GITOPS_WRITE`, a GitHub fine-grained PAT token with write access to this repository.
- `GITHUB_TOKEN_GITOPS_READ`, a GitHub fine-grained PAT token with read access to the `mac-k8s` repository and this repository.
- `DOCKERHUB_CREDS`, optional, a Docker Hub personal access token with read-only access. `.envrc` uses it to set `DOCKERHUB_TOKEN` for the kind docker.io pull-through cache, with `DOCKERHUB_USER` set from `DOCKERREGISTRY`, giving a higher pull rate limit.

## DNS

Add the ingress host names to `/etc/hosts`, e.g.

```text
127.0.0.1        vault.kubernetes.local.internal grafana.kubernetes.local.internal
```

## Deploy

Change into this repository and do `direnv allow` to source the `.envrc` file, then run `setup.sh --kind`.

`setup.sh` first creates the CA certificate in `resources/CA.cer` if it does not exist, which is used as the kind cluster CA and by cert-manager. The CA key, `resources/CA.key`, is not committed to this repository.

To use Docker Kubernetes instead, start or reset the Kubernetes cluster using the Docker Dashboard and run `setup.sh` without the `--kind` option.

## Destroy

Run `kind-cluster.sh --delete` to delete the kind cluster. For Docker Kubernetes, reset the Kubernetes cluster using the Docker Dashboard.

---

## Document Provenance

### AI Generation Disclosure

| Field | Value |
| --- | --- |
| AI Involvement | Co-authored |
| AI Model | Claude Opus 5.5 (claude-opus-5-5) |
| AI Platform | Claude Code, VS Code extension (Anthropic) |
| Human Accountable | Paul Carlton |
| Date of Generation | 28 September 2026 |
| Document Status | DRAFT - UNREVIEWED |
| Human Oversight Record | Unreviewed |
| Personal Data Flag | Contains the repository owner's name only, lawful basis legitimate interests (authorship attribution). |
| Intended Audience | Public, the repository owner and readers of this repository |
| Known Limitations | Configuration table reflects `.envrc` at the time of writing and must be kept in step with it. |
| Confidentiality Classification | PUBLIC |
| Version | v0.2 |

### Input Document Register

| Ref | Document Title | Document Type | Author / Source | Date | Classification | How Used |
| --- | --- | --- | --- | --- | --- | --- |
| IDR-001 | Previous README.md | Report | Paul Carlton | Undated | Public | Base content, updated. |
| IDR-002 | .envrc | Code File | Paul Carlton | 28 September 2026 | Public | Source of the configuration and secrets variables described. |
| IDR-003 | mac-k8s scripts and resources | Code File | Paul Carlton, pc-dae-gitops/mac-k8s | 28 September 2026 | Public | Source of setup and kind behaviour described. |
| IDR-004 | Claude Code session | Conversation | Paul Carlton, Claude | 28 September 2026 | Internal | Requirements for secrets handling and prerequisites. |
