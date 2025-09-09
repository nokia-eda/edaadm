ENTROPY=$(shell date +"%F-%H-%M-%S-%N")

$(info --> INFO: List of goals: $(MAKECMDGOALS))

SHELL = /usr/bin/env bash -o pipefail
.SHELLFLAGS = -ec
# Set the default target
.DEFAULT_GOAL := help

BASE=$(CURDIR)
CFG ?= $(CURDIR)/configs
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

SED ?= sed
INDENT_OUT := $(SED) 's/^/    /'

TIMEOUT_INGRESS ?= 120s

EDA_ASSETS_NS ?= eda-assets
ASSET_HOST_HTTP_PORT ?= 80
ASSET_HOST_HTTPS_PORT ?= 443

## Tool Versions:
KPT_VERSION ?= v1.0.0-beta.44
KUBECTL_VERSION ?= v1.31.1

## Tools:
KPT ?= $(TOOLS)/kpt-$(KPT_VERSION)
KUBECTL ?= $(TOOLS)/kubectl-$(KUBECTL_VERSION)

APPLY_SETTER_IMG=ghcr.io/srl-labs/kpt-apply-setters:0.1.1
KUBECTL_SRC ?= https://dl.k8s.io/release/$(KUBECTL_VERSION)/bin/$(OS)/$(ARCH)/kubectl
KPT_SRC ?= https://github.com/GoogleContainerTools/kpt/releases/download/$(KPT_VERSION)/kpt_$(OS)_$(ARCH)

## Curl options:
CURL := curl --silent --fail --show-error

KPT_LIVE_INIT_FORCE ?= 0

define download-bin
    $(info --> INFO: Downloading $(2))
	if test ! -f $(1); then $(CURL) -Lo $(1) $(2) >/dev/null && chmod a+x $(1); fi
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


ifeq ($(KPT_INVENTORY_ADOPT),1)
KPT_LIVE_APPLY_ARGS += --inventory-policy=adopt
endif

define INSTALL_KPT_PACKAGE
	{	\
		echo -e "--> KPT: [$2] - Applying kpt package"									;\
		pushd $1 &>/dev/null || (echo "[ERROR]: Failed to switch cwd to $2" && exit 1)	;\
		if [[ ! -f resourcegroup.yaml ]] || [[ $(KPT_LIVE_INIT_FORCE) -eq 1 ]]; then	 \
			$(KPT) live init --force |& sed 's/^/    /'									;\
		else																			 \
			echo "--> KPT: [$2] resource group found, not re-init this package"			;\
		fi																				;\
		set  -x ;\
		$(KPT) live apply $(KPT_LIVE_APPLY_ARGS) |& sed 's/^/    /'						;\
		set +x ;\
		popd &>/dev/null || (echo "[ERROR]: Failed to switch back from $2" && exit 1)	;\
		echo -e "--> KPT: [$2] - Applied and reconciled package"						;\
	}
endef

define UN_INSTALL_KPT_PACKAGE
	{	\
		echo -e "--> KPT: [$2] - Uninstalling kpt package"								;\
		pushd $1 &>/dev/null || (echo "[ERROR]: Failed to switch cwd to $2" && exit 1)	;\
		set  -x ;\
		$(KPT) live destroy $(KPT_LIVE_APPLY_ARGS) |& sed 's/^/    /'					;\
		set +x ;\
		popd &>/dev/null || (echo "[ERROR]: Failed to switch back from $2" && exit 1)	;\
		echo -e "--> KPT: [$2] - Removed and reconciled package"						;\
	}
endef

##@ Download tools

.PHONY: download-tools
download-tools: | $(TOOLS) $(KPT) $(KUBECTL)

$(TOOLS): | $(BASE)
	mkdir -p $(TOOLS)

$(KUBECTL): | $(BASE) $(TOOLS) ; $(info --> TOOLS: Ensuring kubectl is present in $(KUBECTL))
	@$(call download-bin,$(KUBECTL),$(KUBECTL_SRC))

$(KPT): | $(BASE) $(TOOLS) ; $(info --> TOOLS: Ensuring kpt is present in $(KPT))
	@$(call download-bin,$(KPT),$(KPT_SRC))

##@ Configure components

