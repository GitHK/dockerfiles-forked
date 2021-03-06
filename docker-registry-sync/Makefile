.DEFAULT_GOAL := help

BRIDGE_NETWORK_NAME := dev_sync_net

export IMAGE_NAME := docker-registry-sync
export DOCKER_REG_SYNC_VERSION := 0.1.0
# if this image is built by this script it will be tagged as stable
export DOCKER_REG_SYNC_TAS_AS_STABLEVERSION := 0.1.0
export DOCKER_REGISTRY ?=itisfoundation

export VCS_URL         := $(shell git config --get remote.origin.url)
export VCS_REF         := $(shell git rev-parse --short HEAD)
export BUILD_DATE      := $(shell date -u +"%Y-%m-%dT%H:%M:%SZ")

current_dir = $(shell pwd)


.PHONY: help
help: ## help on rule's targets
	@awk 'BEGIN {FS = ":.*?## "} /^[[:alpha:][:space:]_-]+:.*?## / {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}' $(MAKEFILE_LIST)

##### PYTHON PACKAGE -START

.PHONY: _ensure-in-venv
_ensure-in-venv:
	@python3 -c "import os; os.environ['VIRTUAL_ENV']" || (echo "\n>>>> You are not in a virtualenv. Activate one <<<<\n"; exit 1)


.PHONY: dev-dependencies
dev-dependencies: _ensure-in-venv ## install development depenencies in your venv
	@pip install pip-tools black rope


.PHONY: compile-dependencies
compile-dependencies: _ensure-in-venv ## snapshot dependencies
	@pip-compile --build-isolation $(current_dir)/reposync/requirements/base.in --output-file $(current_dir)/reposync/requirements/base.txt 
	@pip-compile --build-isolation $(current_dir)/reposync/requirements/test.in --output-file $(current_dir)/reposync/requirements/test.txt 


.PHONY: install-dependencies
install-dependencies: _ensure-in-venv ## install package dependencies
	@pip install -r $(current_dir)/reposync/requirements/base.txt 


.PHONY: install-py-package
install-py-package: _ensure-in-venv ## install package dependencies
	@pip install -e reposync

##### PYTHON PACKAGE - END

.PHONY: start-dev-registry
start-dev-registry: ## starts development registries used by development-run
	@docker network create ${BRIDGE_NETWORK_NAME}
	@docker-compose -f dev/docker-compose.dev.yaml up -d

.PHONY: stop-dev-registry
stop-dev-registry: ## stops development registries used by development-run
	@docker-compose -f dev/docker-compose.dev.yaml down
	@docker network rm ${BRIDGE_NETWORK_NAME}
	@sudo rm -rf dev/data_*

.PHONY: development-run
development-run: _ensure-in-venv build ## used to test out changes during developmen
	@docker run --rm -it \
		-e ENV_VAR_USERNAME_DOCKERHUB=dockerhub_user \
		-e ENV_PASSWORD_DOCKERHUB=dockerhub_password \
		-e ENV_VAR_USERNAME_MASTER=testuser \
		-e ENV_PASSWORD_MASTER=testpassword \
		-e ENV_VAR_USERNAME_STAGING=testuser \
		-e ENV_PASSWORD_STAGING=testpassword \
		-e ENV_VAR_USERNAME_AWS=testuser \
		-e ENV_PASSWORD_AWS=testpassword \
		-e SYNC_EXIT_ON_FIRST_ERROR=true \
		-v $(current_dir)/dev/dev-sync-cfg.yml:/etc/cfg.yaml \
		--network=${BRIDGE_NETWORK_NAME} \
		${DOCKER_REGISTRY}/${IMAGE_NAME}:${DOCKER_REG_SYNC_VERSION}

# Used by deployment stages

.PHONY: build
build: ## build and tag the iamge
	@docker-compose build

.PHONY: push
push: ## pushes image to Dockerhub
	@docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${DOCKER_REG_SYNC_VERSION}

.PHONY: pull
pull: ## pulls image from Dockerhub
	@docker pull ${DOCKER_REGISTRY}/${IMAGE_NAME}:${DOCKER_REG_SYNC_VERSION}

define show-meta
	$(foreach iid,$(shell docker images */$(1):* -q | sort | uniq),\
		docker image inspect $(iid) | jq '.[0] | .RepoTags, .ContainerConfig.Labels';)
endef

.PHONY: info
info: ## gives some information about the docker image
	@docker images */${IMAGE_NAME}:*;\
	$(call show-meta,${IMAGE_NAME})

# Used by GitHub Actions, these should be defined by each project

.PHONY: github-ci-pull
github-ci-pull: ## reserved for Git Hub CI to pull image
	@make pull

.PHONY: github-ci-build
github-ci-build: ## reserved for Git Hub CI to build image
	@make build
	# checkk the version for stable
ifeq (${DOCKER_REG_SYNC_VERSION}, ${DOCKER_REG_SYNC_TAS_AS_STABLEVERSION})
	docker tag ${DOCKER_REGISTRY}/${IMAGE_NAME}:${DOCKER_REG_SYNC_VERSION} ${DOCKER_REGISTRY}/${IMAGE_NAME}:stable
endif
	@make info

.PHONY: github-ci-tests
github-ci-tests:	## reserved for Git Hub CI to start testsuite
	@echo "No tests at this time..."

.PHONY: github-ci-push
github-ci-push:	## reserved for Git Hub CI to push image
	@make push
ifeq (${DOCKER_REG_SYNC_VERSION}, ${DOCKER_REG_SYNC_TAS_AS_STABLEVERSION})
	@docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:stable
endif
	
