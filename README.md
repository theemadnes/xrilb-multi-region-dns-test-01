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

kubectl --context ${KUBECTX_1} apply -f whereami-csm
kubectl --context ${KUBECTX_2} apply -f whereami-csm


```

### create x-region internal NLBs

```
gcloud compute networks subnets create $REGION_1-x-pos \
    --purpose=GLOBAL_MANAGED_PROXY \
    --role=ACTIVE \
    --region=$REGION_1 \
    --network=$VPC \
    --range=172.16.30.0/24 \
    --project ${PROJECT}

gcloud compute networks subnets create $REGION_2-x-pos \
    --purpose=GLOBAL_MANAGED_PROXY \
    --role=ACTIVE \
    --region=$REGION_2 \
    --network=$VPC \
    --range=172.16.40.0/24 \
    --project ${PROJECT}

gcloud compute firewall-rules create fw-allow-health-check \
    --network=$VPC \
    --action=allow \
    --direction=ingress \
    --target-tags=allow-health-check \
    --source-ranges=130.211.0.0/22,35.191.0.0/16 \
    --rules=tcp:15021,tcp:8080,tcp:80 \
    --project ${PROJECT}

gcloud compute firewall-rules create fw-allow-proxy-only-subnet \
    --network=$VPC \
    --action=allow \
    --direction=ingress \
    --target-tags=allow-proxy-only-subnet \
    --source-ranges=172.16.30.0/24,172.16.40.0/24 \
    --rules=tcp:8080,tcp:15021,tcp:80 \
    --project ${PROJECT}

# create health check 
gcloud compute health-checks create tcp ig-basic-check \
   --port 15021 \
   --global \
   --project ${PROJECT}

# update clusters to allow for health checks 
gcloud container clusters update $CLUSTER_1 \
    --autoprovisioning-network-tags="allow-proxy-only-subnet","allow-health-check" \
    --region $REGION_1 \
    --project ${PROJECT}

gcloud container clusters update $CLUSTER_2 \
    --autoprovisioning-network-tags="allow-proxy-only-subnet","allow-health-check" \
    --region $REGION_2 \
    --project ${PROJECT}

# create backend service
gcloud compute backend-services create ingress-gateway \
  --load-balancing-scheme=INTERNAL_MANAGED \
  --protocol=TCP \
  --enable-logging \
  --logging-sample-rate=1.0 \
  --health-checks=ig-basic-check \
  --global-health-checks \
  --global \
  --project ${PROJECT}

### hack-y way to get the negs added to the backend services
for ZONE in $REGION_1-a $REGION_1-b $REGION_1-c $REGION_1-d $REGION_1-e $REGION_1-f $REGION_2-a $REGION_2-b $REGION_2-c $REGION_2-d $REGION_2-e $REGION_2-f
do
	gcloud compute backend-services add-backend ingress-gateway \
    --global \
    --balancing-mode=CONNECTION \
    --max-connections-per-endpoint=100 \
    --network-endpoint-group=ingress-gateway \
    --network-endpoint-group-zone=$ZONE \
    --project ${PROJECT}
done

# create target proxy
gcloud compute target-tcp-proxies create ingress-gateway-tcp-proxy \
  --backend-service=ingress-gateway \
  --global \
  --project ${PROJECT}

# create forwarding rules
gcloud compute forwarding-rules create ingress-gateway-forwarding-rule-us-central1 \
  --load-balancing-scheme=INTERNAL_MANAGED \
  --network=$VPC \
  --subnet=default \
  --subnet-region=$REGION_1 \
  --ports=80 \
  --target-tcp-proxy=ingress-gateway-tcp-proxy \
  --global \
  --project ${PROJECT}

gcloud compute forwarding-rules create ingress-gateway-forwarding-rule-us-east4 \
  --load-balancing-scheme=INTERNAL_MANAGED \
  --network=$VPC \
  --subnet=default \
  --subnet-region=$REGION_2 \
  --ports=80 \
  --target-tcp-proxy=ingress-gateway-tcp-proxy \
  --global \
  --project ${PROJECT}

```

Things should be working now, at least for calling via IP address of the forwarding rule. Create a VM (with SSH permitted via FW rule, of course), in the consle find the VIPs of the forwarding rules, and `curl`:

```
admin@instance-20241220-053422:~$ curl -H "Host: whereami.mesh.example.com" http://10.128.0.8
{"cluster_name":"gke-1","gce_instance_id":"2800049394249789930","gce_service_account":"xrilb-multi-region-dns-test-01.svc.id.goog","host_header":"whereami.mesh.example.com","metadata":"whereami","node_name":"gk3-gke-1-pool-2-d0624dc5-jjgp","pod_ip":"10.88.0.135","pod_name":"whereami-988cc4ddf-dxnnl","pod_name_emoji":"\ud83c\udfc3\ud83c\udffd\u200d\u2642\u200d\u27a1","pod_namespace":"whereami","pod_service_account":"whereami","project_id":"xrilb-multi-region-dns-test-01","timestamp":"2024-12-20T05:38:02","zone":"us-central1-a"}
admin@instance-20241220-053422:~$ curl -H "Host: whereami.mesh.example.com" http://10.150.0.8
{"cluster_name":"gke-2","gce_instance_id":"957125274528280900","gce_service_account":"xrilb-multi-region-dns-test-01.svc.id.goog","host_header":"whereami.mesh.example.com","metadata":"whereami","node_name":"gk3-gke-2-pool-2-34782f5d-77m5","pod_ip":"10.24.0.136","pod_name":"whereami-988cc4ddf-f9jkh","pod_name_emoji":"\ud83e\uddcb","pod_namespace":"whereami","pod_service_account":"whereami","project_id":"xrilb-multi-region-dns-test-01","timestamp":"2024-12-20T05:38:15","zone":"us-east4-a"}
```

### set up Cloud DNS for failover