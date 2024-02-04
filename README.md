# MapApp main repository

## Description
Basic map web application for displaying a map, performing a search and calculating a route from an origin point to a search result, built on top of Mapbox's services, deployable on AWS or Azure.

Components:
- frontend:
    - [local README](map_app_frontend/README.md)
    - [Github README](https://github.com/danimihalca/map_app_frontend/blob/develop/README.md)
- services:
    - search service:
        - [local README](services/map_app_search_service/README.md)
        - [Github README](https://github.com/danimihalca/map_app_search_service/blob/main/README.md)
    - navigation service:
        - [local README](services/map_app_navigation_service/README.md)
        - [Github README](https://github.com/danimihalca/map_app_navigation_service/blob/main/README.md)

## Development project board
 [Board](https://github.com/users/danimihalca/projects/1)

## Demo on AWS
![](demo.gif)

> [!NOTE]
> The application isn't always deployed, nor will it have the URL shown in the demo.


## Build
### Prerequisites

* Initialize all submodules:

```
git submodule update --init --recursive
```

* Set Mapbox API token in `mapbox_token.txt`


### Build and run using Docker Compose


* Run Docker Compose:

```
docker compose up
```

### Build & run as a local k8s cluster via minikube

* Start minikube:

```
minikube start --vm-driver=kvm2
```
* Point Docker engine towards minikube

```
eval $(minikube -p minikube docker-env)
```

* Build docker images

```
# Base enironment for search service
cd $ROOT/services/map_app_search_service/map_app_search_service_base
docker build --target=map-app-search-service-dep-build \
    `./docker_build/source-build-args.sh docker_build/args/release.args` \
    -t <SEARCH_SERVICE_BASE_ENV_IMG> .

# Search service
cd $ROOT/services/map_app_search_service
docker build --build-arg="BASE_ENV=<SEARCH_SERVICE_BASE_ENV_IMG>" \
    --target runner -t <SEARCH_SERVICE_IMG> .

# Navigation service
cd $ROOT/services/map_app_navigation_service
docker build --target runner -t <NAVIGATION_SERVICE_IMG> .

# Frontend
cd $ROOT/map_app_frontend
docker build --target runner -t <FRONTEND_IMG> .
```

* Set Mapbox API token as secret from file

```
kubectl delete secret mapbox-access --ignore-not-found
kubectl create secret generic mapbox-access --from-file=token=mapbox_token.txt
```

* Generate search service manifest with image name from template (or manual edit) and apply

```
cat services/map_app_search_service/k8s_service.tmp.yml | IMAGE_NAME=<SEARCH_SERVICE_IMG> envsubst > k8s_service_search.yml
kubectl apply -f k8s_service_search.yml
```

* Optional for images built locally: patch manifest to disable pull

```
kubectl patch deployment search-deploy -p '{"spec": {"template": {"spec":{"containers":[{"name": "search-service-cont", "imagePullPolicy":"Never"}]}}}}'
```

* Generate navigation service manifest with image name from template (or manual edit) and apply

```
cat services/map_app_navigation_service/k8s_service.tmp.yml | IMAGE_NAME=<NAVIGATION_SERVICE_IMG> envsubst > k8s_service_navigation.yml
kubectl apply -f k8s_service_navigation.yml
```

* Optional for images built locally: patch manifest to disable pull

```
kubectl patch deployment navigation-deploy -p '{"spec": {"template": {"spec":{"containers":[{"name": "navigation-service-cont", "imagePullPolicy":"Never"}]}}}}'
```

* Generate frontend manifest with image name from template (or manual edit) and apply

```
cat map_app_frontend/k8s_service.tmp.yml | IMAGE_NAME=<FRONTEND_IMG> envsubst > k8s_service_frontend.yml
kubectl apply -f k8s_service_frontend.yml
```

* Optional for images built locally: patch manifest to disable pull

```
kubectl patch deployment frontend-deploy -p '{"spec": {"template": {"spec":{"containers":[{"name": "frontend-cont", "imagePullPolicy":"Never"}]}}}}'
```

* Expose frontend service from minikube and access the frontend via the provided address:

```
minikube service frontend-service
```

* For debugging inside pods:

```
kubectl exec --stdin --tty <POD_ID> -- /bin/bash
```
