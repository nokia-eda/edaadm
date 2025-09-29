ENTROPY=$(shell date +"%F-%H-%M-%S-%N")

$(info --> INFO: List of goals: $(MAKECMDGOALS))

SHELL = /usr/bin/env bash -o pipefail
.SHELLFLAGS = -ec
# Set the default target
.DEFAULT_GOAL := help

TOP_DIR := $(abspath $(dir $(abspath $(lastword $(MAKEFILE_LIST)))))
BASE=$(CURDIR)

TOOLS ?= $(CURDIR)/tools

ARCH_QUERY := $(shell uname -m)
ifeq ($(ARCH_QUERY), x86_64)
	ARCH := amd64
else ifeq ($(ARCH_QUERY),$(filter $(ARCH_QUERY), arm64 aarch64))
	ARCH := arm64
else
	ARCH := $(ARCH_QUERY)
endif

# i.e Darwin / Linux
UNAME := $(shell uname)
# Lowercase - sane version
OS := $(shell echo "$(UNAME)" | tr '[:upper:]' '[:lower:]')

EDAADM_VERSION ?= v2.0.0
EDAADM_SRC ?= https://github.com/nokia-eda/edaadm/releases/download/$(EDAADM_VERSION)/edaadm-$(OS)-$(ARCH)

# Fake location to force the user to provide it
ifdef EDAADM
EDAADM := $(realpath $(EDAADM))
else
EDAADM ?= $(TOOLS)/edaadm-$(EDAADM_VERSION)
endif

INSECURE_REGISTRY ?= 1

ARGS_INSECURE=
ifeq ($(INSECURE_REGISTRY),1)
ARGS_INSECURE += --dest-registry-insecure
endif

## Curl options:
CURL_OPTS ?= --fail --show-error --location
CURL ?= curl --silent $(CURL_OPTS)
# use the standard progress meter
CURL_PM ?= curl $(CURL_OPTS)
# use the progress bar instead of the meter - does weird things in output if there are multiple redirects/hops
CURL_PB ?= curl $(CURL_OPTS) --progress-bar

TALOS_IMAGER_IMG ?= ghcr.io/siderolabs/imager:v1.9.2

# This is the parent directory where we store/load everything for this Makefile
OUTPUT_DIR ?= $(TOP_DIR)/eda-cargo

EDA_CORE_REL ?= 25.8.2

