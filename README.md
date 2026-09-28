# Template for deploying K8s Cluster

This repository contains the template for deploying a K8s cluster on a MacBook. Use this repository template to create a new repository and follow the instructions below to deploy a K8s cluster.

It can use the Docker Kubernetes cluster deployed from the Docker Dashboard, or create a [kind](https://kind.sigs.k8s.io/) cluster. The scripts and shared configuration are in the [mac-k8s](https://github.com/pc-dae-gitops/mac-k8s) repository, which should be cloned alongside your configuration repository.

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
- [OpenShift Local (crc)](https://crc.dev/docs/installing/) and [oc](https://docs.openshift.com/container-platform/latest/cli_reference/openshift_cli/getting-started-cli.html) (optional, required for OpenShift Local clusters)
- [Helm](https://helm.sh/docs/intro/install/) (optional, required for kind clusters using the Cilium or Calico CNI)
- [direnv](https://direnv.net/docs/installation.html) (optional)
- envsubst-a8m - See below (required to parse template files)

```bash
curl -L https://github.com/a8m/envsubst/releases/download/v1.4.3/envsubst-`uname -s`-`uname -m` -o envsubst
chmod +x envsubst
sudo mv envsubst /usr/local/bin
```

## Setup

Once you have created your own configuration repository using this template, you will need to update the `.envrc` file with the correct values for your environment.
Replace `...` occurrences with the correct values.

### Secrets

The `.envrc` file is committed to your repository, so it must never contain tokens or passwords. Set these in your bash profile, e.g. `~/.bash_profile`, instead.

The `setup.sh` script assumes you have two GitHub fine-grained PAT tokens.

```bash
export GITHUB_TOKEN_GITOPS_WRITE=...
export GITHUB_TOKEN_GITOPS_READ=...
```

The `GITHUB_TOKEN_GITOPS_WRITE` token should have write access to your configuration repository, i.e. the repository you create from this template.
The `GITHUB_TOKEN_GITOPS_READ` needs read access for the `mac-k8s` repository and your configuration repository.

Kind clusters use a local pull-through cache for Docker Hub. Docker Hub credentials are optional but give a higher pull rate limit. The `.envrc` sets `DOCKERHUB_USER` from `DOCKERREGISTRY` and `DOCKERHUB_TOKEN` from a `DOCKERHUB_CREDS` variable, so set a Docker Hub [personal access token](https://docs.docker.com/security/for-developers/access-tokens/) with read-only access in your bash profile.

```bash
export DOCKERHUB_CREDS=...
```

Other secrets are loaded into Vault by `secrets.sh` from JSON files in `resources/secrets`, the file path is the Vault secret name. These files are passed through `envsubst`, so reference environment variables set in your bash profile rather than putting secret values in them.

### DNS

Ingress host names are subdomains of `local_dns`, set in `.envrc`, which defaults to `kubernetes.local.internal`. Add the host names you use to `/etc/hosts`, e.g.

```text
127.0.0.1        vault.kubernetes.local.internal grafana.kubernetes.local.internal
```

`setup.sh` warns if `vault.${local_dns}` does not resolve.

### Certificate Authority

`setup.sh` first creates a CA certificate in `resources/CA.cer`, with its key in `resources/CA.key`, using `ca-cert.sh`. It adds the CA certificate to the macOS System keychain as a trusted root, prompting for an admin user name and password.

The CA is used by cert-manager to issue ingress certificates and, for kind clusters, as the Kubernetes cluster CA, so the API server and kubelet certificates are trusted too. The CA certificate is committed to your repository but the key is not, keep a secure backup of `resources/CA.key` if you want to reuse the CA on another machine.

## Deploy

Change into your configuration repository and do `direnv allow` to source the `.envrc` file.

### Docker Kubernetes

Start or reset the Kubernetes cluster using the Docker Dashboard, then run the `setup.sh` script.

### OpenShift Local (crc)

Start the cluster using `crc start` and log in as `kubeadmin`, e.g. `eval $(crc oc-env)` then `oc login -u kubeadmin https://api.crc.testing:6443`, then run the `setup.sh` script. `setup.sh` detects an OpenShift Local cluster when the current context's api server is `https://api.crc.testing:6443` and runs `crc-setup.sh`, which deploys the OpenShift equivalents of the core components using `resources/flux-crc.yaml` in the mac-k8s repository, see the mac-k8s README.

For OpenShift Local clusters, `crc-setup.sh` configures OVN-Kubernetes to route egress traffic via the host network stack, setting `routingViaHost` in the cluster network operator configuration, if not already set, and waits for the network operator to apply the change.

Docker Kubernetes and OpenShift Local both bind ports 80 and 443 on the host, so only run one of them at a time.

### Kind

Run `setup.sh --kind` to create a kind cluster and deploy to it. If a kind cluster named `$CLUSTER_NAME` already exists it is used rather than recreated. The cluster can also be created on its own using `kind-cluster.sh`.

Kind clusters are configured using the following environment variables, set in `.envrc`. See `kind-cluster.sh --help` for details.

| Variable | Description | Default |
| --- | --- | --- |
| `KIND_K8S_VERSION` | Kubernetes version, must be a [node image](https://github.com/kubernetes-sigs/kind/releases) supported by the installed kind release | kind release default |
| `KIND_CNI` | `kindnet`, `cilium` or `calico` | `kindnet` |
| `KIND_EXTRAS` | Optional features, see below | `ca metrics registry mirror` |
| `KIND_HTTP_PORT`, `KIND_HTTPS_PORT` | Host ports for the ingress controller | `80` and `443` |
| `KIND_DATA_DIR` | Host directory for persistent volumes and audit logs | `~/.kind/<cluster name>` |

The available extras are:

| Extra | Description |
| --- | --- |
| `ca` | Uses `resources/CA.cer` as the Kubernetes cluster CA |
| `metrics` | Exposes control plane metrics (etcd, controller manager, scheduler, kube-proxy) and uses kubelet serving certificates signed by the cluster CA, e.g. for OpenTelemetry collector kubeletstats and prometheus receivers |
| `registry` | Local image registry, push images to `localhost:5001` |
| `mirror` | Pull-through caches for docker.io, registry.k8s.io, ghcr.io and quay.io, so images are cached when clusters are recreated |
| `audit` | Kubernetes API audit logging to `$KIND_DATA_DIR/audit` |

The kind configuration is the `resources/kind.yaml` file in the mac-k8s repository. To change it, copy it to `resources/kind.yaml` in your configuration repository and edit it, it is passed through `envsubst` so it can reference environment variables. Extras can be overridden or added in the same way, in `resources/kind-extras`.

### Local OpenShift CRC Deployment

For a local CRC cluster [instructions](https://crc.dev/docs/using/) to deploy a local OpenShift cluster.

> Before running `setup.sh`, set following configuration setting...
> ```bash
> crc config set cpus 8
> crc config set memory 20000
> crc config set disk-size 200
> crc config set enable-cluster-monitoring true
> crc start
> ```

Then use the `oc login` command to login to the cluster. You can automate this process by adding the password to your `~/.bash_profile` file. After doing so you need to source the `~/.bash_profile` or set perform the export command in your shell.

```bash
export CRC_PASSWORD=...
oc login -u kubeadmin -p ${CRC_PASSWORD}  https://api.crc.testing:6443
```

### Deployed components

The `setup.sh` script deploys Flux, which deploys core utilities: Kyverno, cert-manager, ingress-nginx, Vault, External Secrets, Reloader, Secrets Store CSI driver, metrics-server and kube-state-metrics. It then initialises and unseals Vault and loads secrets.

Addons, namespaces and applications are deployed by listing them in files in `resource-descriptions`:

- `addons.yaml`, addons from the mac-k8s `local-cluster/addons` directory, e.g. grafana, loki, tempo, otel-collector or newrelic

  ```yaml
  addons:
    - name: grafana
  ```

- `namespaces.yaml`, namespaces to create
- `apps.yaml`, applications to deploy

## Destroy

For Docker Kubernetes, reset the Kubernetes cluster using the Docker Dashboard.

For kind, run `kind-cluster.sh --delete`. The local registry and mirror containers are retained so that cached images can be reused, remove them using `docker rm -f kind-registry $(docker ps -aq --filter name=kind-mirror-)`.

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
| Document Status | REVIEWED |
| Human Oversight Record | reviewed |
| Personal Data Flag | No personal data. |
| Intended Audience | Public, users of this repository template |
| Known Limitations | Tested with kind v0.33.0 on macOS. Docker Kubernetes behaviour of the new kind related changes has not been retested. |
| Confidentiality Classification | PUBLIC |
| Version | v0.2 |

### Input Document Register

| Ref | Document Title | Document Type | Author / Source | Date | Classification | How Used |
| --- | --- | --- | --- | --- | --- | --- |
| IDR-001 | Previous README.md | Report | Paul Carlton | Undated | Public | Base content, updated. |
| IDR-002 | mac-k8s scripts and resources | Code File | Paul Carlton, pc-dae-gitops/mac-k8s | 28 September 2026 | Public | Source of setup, kind and secrets behaviour described. |
| IDR-003 | Claude Code session | Conversation | Paul Carlton, Claude | 28 September 2026 | Internal | Requirements for kind support, secrets handling and prerequisites. |
| IDR-004 | kind documentation and v0.33.0 release notes | Web Page | <https://kind.sigs.k8s.io/>, <https://github.com/kubernetes-sigs/kind/releases> | 26 August 2026 | Public | Kind installation and node image references. |
