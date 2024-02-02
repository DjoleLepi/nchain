# Project

This document explains how FluxCD was bootstrapped and wordpress configured to deploy latest image updates. The instructions for the setup were as follows:

1. Create a clean Kubernetes cluster (k3s/Rancher desktop/minikube is fine).
1. You should use FluxCD for GitOps and Helm for deploying WordPress.
1. Automate the process of image updates for WordPress using FluxCD's automated sync mechanism.
1. Write documentation to explain your setup and deployment process.

## FluxCD

### Installation

The installation of FluxCD is described on this page: [Flux Installation](https://fluxcd.io/flux/installation/).

The following steps describe how Flux was installed and configured in this repository

1. [Install CLI](https://fluxcd.io/flux/installation/#install-the-flux-cli)

    ```bash
    ~ curl -s https://fluxcd.io/install.sh | sudo bash
    ```

1. [Bootstrap on existing cluster and repo](https://fluxcd.io/flux/installation/#bootstrap-with-flux-cli)

    Bootstrapping FluxCD was done on the existing repo and on a [Rancher Desktop](https://rancherdesktop.io/) cluster.

    The method used was to authenticate to [GitHub with a PAT (Personal Access Token)](https://rancherdesktop.io/) that has admin permissions on the repository. Below is the command for how FluxCD was bootstrapped. The command prompts for the PAT to be entered.

    ```bash
    ~ flux bootstrap github \
    --token-auth \
    --owner=DjoleLepi \
    --repository=nchain \
    --branch=master \
    --path=fluxcd \
    --personal
    ```

## Wordpress

All of the manifests that are used to deploy the **bitnami/wordpres** helm chart and automatically update its image are located at [fluxcd/flux-system/app-sync.yaml](fluxcd/flux-system/app-sync.yaml). The chart used to deploy wordpress can be found [here](https://artifacthub.io/packages/helm/bitnami/wordpress).

The **HelmRepository** and **HelmRelease** resources are the [recommended way to deploy a HelmChart](https://fluxcd.io/flux/guides/helmreleases/#helm-repository). Aditionally the `app-sync.yaml` was configured to be tracked by FluxCD at [fluxcd/flux-system/kustomization.yaml](fluxcd/flux-system/kustomization.yaml)

**HelmRepository** defines where specific charts can be located and **HelmRelease** defines which chart and config should be deployed (with a reference to the HelmRepository).

## Image updater configuration

The automation of FluxCD image updates to git is described on this page: [FluxCD - Automate image updates to Git](https://fluxcd.io/flux/guides/image-update).

Additional components need to be installed to allow FluxCD to automate image updates on Git: **image-reflector-controller** and **image-automation-controller**. The manifests can be generated with the following command, and then added to the existing manifests:

```bash
~ flux install \
--components-extra image-reflector-controller,image-automation-controller \
--export > ./fluxcd/flux-system/gotk-components.yaml
```

The **ImageRepository** and **ImagePolicy** resources are required for [image tag scanning](https://fluxcd.io/flux/guides/image-update/#configure-image-scanning). **ImageRepository** defines where the image with the latest tags can be found and **ImagePolicy** defines which constraints we want to add to the tags that will be used for our image updater (e.g. `range: '>=6.0.0 <7.0.0'`).

The [**ImageUpdateAutomation**](https://fluxcd.io/flux/guides/image-update/#configure-image-updates) resource configures how and where in the git repository the latest update changes will be commited.

The `# {"$imagepolicy": "flux-system:wordpress:tag"}` image policy marker is used for FluxCD to find and replace the image tag located on the **HelmRelease** values. The configuration is described on this page: [Configure image update for custom resources](https://fluxcd.io/flux/guides/image-update/#configure-image-update-for-custom-resources).
