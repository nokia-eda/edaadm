ENTROPY=$(shell date +"%F-%H-%M-%S-%N")

$(info --> INFO: List of goals: $(MAKECMDGOALS))

SHELL = /usr/bin/env bash -o pipefail
.SHELLFLAGS = -ec
# Set the default target
.DEFAULT_GOAL := help

TOP_DIR := $(abspath $(dir $(abspath $(lastword $(MAKEFILE_LIST)))))
BASE=$(CURDIR)

TOOLS ?= $(CURDIR)/tools
USER_BUNDLES_DIR ?= $(TOP_DIR)/user-bundles


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

EDAADM_VERSION ?= v3.3.0
EDAADM_SRC ?= https://github.com/nokia-eda/edaadm/releases/download/$(EDAADM_VERSION)/edaadm-$(OS)-$(ARCH)

# Number of workers to use in bundle save operations (half of online CPUs, nproc/2)
WORKERS ?= $(shell echo $$(($$(nproc)/2)))

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

TALOS_IMAGER_IMG ?= ghcr.io/siderolabs/imager:v1.11.5

# This is the parent directory where we store/load everything for this Makefile
OUTPUT_DIR ?= $(TOP_DIR)/eda-cargo

ifdef EDA_CORE_REL
$(error "[ERROR] Please use EDA_CORE_VERSION instead")
endif

EDA_CORE_VERSION ?= 26.4.3
EDA_APPS_VERSION ?= 26.4.3

EDA_ASSETS_IMAGE_CACHE ?= $(TOP_DIR)/eda-assets-image-cache-3-0-0.yaml

B64_ASSET_HOST_REGISTRY_USERNAME ?= ""
B64_ASSET_HOST_REGISTRY_PASSWORD ?= ""
ASSET_HOST_GIT_SCHEME ?= https
ASSET_HOST_ARITFACTS_SCHEME ?= https