# This is the name of files without the yaml extension
# We could have used a find expression, but best to
# manually control, that way we can control when to enable things
BUNDLE_LIST=
BUNDLE_LIST += eda-bundle-apps-24-12-1
BUNDLE_LIST += eda-bundle-apps-24-12-2
BUNDLE_LIST += eda-bundle-apps-24-12-3
BUNDLE_LIST += eda-bundle-apps-24-12-4
BUNDLE_LIST += eda-bundle-apps-25-4-1
BUNDLE_LIST += eda-bundle-apps-25-4-2
BUNDLE_LIST += eda-bundle-apps-25-4-3
BUNDLE_LIST += eda-bundle-apps-25-8-1
BUNDLE_LIST += eda-bundle-apps-25-8-2
BUNDLE_LIST += eda-bundle-connect-2-0-0
BUNDLE_LIST += eda-bundle-connect-2-0-1
BUNDLE_LIST += eda-bundle-connect-3-0-0
BUNDLE_LIST += eda-bundle-connect-3-0-1
BUNDLE_LIST += eda-bundle-connect-4-0-0
BUNDLE_LIST += eda-bundle-core-24-12-2
BUNDLE_LIST += eda-bundle-core-24-12-3
BUNDLE_LIST += eda-bundle-core-24-12-4
BUNDLE_LIST += eda-bundle-core-25-4-1
BUNDLE_LIST += eda-bundle-core-25-4-2
BUNDLE_LIST += eda-bundle-core-25-4-3
BUNDLE_LIST += eda-bundle-core-25-8-1
BUNDLE_LIST += eda-bundle-core-25-8-2
BUNDLE_LIST += eda-bundle-influxdb-exporter-1-0-0
BUNDLE_LIST += eda-bundle-kafka-exporter-2-0-1
BUNDLE_LIST += eda-bundle-kafka-exporter-3-0-0
BUNDLE_LIST += eda-bundle-kafka-exporter-4-0-0
BUNDLE_LIST += eda-bundle-metallb-0-13-7
BUNDLE_LIST += eda-bundle-nats-exporter-1-0-0
BUNDLE_LIST += eda-bundle-nats-exporter-2-0-0
BUNDLE_LIST += eda-bundle-netbox-1-0-0
BUNDLE_LIST += eda-bundle-netbox-2-0-0
BUNDLE_LIST += eda-bundle-notifier-2-0-0
BUNDLE_LIST += eda-bundle-notifier-3-0-0
BUNDLE_LIST += eda-bundle-notifier-4-0-0
BUNDLE_LIST += eda-bundle-otlp-exporter-1-0-0
BUNDLE_LIST += eda-bundle-otlp-exporter-1-1-0
BUNDLE_LIST += eda-bundle-pager-duty-1-0-0
BUNDLE_LIST += eda-bundle-pager-duty-2-0-0
BUNDLE_LIST += eda-bundle-pager-duty-3-0-0
BUNDLE_LIST += eda-bundle-prom-exporter-2-0-0
BUNDLE_LIST += eda-bundle-prom-exporter-3-0-0
BUNDLE_LIST += eda-bundle-prom-exporter-4-0-0
BUNDLE_LIST += eda-bundle-prom-exporter-4-1-0
BUNDLE_LIST += eda-bundle-remote-write-1-0-0
BUNDLE_LIST += eda-bundle-remote-write-2-0-0
BUNDLE_LIST += eda-bundle-remote-write-2-1-0
BUNDLE_LIST += eda-bundle-service-now-1-0-0
BUNDLE_LIST += eda-bundle-service-now-2-0-0
BUNDLE_LIST += eda-bundle-service-now-3-0-0
BUNDLE_LIST += eda-bundle-sim-ghcr-srlinux-24-10-1-492
BUNDLE_LIST += eda-bundle-sim-ghcr-srlinux-24-10-2-357
BUNDLE_LIST += eda-bundle-sim-ghcr-srlinux-24-10-3-201
BUNDLE_LIST += eda-bundle-sim-ghcr-srlinux-24-10-4-244
BUNDLE_LIST += eda-bundle-sim-ghcr-srlinux-24-10-5-344
BUNDLE_LIST += eda-bundle-sim-ghcr-srlinux-25-3-2-312
BUNDLE_LIST += eda-bundle-sim-ghcr-srlinux-25-7-1-187
BUNDLE_LIST += eda-bundle-sim-ghcr-srlinux-25-7-2-266
BUNDLE_LIST += eda-bundle-talos-1-9-2
BUNDLE_LIST += eda-bundle-tools-1-0-0
# Run this last
BUNDLE_LIST += eda-bundle-storage-1-0-0

