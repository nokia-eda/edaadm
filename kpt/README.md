# The EDA-Asset Shipyard

---
- [The EDA-Asset Shipyard](#the-eda-asset-shipyard)
  - [Customize the package:](#customize-the-package)
    - [Make options:](#make-options)
  - [Apply the package](#apply-the-package)
    - [Set the `kubeconfig` for the asset cluster](#set-the-kubeconfig-for-the-asset-cluster)
    - [Install all components](#install-all-components)
    - [How to reach the shipyard](#how-to-reach-the-shipyard)
- [Appendix](#appendix)
  - [Install each component separately](#install-each-component-separately)
  - [Uninstall the shipyard](#uninstall-the-shipyard)

---

This package boots up the following components on a given kubernetes cluster:

* Image Registry to acts as a pull mirror
* Git Server to act as an upstream mirror for git repositories
* Web Server to act as an upstream mirror for binary artifacts

---

## Customize the package:

* View available setters `make ls-setters`
* Setters file is located at [configs/kpt-setters.yaml](./configs/kpt-setters.yaml)
* A custom one can be specified using the make variable: `KPT_SETTERS`
* Git server:
  * `B64_ASSET_HOST_GIT_USERNAME` - `echo -n "username" | base64`
  * `B64_ASSET_HOST_GIT_PASSWORD` - `echo -n "password" | base64`
* Web Server:
  * `B64_ASSET_HOST_ARTIFACTS_HTPASSWORD`
    * See [nginx-basic auth](https://kubernetes.github.io/ingress-nginx/examples/auth/basic/#basic-authentication)
    * This is a base64 encoded htpasswd digest.
    * `printf "$(htpasswd -nbB username password)" | tr -d '\n' | base64 -w0`
* Run `make eda-configure-shipyard` to apply the customizations using the setters file.

---

### Make options:

* Tools:
  * If the tools are not found in `$PATH` then the Makefile errors out
  * `KPT` path of the kpt binary
  * `KUBECTL` path of the kubectl binary

---

## Apply the package

### Set the `kubeconfig` for the asset cluster


```shell
export KUBECONFIG=eda-asset-01/kubeconfig.yaml
kubectl config get current-context # Verify it is pointing to the correct cluster
```
---

### Install all components

This will install the following components:
  * ingress
    * nginx-ingress to provide a common ingress point to the shipyard
  * storage
    * Local-path storage to provide persistence for pvc's
  * asset-shipyard
    * Shipyard k8s namespace `eda-assets`
    * Image Registry `http://<ingress-ip>/v2`
    * Git server `http://<ingress-ip>/git`
    * Web server `http://<ingress-ip>/artifacts`

```shell
cd kpt
make eda-setup-shipyard
```

### How to reach the shipyard

```bash
kubectl --kubeconfig ${KUBECONFIG} -n eda-assets get ingresses.networking.k8s.io
```

---

# Appendix

## Install each component separately

```shell
make ingress
make storage
make eda-install-asset-host
```

## Uninstall the shipyard

`make uninstall-*` targets similarly exist for each component.