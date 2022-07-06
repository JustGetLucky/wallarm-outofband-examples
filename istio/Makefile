include env.ini

all: create_cluster deploy_istio deploy_demo

create_cluster:
	kind create cluster --config kind.yaml -v 1

deploy_istio:
	helm repo add istio https://istio-release.storage.googleapis.com/charts
	helm repo update
	helm install istio-base istio/base -n istio-system --wait --create-namespace
	helm install istiod istio/istiod -n istio-system --wait
	helm install istio-ingress istio/gateway -f istio.yaml -n istio-system --wait
	kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.14/samples/addons/prometheus.yaml
	kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.14/samples/addons/kiali.yaml
	kubectl wait --for=condition=Ready pods --all --namespace istio-system --timeout 5m0s

deploy_demo:
	yq -e -i '.spec.hosts[0] = "$(WALLARM_ALB_HOST)"' resources/istio-service-entry.yaml
	yq -e -i '.spec.ports[0].number = $(WALLARM_ALB_PORT)' resources/istio-service-entry.yaml
	yq -e -i '.spec.http[0].mirror.host = "$(WALLARM_ALB_HOST)"' resources/istio-virtual-service.yaml
	yq -e -i '.spec.http[0].mirror.port.number = $(WALLARM_ALB_PORT)' resources/istio-virtual-service.yaml
	kubectl apply -k resources
	kubectl wait --for=condition=Ready pods --all --namespace wallarm-demo --timeout 5m0s

delete_demo:
	kubectl delete -k resources

clean:
	kind delete cluster