# Skip core bundles based on the selected release
ifeq ($(EDA_CORE_REL),25.8.2)
SKIP_CORE_25_4_X=1
SKIP_CORE_24_12_X=1
else ifeq ($(EDA_CORE_REL),25.8.1)
SKIP_CORE_25_4_X=1
SKIP_CORE_24_12_X=1
else ifeq ($(EDA_CORE_REL),25.4.3)
eda-bundle-core-25-4-2_SKIP=1
eda-bundle-core-25-4-1_SKIP=1
SKIP_CORE_25_8_X=1
SKIP_CORE_24_12_X=1
else ifeq ($(EDA_CORE_REL),25.4.2)
eda-bundle-core-25-4-3_SKIP=1
eda-bundle-core-25-4-1_SKIP=1
SKIP_CORE_25_8_X=1
SKIP_CORE_24_12_X=1
else ifeq ($(EDA_CORE_REL),25.4.1)
eda-bundle-core-25-4-3_SKIP=1
eda-bundle-core-25-4-2_SKIP=1
SKIP_CORE_25_8_X=1
SKIP_CORE_24_12_X=1
else ifeq ($(EDA_CORE_REL),24.12.4)
eda-bundle-core-24-12-3_SKIP=1
eda-bundle-core-24-12-2_SKIP=1
SKIP_CORE_25_8_X=1
SKIP_CORE_25_4_X=1
else ifeq ($(EDA_CORE_REL),24.12.3)
eda-bundle-core-24-12-4_SKIP=1
eda-bundle-core-24-12-2_SKIP=1
SKIP_CORE_25_8_X=1
SKIP_CORE_25_4_X=1
else ifeq ($(EDA_CORE_REL),24.12.2)
eda-bundle-core-24-12-4_SKIP=1
eda-bundle-core-24-12-3_SKIP=1
SKIP_CORE_25_8_X=1
SKIP_CORE_25_4_X=1
else
$(error [ERROR] EDA_CORE_REL=$(EDA_CORE_REL) is set to an invalid version!)
endif

ifeq ($(SKIP_CORE_25_8_X),1)
eda-bundle-core-25-8-1_SKIP=1
endif

ifeq ($(SKIP_CORE_25_4_X),1)
eda-bundle-core-25-4-3_SKIP=1
eda-bundle-core-25-4-2_SKIP=1
eda-bundle-core-25-4-1_SKIP=1
endif

ifeq ($(SKIP_CORE_24_12_X),1)
eda-bundle-core-24-12-4_SKIP=1
eda-bundle-core-24-12-3_SKIP=1
eda-bundle-core-24-12-2_SKIP=1
endif


# Optional bundles to skip the default list
# eda-bundle-sim-ghcr-srlinux-24-10-1-492_SKIP=1
# eda-bundle-apps-24-12-1_SKIP=1

LIST_OF_SAVE_TARGETS=
LIST_OF_LOAD_TARGETS=

## Macros
define check-if-var-is-defined
{	\
	if [[ -z "$($1)" ]]; then echo "[ERROR] - $1 is not defined!" && exit 1; fi;\
}
endef

define does-command-exist
{	\
	location=$$(command -v $1)												;\
	if [[ $$? -ne 0 ]] ; then												 \
		echo "--> [ERROR] $1 could not be found, please use $2=<path>"		;\
		exit 1																;\
	else																	 \
		echo "--> INFO: Using $2=$${location}"								;\
	fi																		;\
}
endef

define download-bin
    $(info --> INFO: Downloading $(2))
	if test ! -f $(1); then $(CURL) -o $(1) $(2) >/dev/null && chmod a+x $(1); fi
endef

define gen-targets
ifneq ($($1_SKIP),1)
LIST_OF_SAVE_TARGETS += save-$1
endif
.PHONY: save-$1
save-$1: | $(OUTPUT_DIR) $(EDAADM)
	@{	\
		START=$$(date +%s)					;\
		echo "--> [SAVE] $1"				;\
		to=$(OUTPUT_DIR)/$(1)				;\
		mkdir -p  $$$${to}					;\
		$(EDAADM) bundle save				 \
		-f $(TOP_DIR)/$1.yaml				 \
		-t $$$${to} | sed 's/^/    /'		;\
		echo "--> [SAVE] Saved to $$$${to}"	;\
	}

