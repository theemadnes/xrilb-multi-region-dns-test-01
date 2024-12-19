# xrilb-multi-region-dns-test-01

using cross-region NLB in front of multiple GKE clusters with CSM enabled w/ ingress gateways. this will just use non-TLS traffic for demo purposes.

### setup

assumes two existing GKE (in this case, autopilot) clusters in `REGION_1` and `REGION_2`, with CSM enabled on both clusters.

```
export PROJECT=xrilb-multi-region-dns-test-01
export VPC=default
export REGION_1=us-central1
export REGION_2=us-east4
export CLUSTER_1=gke-1
export CLUSTER_2=gke-2
export KUBECTX_1=gke_${PROJECT}_${REGION_1}_${CLUSTER_1}
export KUBECTX_2=gke_${PROJECT}_${REGION_2}_${CLUSTER_2}
gcloud config set project $PROJECT
```

### create ingress gateways and sample workload



```
kubectl --context=${KUBECTX_1} create namespace csm-ingress
kubectl --context=${KUBECTX_2} create namespace csm-ingress

kubectl --context=${KUBECTX_1} label namespace csm-ingress istio-injection=enabled
kubectl --context=${KUBECTX_2} label namespace csm-ingress istio-injection=enabled

kubectl --context ${KUBECTX_1} apply -k ingress-gateway/variant
kubectl --context ${KUBECTX_2} apply -k ingress-gateway/variant

kubectl --context=${KUBECTX_1} create namespace whereami
kubectl --context=${KUBECTX_2} create namespace whereami

kubectl --context=${KUBECTX_1} label namespace whereami istio-injection=enabled
kubectl --context=${KUBECTX_2} label namespace whereami istio-injection=enabled

kubectl --context ${KUBECTX_1} apply -k whereami/variant
kubectl --context ${KUBECTX_2} apply -k whereami/variant

```