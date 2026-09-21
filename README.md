# Template for deploying K8s Cluster

This repository contains the template for deploying a K8s cluster on a MacBook. Use this repository template to create a new repository and follow the instructions below to deploy a K8s cluster.

It is designed to use the Docker Kubernetes deployed from Docker Dashboard but can be used with any Kubernetes cluster.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) (required for local cluster)
- [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) (required by various scripts)
- [Flux](https://fluxcd.io/docs/installation/) (required by various scripts)
- [vault cli](https://www.vaultproject.io/docs/install) (required by various scripts)
- [jq](https://stedolan.github.io/jq/download/) (required by various scripts)
- [yq](https://mikefarah.gitbook.io/yq/) (required by various scripts)
- [openssl](https://www.openssl.org/source/) (required to generate cluster certificate)
- [direnv](https://direnv.net/docs/installation.html) (optional)

## Setup

Once you have created your own configuration repository using this template, you will need to update the `.envrc` file with the correct values for your environment.
Replace `...` occurences with with correct values.

Also edit `resources/github-sample.sh` and `resources/newrelic-sample.sh`. Rename these files to `-secrets.sh` and edit to add your GitHub PAT tokens and NewRelic account information. Files with `-secrets.sh` suffix will not be committed to your repository.

You will need a PAT token with read access to the `mac-k8s` repository and your own copy of this repository and another one with write access to your copy of this repository.

Start or reset the Kubernetes cluster using the Docker Dashboard then change into your configuration repository. Do `direnv allow` to source the `.envrc` file and then run the `setup.sh` script.

## Deploy

Once Flux has deployed the cluster

## Destroy

To destroy the cluster run the `reset.sh` script. This will destroy the K8s cluster.
