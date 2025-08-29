# eShop .NET OpenShift Migration and Pipeline Demo

This demo is based on [Red Hat OpenShift](https://www.redhat.com/en/technologies/cloud-computing/openshift) and uses the [eShop Reference Application](https://github.com/dotnet/eShop) from the upstream .NET community as an application base.

## Purpose of this Demo

The migration and pipeline demo performs the following:

1. Containerize the application so that it can be migrated to OpenShift.  Required changes can be found in the [container-base branch](https://github.com/na-launch/eshop-dotnet-pipeline-demo/tree/container-base).
2. Build the eShop application, tag the resulting image as `dev` and promote the tag to `qa` with OpenShift Pipelines.
3. Deploy both the Development and QA versions of the application with OpenShift GitOps.
4. Detect image updates for `dev` and `qa` tags with ArgoCD Image Updater.
5. Based on a source code push, Pipelines as Code will automatically run the respective pipelines from step 2 and report the status.

## Setup

This demo assumes the following have been set up:
- Red Hat OpenShift (tested on 4.19)
  - OpenShift Pipelines (tested on 1.19) with Pipelines as Code
  - OpenShift GitOps (tested on 1.17) with [ArgoCD Image Updater (community version)](https://argocd-image-updater.readthedocs.io/)
- Code repository hosted on GitHub
- Image repository hosted on Quay.io

For the community version of ArgoCD Image Updater, we deployed into the `openshift-gitops` namespace:
```
oc apply -n openshift-gitops -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/v0.16.0/manifests/install.yaml
```
Fork this repository to your own GitHub repository which will be used for the application code base.

## The Pipeline

### Build and Promote

For this demo, we configured authentication to the repositories with a [GitHub personal access token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens) and [Quay.io robot account](https://docs.redhat.com/en/documentation/red_hat_quay/3.15/html/use_red_hat_quay/allow-robot-access-user-repo).  Replace the contents of [github-secret.yaml](/manifests/secrets/github-secret.yaml) and [quay-secret.yaml](/manifests/secrets/quay-secret.yaml) with your own (the secrets in this repository are just dummy values).

In the [build](manifests/build/eshop-dotnet-build.pipeline.yaml) and [promote](manifests/build/eshop-dotnet-promote.pipeline.yaml) pipelines, replace the value of the `IMAGE` parameter with your own image.  Also in the [base deployment](manifests/base/deployment.yaml), replace the value of the container image to be used with your own.  Then deploy the pipelines, which can be run manually:
```
oc apply -k manifests/build
```

Pipelines as Code is an enhancement from manual pipeline runs by automating pipeline runs based on source code push.  To set this up, [configure a GitHub app](https://docs.redhat.com/en/documentation/red_hat_openshift_pipelines/1.19/html/pipelines_as_code/using-pipelines-as-code-repos#pac-configuring-github-app-manually_using-pipelines-as-code-repos) for Pipelines as Code.  In the [eshop-dev-repository.yaml](manifests/pipelinesascode/eshop-dev-repository.yaml) definition, update the URL with your own Git fork then apply:
```
oc apply -k manifests/pipelinesascode
```

### Deploy

The following applications manage the Development and QA versions:
- [Development application](manifests/application/eshop-dotnet-dev.application.yaml) in `eshop-dotnet-dev` namespace
- [QA application](manifests/application/eshop-dotnet-qa.application.yaml) in `eshop-dotnet-qa` namespace, also in the same cluster

Replace the image in `argocd-image-updater.argoproj.io/image-list` with your own, then deploy the ArgoCD applications:
```
oc apply -k manifests/application
```

Image Updater detects updates to the image digest for the `dev` and `qa` tags and deploys the latest image via ArgoCD.

```
oc apply -k manifests/imageupdater
