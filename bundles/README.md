# Bundles - EDA Cargo

---

- [Bundles - EDA Cargo](#bundles---eda-cargo)
- [Make options:](#make-options)
- [Creating boot iso for the asset talos machine:](#creating-boot-iso-for-the-asset-talos-machine)
- [Download boot-media for eda-cluster-vms](#download-boot-media-for-eda-cluster-vms)
- [Saving and Loading EDA assets](#saving-and-loading-eda-assets)
  - [Single bundle operations:](#single-bundle-operations)
  - [Bulk operations:](#bulk-operations)
    - [Save all of the bundles](#save-all-of-the-bundles)
    - [Load all of the bundles into an asset host](#load-all-of-the-bundles-into-an-asset-host)

---

# Make options:

* Required:
  * Only for load operations:
    * `ASSET_HOST` - Ingress ip: `kubectl -n eda-assets get ingresses.networking.k8s.io`
    * `ASSET_HOST_GIT_USERNAME` - base64 encoded username
    * `ASSET_HOST_GIT_PASSWORD` - base64 encoded password
    * `ASSET_HOST_ARTIFACTS_USERNAME` - base64 encoded username
    * `ASSET_HOST_ARTIFACTS_PASSWORD` - base64 encoded password
* Optional:
  * `EDAADM` - path of the edaadm binary
    * If not specified it will downloaded from [github.com/nokia-eda/edaadm](https://github.com/nokia-eda/edaadm)
  * `OUTPUT_DIR`
    * Root dir where this `Makefile` load/save assets.
    * Default is `$(pwd)/eda-cargo`
  * `EDA_CORE_REL` version of core to load/save by default.
    * `make ls-bundles` will show the default value, this is the latest released version.

---

# Creating boot iso for the asset talos machine:

To boot the asset vm in an airgapped/isolated environment which
can then be used as a mirror for artifacts for a eda cluster, we
need to prepare a boot iso/image for the given platform (nocloud/vmware) which not only boots the TALOS basic machine + also boots
the k8s controlplane and the pod images required to run mirrors for a registry, git server, web-server

1. Login to `ghcr.io` to access images using the token:
   1. `docker login ghcr.io -u <username> -p <password>`
2. Prepare the image cache `make create-assets-host-bootstrap-image-cache`
3. Create the iso:
   1. `make create-asset-vm-metal-boot-iso`
   2. `make create-asset-vm-nocloud-boot-iso`
   3. `make create-asset-vm-vmware-boot-ova`

---

# Download boot-media for eda-cluster-vms

This is the stock talos boot media that is to be used to create machines to run the *eda-core platform*

```shell
# This will download all types of boot media for all machines (vmware, nocloud, metal)
make download-talos-stock-boot-media
```
`edaadm images` lists the media download url for the given version of talos from the Talos Image Factory.
```shell
# nocloud - kvm
edaadm images --mach-type nocloud

# vmware
edaadm images --mach-type vmware

# metal
edaadm images --mach-type metal --enable-console

# curl/wget the above
```

---

# Saving and Loading EDA assets

These assets files can be used to save and load artifacts needed for eda.

The Makefile here helps automate some of the edaadm invocations, the `EDAADM`
make option is used to point to where the tool is.

It can be downloaded [here](github.com/nokia-eda/edaadm)

## Single bundle operations:

* Show available list of bundles `make ls-bundles`
* Each bundle can also be loaded individually with its `make save-<tab> <tab>` and `make save-<tab> <tab>` target
* All outputs are saved and loaded from `eda-cargo`
  * Use `OUTPUT_DIR` as a make option to override

## Bulk operations:

### Save all of the bundles
```shell
make save-all-bundles
```

### Load all of the bundles into an asset host
  * `ASSET_HOST` is the `fqdn/dns/ip` on how to reach the asset host.
  * All username/password values are base64 encoded of originals.
  * `echo -n "username" | base64`
  * `echo -n "password" | base64`
```shell
make load-all-bundles \
EDAADM=<path>/edaadm \
ASSET_HOST=eda-assets-01.local \
ASSET_HOST_GIT_USERNAME=<base64 encoded username> \
ASSET_HOST_GIT_PASSWORD=<base64 encoded password> \
ASSET_HOST_ARTIFACTS_USERNAME=<base64 encoded username> \
ASSET_HOST_ARTIFACTS_PASSWORD=<base64 encoded password>
```