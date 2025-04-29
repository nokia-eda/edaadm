# The EDA-Asset Shipyard

---
- [The EDA-Asset Shipyard](#the-eda-asset-shipyard)
  - [Customize the package:](#customize-the-package)
    - [Customize the passwords:](#customize-the-passwords)
    - [Make options:](#make-options)
    - [List KPT setters to customize defaults:](#list-kpt-setters-to-customize-defaults)
      - [Authentication parameters](#authentication-parameters)
  - [Apply the package](#apply-the-package)
    - [Set the `kubeconfig` for the asset cluster](#set-the-kubeconfig-for-the-asset-cluster)
    - [Install all components](#install-all-components)
    - [How to reach the shipyard](#how-to-reach-the-shipyard)
- [Appendix](#appendix)
  - [Install each component separately](#install-each-component-separately)

---

This package boots up the following components on a given kubernetes cluster:

* Image Registry to acts as a pull mirror
* Git Server to act as an upstream mirror for git repositories
* Web Server to act as an upstream mirror for binary artifacts

---

## Customize the package:

---

### Customize the passwords:

```shell
make eda-configure-shipyard \
ASSET_HOST_GIT_USERNAME=<base64 encoded username> \
ASSET_HOST_GIT_PASSWORD=<base64 encoded password> \
ASSET_HOST_ARTIFACTS_HTPASSWORD=<htpasswd encoded string>
```

---

### Make options:

* Tools:
  * If the tools are not found in `$PATH` then the Makefile errors out
  * `KPT` path of the kpt binary
  * `KUBECTL` path of the kubectl binary

---

### List KPT setters to customize defaults:

```shell
make ls-setters
```
or
```shell
kpt fn eval --image gcr.io/kpt-fn/list-setters:v0.1.0 --truncate-output=false |& grep -v -e '^\[RUNNING\].*$' -e'^\[PASS\].*$' -e'\ *Results\:.*$' | awk '{$1=$1;print}'
```
will list:
```yaml
[info]: Name: EDA_ASSET_REGISTRY_PVC, Value: 10Gi, Type: str, Count: 1
[info]: Name: EDA_NAMESPACE_ASSET_HOST, Value: eda-assets, Type: str, Count: 14
[info]: Name: GIT_SVC_TYPE, Value: ClusterIP, Type: str, Count: 1
[info]: Name: GOGS_ADMIN_PASS, Value: ZWRhCg==, Type: str, Count: 1
[info]: Name: GOGS_ADMIN_USER, Value: ZWRhCg==, Type: str, Count: 1
[info]: Name: GOGS_IMG_TAG, Value: ghcr.io/nokia-eda/adm/gogs:0.13.0, Type: str, Count: 1
[info]: Name: GOGS_PV_CLAIM_SIZE, Value: 24Gi, Type: str, Count: 1
[info]: Name: LIGHTTPD_EDA_HTPASSWD, Value: ZWRhOiRhcHIxJE15RHNDZE1CJDBVRWNEc2RHWG9xaFdTZW5VVTBNSjEK, Type: str, Count: 1
[info]: Name: REPOS_TOCREATE, Value: catalog playground kpt connect-k8s-helm-charts, Type: str, Count: 1
```

> Use [kpt-setter](https://catalog.kpt.dev/apply-setters/v0.1/) to update these values

#### Authentication parameters

* Git server:
  * `GOGS_ADMIN_USER` - `echo -n "username" | base64`
  * GOGS_ADMIN_PASS - `echo -n "password" | base64`
* Web Server:
  * `LIGHTTPD_EDA_HTPASSWD`
    * See [nginx-basic auth](https://kubernetes.github.io/ingress-nginx/examples/auth/basic/#basic-authentication)
    * This is a base64 encoded htpasswd digest.

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
cd pt
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

`make uninstall-*` targets similarly exist for each component.