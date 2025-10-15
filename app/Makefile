image ?= "abuelnaga0/vprofile-app-backend"
version ?= $(shell mvn -q -DforceStdout help:evaluate -Dexpression=project.version)

.PHONY: docker-push
docker-push: docker-build
	docker push ${image}:${version}

.PHONY: docker-build
docker-build:
	docker build --network host --build-arg version=${version} --tag ${image}:${version} .

.PHONY: local-build
local-build:
	mvn clean install

.PHONY: get-version
get-version:
	@echo ${version}