# The _BUNDLES variables are lists of the names of the bundle files without the yaml extension
ALL_BUNDLES := $(basename $(notdir $(wildcard $(TOP_DIR)/*.yaml)))

CONNECT_BUNDLES := $(basename $(notdir $(wildcard $(TOP_DIR)/eda-bundle-connect-*.yaml)))
EXPORTER_BUNDLES := $(basename $(notdir $(wildcard $(TOP_DIR)/eda-bundle*-exporter-*.yaml $(TOP_DIR)/eda-bundle*-write-*.yaml $(TOP_DIR)/eda-bundle-snmp-traps-*.yaml $(TOP_DIR)/eda-bundle-netbox-*.yaml)))
IMPORTER_BUNDLES := $(basename $(notdir $(wildcard $(TOP_DIR)/eda-bundle*-importer-*.yaml)))
NOTIFICATION_BUNDLES := $(basename $(notdir $(wildcard $(TOP_DIR)/eda-bundle*-notifier-*.yaml $(TOP_DIR)/eda-bundle-pager-duty-*.yaml $(TOP_DIR)/eda-bundle-service-now-*.yaml)))
SIM_BUNDLES := $(basename $(notdir $(wildcard $(TOP_DIR)/eda-bundle-sim-*.yaml)))
USER_BUNDLES := $(basename $(notdir $(wildcard $(USER_BUNDLES_DIR)/*.yaml)))

# Build the save/load targets names that we want to be as default
DEFAULT_BUNDLES=

DEFAULT_BUNDLES += eda-bundle-core-$(subst .,-,$(EDA_CORE_VERSION))
DEFAULT_BUNDLES += eda-bundle-apps-$(subst .,-,$(EDA_APPS_VERSION))

ifndef SKIP_DEFAULT_BUNDLES
DEFAULT_BUNDLES += eda-bundle-talos-1-11-5
DEFAULT_BUNDLES += eda-bundle-tools-2-0-0
DEFAULT_BUNDLES += eda-bundle-storage-2-0-0
DEFAULT_BUNDLES += eda-bundle-metallb-0-13-7
endif

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


# $1 is the name of the target for which to create a save-$1 and load-$1 target
# $2 is the path to the bundle.yaml file for the operation
define gen-targets
.PHONY: save-$1
save-$1: | $(OUTPUT_DIR) $(EDAADM)
	@if [[ ! -f "$(2)" ]]; then (echo "--> [ERROR] The bundle file: $2 does not exist!" & exit 1); fi
	@{	\
		START=$$(date +%s)					;\
		echo "--> [SAVE] $1"				;\
		to=$(OUTPUT_DIR)/$(1)				;\
		mkdir -p  $$$${to}					;\
		$(EDAADM) bundle save				 \
		-w $(WORKERS)						 \
		-f $(2)								 \
		-t $$$${to}							;\
		echo "--> [SAVE] Saved to $$$${to}"	;\
	}

.PHONY: load-$1
load-$1: | $(OUTPUT_DIR) $(EDAADM) check-load-params
	@if [[ ! -f "$(2)" ]]; then (echo "--> [ERROR] The bundle file: $2 does not exist!" & exit 1); fi
	@{	\
		if [[ -z "$(ASSET_HOST)" ]]; then																								 \
			export _ASSET_HOST_ARTIFACTS=$(ASSET_HOST_ARTIFACTS)																		;\
			export _ASSET_HOST_GIT=$(ASSET_HOST_GIT)																					;\
			export _ASSET_HOST_REGISTRY=$(ASSET_HOST_REGISTRY)																			;\
		else																															 \
			export _ASSET_HOST_ARTIFACTS="$(ASSET_HOST_ARITFACTS_SCHEME)://$(ASSET_HOST)/artifacts-upload"								;\
			export _ASSET_HOST_GIT=$(ASSET_HOST_GIT_SCHEME)://$(ASSET_HOST)/git/$$$$(echo -n "$(B64_ASSET_HOST_GIT_USERNAME)" | base64 -d)	;\
			export _ASSET_HOST_REGISTRY=$(ASSET_HOST)																					;\
		fi																																;\
		echo "--> [LOAD] $1"																											;\
		from=$(OUTPUT_DIR)/$(1)																											;\
		if [[ ! -d $$$${from}  ]]; then 																								 \
			echo "[ERROR] $$$${from} does not exist!"																					;\
			echo "        did you run save-$(1) ?"																						;\
			exit 1																														;\
		fi 																																;\
		$(EDAADM) bundle load \
		-f $(2) \
		-l $$$${from} \
		--dest-registry $$$${_ASSET_HOST_REGISTRY} \
		--dest-registry-user $(B64_ASSET_HOST_REGISTRY_USERNAME) \
		--dest-registry-pass $(B64_ASSET_HOST_REGISTRY_PASSWORD) \
		$(ARGS_INSECURE) \
		--dest-artifactserver "$$$${_ASSET_HOST_ARTIFACTS}" \
		--dest-artifactserver-user $(B64_ASSET_HOST_ARTIFACTS_USERNAME) \
		--dest-artifactserver-pass $(B64_ASSET_HOST_ARTIFACTS_PASSWORD) \
		--dest-reposerver "$$$${_ASSET_HOST_GIT}" \
		--dest-reposerver-user $(B64_ASSET_HOST_GIT_USERNAME)	 \
		--dest-reposerver-pass $(B64_ASSET_HOST_GIT_PASSWORD)	;\
		echo "--> [LOAD] Loaded from $1"					;\
	}
endef

# Generate all of the load/save targets for the bundle files in TOP_DIR/
$(foreach bundle,$(ALL_BUNDLES),$(eval $(call gen-targets,$(bundle),$(TOP_DIR)/$(bundle).yaml)))
# Generate all of the load/save targets for the found user bundles in the user-bundles/ directory
$(foreach userbundle,$(USER_BUNDLES),$(eval $(call gen-targets,user-$(userbundle),$(USER_BUNDLES_DIR)/$(userbundle).yaml)))

## Bulk targets
## HELP_ENTRIES tuples: target|description, separated by ::
## $1 is the name/prefix to use in the target - it must be a single word in lowercase
##    the following targets will be generated:
##    save-<name>-bundles
##    load-<name>-bundles
##    ls-<name>-bundles
## $2 is the list of bundles to generate the targets for
## $3 is the description of the bundles
## $4 is an optional prefix to the name of the bundle file target name, it is the same prefix used in the target name generated by gen-targets - it must be a single word in lowercase
## ----------------------------------------------------------------------------|
define generate-bulk-targets-from-list
LIST_SAVE_$(1)_BUNDLES := $(addprefix save-$(4),$(2))
LIST_LOAD_$(1)_BUNDLES := $(addprefix load-$(4),$(2))

HELP_ENTRIES+=save-$(1)-bundles|Save $(3)::
.PHONY: save-$(1)-bundles
save-$(1)-bundles: | $$(LIST_SAVE_$(1)_BUNDLES)

HELP_ENTRIES+=load-$(1)-bundles|Load $(3)::
.PHONY: load-$(1)-bundles
load-$(1)-bundles: | $$(LIST_LOAD_$(1)_BUNDLES)

HELP_ENTRIES+=ls-$(1)-bundles|List $(3)::
.PHONY: ls-$(1)-bundles
ls-$(1)-bundles: | $(BASE)
ifeq ($(1),default)
	@echo "--> Default eda core version to load/save: EDA_CORE_VERSION=$(EDA_CORE_VERSION), EDA_APPS_VERSION=$(EDA_APPS_VERSION)"
endif
	@echo "--> List of $(1) bundles:"
	@if [[ ! -z "$(2)" ]]; then echo "$(2)" | tr ' ' '\n' | sort | sed 's/^/$(4)/' | sed 's/^/    /'; fi
endef

$(eval $(call generate-bulk-targets-from-list,all,$(ALL_BUNDLES),'all of the bundles found in  $(TOP_DIR)'))
$(eval $(call generate-bulk-targets-from-list,default,$(DEFAULT_BUNDLES),'the selected set of default bundles'))
$(eval $(call generate-bulk-targets-from-list,connect,$(CONNECT_BUNDLES),'the connect app bundles'))
$(eval $(call generate-bulk-targets-from-list,exporter-app,$(EXPORTER_BUNDLES),'bundles that export data from the system'))
$(eval $(call generate-bulk-targets-from-list,importer-app,$(IMPORTER_BUNDLES),'bundles that import data into the system'))
$(eval $(call generate-bulk-targets-from-list,notification-app,$(NOTIFICATION_BUNDLES),'bundles that send notifications about the system'))
$(eval $(call generate-bulk-targets-from-list,sim,$(SIM_BUNDLES),'bundles for use in digital twins'))
$(eval $(call generate-bulk-targets-from-list,user,$(USER_BUNDLES),'bundles provided by the user in $(USER_BUNDLES_DIR)',user-))

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
create-assets-host-bootstrap-image-cache: | $(OUTPUT_DIR) $(EDAADM) ## Create the image cache used to self bootstrap the asset cluster
	@{	\
		$(EDAADM) bundle cache-create								 \
		--output-cache-path $(OUTPUT_DIR)/eda-assets-image-cache	 \
		--assets $(EDA_ASSETS_IMAGE_CACHE)							;\
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
		to=$(OUTPUT_DIR)/talos-asset-vm-boot-imgs								;\
		output_file_name=metal-amd64.iso										;\
		output_file=$${to}/$${output_file_name}									;\
		rename_file=$${to}/asset-vm-$${output_file_name}						;\
		image_cache_loc=$(OUTPUT_DIR)/eda-assets-image-cache					;\
		mkdir -p $(OUTPUT_DIR)/talos-asset-vm-boot-imgs							;\
		docker run \
		--rm -t \
		-v $${to}:/secureboot:ro \
		-v $${to}:/out \
		-v $${image_cache_loc}:/image-cache.oci:ro \
		-v /dev:/dev \
		--privileged \
		$(TALOS_IMAGER_IMG) \
		iso --image-cache /image-cache.oci --extra-kernel-arg net.ifnames=0 --extra-kernel-arg=-console --extra-kernel-arg=console=ttyS0 ;\
		\mv -fv $${output_file} $${rename_file}									;\
		echo "--> INFO: Created $$(readlink -f $${rename_file})"				;\
	}

.PHONY: create-asset-vm-nocloud-boot-iso
create-asset-vm-nocloud-boot-iso: | $(OUTPUT_DIR) pull-talos-imager-container does-image-cache-exist ## Create the nocloud iso for the asset-hoster
	@{	\
		to=$(OUTPUT_DIR)/talos-asset-vm-boot-imgs								;\
		output_file_name=nocloud-amd64.iso										;\
		output_file=$${to}/$${output_file_name}									;\
		rename_file=$${to}/asset-vm-$${output_file_name}						;\
		image_cache_loc=$(OUTPUT_DIR)/eda-assets-image-cache					;\
		mkdir -p $(OUTPUT_DIR)/talos-asset-vm-boot-imgs							;\
		docker run \
		--rm -t \
		-v $${to}:/secureboot:ro \
		-v $${to}:/out \
		-v $${image_cache_loc}:/image-cache.oci:ro \
		-v /dev:/dev \
		--privileged \
		$(TALOS_IMAGER_IMG) \
		iso --image-cache /image-cache.oci --image-disk-size=2GiB --platform nocloud ;\
		\mv -fv $${output_file} $${rename_file}									;\
		echo "--> INFO: Created $$(readlink -f $${rename_file})"				;\
	}

.PHONY: create-asset-vm-vmware-boot-ova
create-asset-vm-vmware-boot-ova: | $(OUTPUT_DIR) pull-talos-imager-container does-image-cache-exist ## Create the vmware ova for the asset-hoster
	@{	\
		to=$(OUTPUT_DIR)/talos-asset-vm-boot-imgs								;\
		output_file_name=vmware-amd64.ova										;\
		output_file=$${to}/$${output_file_name}									;\
		rename_file=$${to}/asset-vm-$${output_file_name}						;\
		image_cache_loc=$(OUTPUT_DIR)/eda-assets-image-cache					;\
		mkdir -p $(OUTPUT_DIR)/talos-asset-vm-boot-imgs							;\
		docker run \
		--rm -t \
		-v $${to}:/secureboot:ro \
		-v $${to}:/out \
		-v $${image_cache_loc}:/image-cache.oci:ro \
		-v /dev:/dev \
		--privileged \
		$(TALOS_IMAGER_IMG) \
		vmware --image-cache /image-cache.oci --image-disk-size=2GiB ;\
		\mv -fv $${output_file} $${rename_file}									;\
		echo "--> INFO: Created $$(readlink -f $${rename_file})"				;\
	}

.PHONY: download-talos-stock-boot-media
download-talos-stock-boot-media: | $(EDAADM) ## Download the Talos boot media for all supported platforms
	$(EDAADM) images -t all -s $(OUTPUT_DIR)/talos-stock-boot-media


##@ Helper targets

.PHONY: check-load-params
check-load-params:
ifndef ASSET_HOST
	@$(call check-if-var-is-defined,ASSET_HOST_GIT)
	@$(call check-if-var-is-defined,ASSET_HOST_REGISTRY)
	@$(call check-if-var-is-defined,ASSET_HOST_ARTIFACTS)
else
	@$(call check-if-var-is-defined,ASSET_HOST)
endif
	@$(call check-if-var-is-defined,B64_ASSET_HOST_GIT_USERNAME)
	@$(call check-if-var-is-defined,B64_ASSET_HOST_GIT_PASSWORD)
	@$(call check-if-var-is-defined,B64_ASSET_HOST_ARTIFACTS_USERNAME)
	@$(call check-if-var-is-defined,B64_ASSET_HOST_ARTIFACTS_PASSWORD)

# This target first gets the help entries for targets that are defined in the Makefile and sorts them
# Then it gets the help entries for the generated bulk targets which are stored in the HELP_ENTRIES variable
# Both are fed into the same stdout stream for formatting, 'column' is removed since it may not be available 
.PHONY: help
help:  ## Show the help menu
	@{ \
		sed -ne 's/^\([^[:space:]]*\):.*## \(.*\)/\1|\2/p' $(MAKEFILE_LIST) | sort			;\
		echo '$(HELP_ENTRIES)' | sed 's/::/\n/g' | sed '/^$$/d' | sed 's/^[[:space:]]*//'	;\
		downloaded_tool_message=""															;\
		if [[ ! -f $(EDAADM) ]]; then														 \
			downloaded_tool_message="($(EDAADM) isn't downloaded - run make download-tools)";\
		else																				 \
			downloaded_tool_message="Downloaded $$($(EDAADM) version)"						;\
		fi																					;\
		echo "ADM Version|Expected: $(EDAADM_VERSION), $${downloaded_tool_message}"					;\
	} | while IFS='|' read -r target desc; do printf '%-40s | %s\n' "$$target" "$$desc"; done
