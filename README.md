# MapApp main repository

## Build and run using Docker Compose

* Initialize all submodules:

```
git submodule update --init --recursive
```

* Set Mapbox API token in `mapbox_token.txt`
* Run Docker Compose:

```
docker compose up
```

## Run k8s cluster locally using minikube

* Start minikube

```
minikube start --vm-driver=kvm2
```
* Point Docker engine towards minikube

```
eval $(minikube -p minikube docker-env)
```

* Build docker images

```
cd $ROOT/services/map_app_search_service/map_app_search_service_base
docker build --target=map-app-search-service-dep-build \
    `./docker_build/source-build-args.sh docker_build/args/release.args` \
    -t <SEARCH_SERVICE_BASE_ENV_IMG> .

cd $ROOT/services/map_app_search_service
docker build --build-arg="BASE_ENV=<SEARCH_SERVICE_BASE_ENV_IMG>" \
    --target runner -t <SEARCH_SERVICE_IMG> .

cd $ROOT/services/map_app_navigation_service
docker build --target runner -t <NAVIGATION_SERVICE_IMG> .

cd $ROOT/map_app_frontend
docker build --target runner -t <FRONTEND_IMG> .
```

* Set Mapbox API token as secret

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

* Expose frontend service from minikube

```
minikube service frontend-service
```

* For debugging inside pods:

```
kubectl exec --stdin --tty <POD_ID> -- /bin/bash
```