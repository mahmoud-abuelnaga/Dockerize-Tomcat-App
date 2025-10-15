version ?= "v1.0.0"
image ?= "abuelnaga0/vprofile-app-nginx"

.PHONY: docker-push
docker-push:
	docker push ${image}:${version}

.PHONY: docker-build
docker-build:
	docker build --tag ${image}:${version} .