ifneq ($($1_SKIP),1)
LIST_OF_LOAD_TARGETS += load-$1
endif
.PHONY: load-$1
load-$1: | $(OUTPUT_DIR) $(EDAADM) check-load-params
	@{	\
		echo "--> [LOAD] $1"							;\
		from=$(OUTPUT_DIR)/$(1)							;\
		if [[ ! -d $$$${from}  ]]; then 				 \
			echo "[ERROR] $$$${from} does not exist!"	;\
			echo "        did you run save-$(1) ?"		;\
			exit 1										;\
		fi 												;\
		$(EDAADM) bundle load \
		-f $(TOP_DIR)/$(1).yaml \
		-l $$$${from} \
		-r $(ASSET_HOST) \
		$(ARGS_INSECURE) \
		--dest-artifactserver "http://$(ASSET_HOST)/artifacts-upload" \
		--dest-artifactserver-user $(ASSET_HOST_ARTIFACTS_USERNAME) \
		--dest-artifactserver-pass $(ASSET_HOST_ARTIFACTS_PASSWORD) \
		--dest-reposerver http://$(ASSET_HOST)/git/$$$$(echo -n "$(ASSET_HOST_GIT_USERNAME)" | base64 -d) \
		--dest-reposerver-user $(ASSET_HOST_GIT_USERNAME) \
		--dest-reposerver-pass $(ASSET_HOST_GIT_PASSWORD) | sed 's/^/    /';\
		echo "--> [LOAD] Loaded from $1"				;\
	}
endef

# Generate all of the load/save targets for the bundle files
$(foreach bundle,$(BUNDLE_LIST),$(eval $(call gen-targets,$(bundle))))

## File targets
## ----------------------------------------------------------------------------|
$(OUTPUT_DIR): | $(BASE)
	@mkdir -p $(OUTPUT_DIR)

.PHONY: download-tools
download-tools: | $(TOOLS) $(EDAADM) ## Download the edaadm tool use EDAADM=<path> to override 

$(TOOLS): | $(BASE)
	mkdir -p $(TOOLS)

$(EDAADM): | $(TOOLS)
	@$(call download-bin,$(EDAADM),$(EDAADM_SRC))

## Operations:
## ----------------------------------------------------------------------------|

.PHONY: create-assets-host-bootstrap-image-cache
create-assets-host-bootstrap-image-cache: | $(OUTPUT_DIR) $(EDAADM)
	@{	\
		$(EDAADM) bundle cache-create								 \
		--output-cache-path $(OUTPUT_DIR)/eda-assets-image-cache	 \
		--assets $(TOP_DIR)/eda-assets-image-cache.yaml				;\
	}

.PHONY: does-image-cache-exist
does-image-cache-exist: | $(OUTPUT_DIR)
	@{	\
		image_cache_loc=$(OUTPUT_DIR)/eda-assets-image-cache	;\
		if [[ ! -d  "$${image_cache_loc}" ]]; then \
			echo "[ERROR] IMAGE_CACHE to embed in iso does not exist!"	;\
			echo "        did you run create-assets-host-bootstrap-image-cache";\
			exit 1														;\
		fi																;\
	}

.PHONY: pull-talos-imager-container
pull-talos-imager-container: ## Pull the image for talos imager to create iso's
	docker pull $(TALOS_IMAGER_IMG)

.PHONY: create-asset-vm-metal-boot-iso
create-asset-vm-metal-boot-iso: | $(OUTPUT_DIR) pull-talos-imager-container does-image-cache-exist ## Create the bare-metal iso for the asset-hoster
	@{	\
		to=$(OUTPUT_DIR)/talos-asset-vm-boot-imgs				;\
		image_cache_loc=$(OUTPUT_DIR)/eda-assets-image-cache	;\
		mkdir -p $(OUTPUT_DIR)/talos-asset-vm-boot-imgs			;\
		docker run \
		--rm -t \
		-v $${to}:/secureboot:ro \
		-v $${to}:/out \
		-v $${image_cache_loc}:/image-cache.oci:ro \
		-v /dev:/dev \
		--privileged \
		$(TALOS_IMAGER_IMG) \
		iso --image-cache /image-cache.oci --extra-kernel-arg net.ifnames=0 --extra-kernel-arg=-console --extra-kernel-arg=console=ttyS0 ;\
	}