.PHONY: eda-configure-shipyard
eda-configure-shipyard: | $(KPT) ; $(info --> INFO: configuring shipyard) @ ## Configure the shipyard
	@{	\
		if [[ -z "$(ASSET_HOST_GIT_USERNAME)" ]]; then echo "[ERROR]: ASSET_HOST_GIT_USERNAME must be defined" && exit 1; fi ;\
		if [[ -z "$(ASSET_HOST_GIT_PASSWORD)" ]]; then echo "[ERROR]: ASSET_HOST_GIT_PASSWORD must be defined" && exit 1; fi ;\
		if [[ -z "$(ASSET_HOST_ARTIFACTS_HTPASSWORD)" ]]; then echo "[ERROR]: ASSET_HOST_ARTIFACTS_HTPASSWORD must be defined" && exit 1; fi ;\
	}
	@{	\
		$(KPT) fn eval \
		--image $(APPLY_SETTER_IMG) \
		--truncate-output=false \
		-- \
		ASSET_HOST_GIT_USERNAME=$(ASSET_HOST_GIT_USERNAME) \
		ASSET_HOST_GIT_PASSWORD=$(ASSET_HOST_GIT_PASSWORD) \
		ASSET_HOST_ARTIFACTS_HTPASSWORD=$(ASSET_HOST_ARTIFACTS_HTPASSWORD) \
		;\
	}

##@ Install K8S components

.PHONY: ingress
ingress: | $(BASE) download-tools ; @ ## Install ingress
	@$(call INSTALL_KPT_PACKAGE,eda-kpt-asset-ingress,ingress)
	@{	\
		START=$$(date +%s.%N)																	;\
		echo "--> INGRESS: Waiting for nginx ingress to start (timeout: $(TIMEOUT_INGRESS))"	;\
		$(KUBECTL) wait --namespace ingress-nginx \
						--for=condition=ready pod \
						--selector=app.kubernetes.io/component=controller \
						--timeout=$(TIMEOUT_INGRESS) | $(INDENT_OUT)							;\
		echo "--> INGRESS: nginx started!"														;\
		echo "--> INGRESS: ingress launch took $$(echo "$$(date +%s.%N) - $$START" | bc)"		;\
	}

.PHONY: storage
storage: | $(BASE) download-tools ; @ ## Install storage - local-path-provisioner
	@$(call INSTALL_KPT_PACKAGE,eda-kpt-asset-storage,storage)
	@{	\
		START=$$(date +%s.%N)																					;\
		echo "--> STORAGE: Waiting for local-path-provisioner to start (timeout: $(TIMEOUT_INGRESS))"			;\
		$(KUBECTL) wait --namespace local-path-storage \
						--for=condition=ready pod \
						--selector=app=local-path-provisioner \
						--timeout=$(TIMEOUT_INGRESS) | $(INDENT_OUT)											;\
		echo "--> STORAGE: local-path-provisioner started!"														;\
		echo "--> STORAGE: local-path-provisioner launch took $$(echo "$$(date +%s.%N) - $$START" | bc)"		;\
	}

.PHONY: eda-shipyard
eda-shipyard: | $(BASE) download-tools ; @ ## Install the shipyard components - registry, git server, web host
	@$(call INSTALL_KPT_PACKAGE,eda-kpt-asset-host,asset-host)

.PHONY: eda-setup-shipyard
eda-setup-shipyard: | ingress storage eda-shipyard ; @ ## Install ingress, storage and eda-asset-hoster

##@ Uninstall K8S components

.PHONY: uninstall-ingress
uninstall-ingress: | $(BASE) download-tools ; @ ## Uninstall ingress
	@$(call UN_INSTALL_KPT_PACKAGE,eda-kpt-asset-ingress,ingress)

.PHONY: uninstall-storage
uninstall-storage: | $(BASE) download-tools ; @ ## Uninstall storage - remove local-path-provisioner
	@$(call UN_INSTALL_KPT_PACKAGE,eda-kpt-asset-storage,storage)

.PHONY: uninstall-eda-asset-host
uninstall-eda-asset-host: | $(BASE) download-tools ; @ ## Uninstall asset-hoster components
	@$(call UN_INSTALL_KPT_PACKAGE,eda-kpt-asset-host,asset-host)

.PHONY: uninstall-eda-shipyard
uninstall-eda-shipyard: | uninstall-ingress uninstall-storage uninstall-eda-asset-host ; @ ## Install ingress, storage and eda-asset-hoster

##@ Help

.PHONY: help
help:  ## Show the help menu
	@sed -ne 's/^\([^[:space:]]*\):.*##/\1\t|\t/p' $(MAKEFILE_LIST) | sort | column -t -s $$'\t' | pager


.PHONY: ls-setters
ls-setters: | $(BASE) $(KPT) ## View available kpt setters
	@$(KPT) fn eval -i list-setters:v0.1.0 --truncate-output=false