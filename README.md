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
export KUBECTX_1=gke_e2m-private-test-01_us-central1_edge-to-mesh-01
export KUBECTX_2=gke_e2m-private-test-01_us-east4_edge-to-mesh-02
gcloud config set project $PROJECT
```

### create ingress gateways and sample workload



```

```