.PHONY: create-asset-vm-nocloud-boot-iso
create-asset-vm-nocloud-boot-iso: | $(OUTPUT_DIR) pull-talos-imager-container does-image-cache-exist ## Create the nocloud iso for the asset-hoster
	@{	\
		to=$(OUTPUT_DIR)/talos-asset-vm-boot-imgs				;\
		image_cache_loc=$(OUTPUT_DIR)/eda-assets-image-cache	;\
		mkdir -p $(OUTPUT_DIR)/talos-asset-vm-boot-imgs			;\
		docker run \
		--rm -t \
		-v $${to}:/secureboot:ro \
		-v $${to}:/out \
		-v $${image_cache_loc}:/image-cache.oci:ro \
		-v /dev:/dev \
		--privileged \
		$(TALOS_IMAGER_IMG) \
		iso --image-cache /image-cache.oci --image-disk-size=2GiB --platform nocloud ;\
	}

.PHONY: create-asset-vm-vmware-boot-ova
create-asset-vm-vmware-boot-ova: | $(OUTPUT_DIR) pull-talos-imager-container does-image-cache-exist ## Create the vmware ova for the asset-hoster
	@{	\
		to=$(OUTPUT_DIR)/talos-asset-vm-boot-imgs				;\
		image_cache_loc=$(OUTPUT_DIR)/eda-assets-image-cache	;\
		mkdir -p $(OUTPUT_DIR)/talos-asset-vm-boot-imgs			;\
		docker run \
		--rm -t \
		-v $${to}:/secureboot:ro \
		-v $${to}:/out \
		-v $${image_cache_loc}:/image-cache.oci:ro \
		-v /dev:/dev \
		--privileged \
		$(TALOS_IMAGER_IMG) \
		vmware --image-cache /image-cache.oci --image-disk-size=2GiB ;\
	}

.PHONY: download-talos-stock-boot-media
download-talos-stock-boot-media: | $(EDAADM)
	@{	\
		output_to=$(OUTPUT_DIR)/talos-stock-boot-media										;\
		mkdir -p $${output_to}																;\
		mach_types=("vmware" "nocloud" "metal")												;\
		for machine in $${mach_types[@]}; do												 \
			echo "--> Downloading boot media for $${machine}"								;\
			urls=$$($(EDAADM) images -t "$${machine}" | grep "^https://")					;\
			for url in $$urls ; do															 \
				echo "    From: $$url"														;\
				filename=$$(echo "$$url" | rev | cut -f1 -d '/' | rev)						;\
				output=$${output_to}/$${filename}											;\
				echo "    To: $${output}"													;\
				$(CURL_PM) $${url} --output $${output}										;\
			done																			;\
		done																				;\
	}

## Bulk targets
## ----------------------------------------------------------------------------| 
.PHONY: save-all-bundles
save-all-bundles: | $(LIST_OF_SAVE_TARGETS)

.PHONY: load-all-bundles
load-all-bundles: | $(LIST_OF_LOAD_TARGETS)

##@ Helper targets

.PHONY: check-load-params
check-load-params:
	@$(call check-if-var-is-defined,ASSET_HOST)
	@$(call check-if-var-is-defined,ASSET_HOST_GIT_USERNAME)
	@$(call check-if-var-is-defined,ASSET_HOST_GIT_PASSWORD)
	@$(call check-if-var-is-defined,ASSET_HOST_ARTIFACTS_USERNAME)
	@$(call check-if-var-is-defined,ASSET_HOST_ARTIFACTS_PASSWORD)

.PHONY: ls-bundles
ls-bundles: | $(BASE) ; @ ## Show which bundle files are found
	@echo "--> Default eda core version to load/save: EDA_CORE_REL=$(EDA_CORE_REL)"
	@echo "--> List of bundles:"
	@echo "$(BUNDLE_LIST)" | tr ' ' '\n' | sort | sed 's/^/    |-> /'
	@echo "--> Default list:"
	@echo "$(LIST_OF_SAVE_TARGETS)" | tr ' ' '\n' | sort | cut -f2- -d '-' | sed 's/^/    |-> /'

.PHONY: help
help:  ## Show the help menu
	@sed -ne 's/^\([^[:space:]]*\):.*##/\1\t|\t/p' $(MAKEFILE_LIST) | sort | column -t -s $$'\t' | pager
