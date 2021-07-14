# OSM Canary Deployments

This guide shows you how to use OSM and Flagger to automate canary deployments.

![Flagger OSM Traffic Split](https://raw.githubusercontent.com/fluxcd/flagger/main/docs/diagrams/flagger-osm-traffic-split.png)

## Prerequisites

Flagger requires a Kubernetes cluster **v1.16** or newer and OSM **0.9.1** or newer.

Install OSM with Prometheus enabled:

```bash
osm install --set=OpenServiceMesh.deployPrometheus=true --set=OpenServiceMesh.enablePermissiveTrafficPolicy=true
```

Install Flagger in the osm-system namespace:

```bash
kubectl apply -k github.com/fluxcd/flagger//kustomize/osm
```

Create the custom Prometheus metrics required for OSM canary analysis:

```bash
kubectl apply -k -k github.com/fluxcd/flagger//kustomize/osm/osm-metrics
```

## Bootstrap

Flagger takes a Kubernetes deployment and optionally a horizontal pod autoscaler (HPA),
then creates a series of objects (Kubernetes deployments, ClusterIP services and SMI traffic split).
These objects expose the application inside the mesh and drive the canary analysis and promotion.

Create a test namespace and enable OSM sidecar injection:

```bash
kubectl create namespace test
osm namespace add test
osm metrics enable --namespace test
```

Create the podinfo deployment:

```bash
kubectl apply -k https://github.com/fluxcd/flagger//kustomize/podinfo?ref=main
```

Deploy the load testing service to generate traffic during the canary analysis:

```bash
kubectl apply -k https://github.com/fluxcd/flagger//kustomize/tester?ref=main
```

Enable inbound port exclusion on the flagger load tester:

```bash
kubectl patch deployment flagger-loadtester -n test -p '{"spec": {"template": {"metadata": {"annotations": {"openservicemesh.io/inbound-port-exclusion-list": "80, 8080"}}}}}' --type=merge
```

Restart the flagger-loadtester pod to reflect the changes:

```bash
kubectl rollout restart deploy/flagger-loadtester -n test
```

Create a canary custom resource for the podinfo deployment:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
 name: podinfo
 namespace: test
spec:
 provider: smi:v1alpha2
 # deployment reference
 targetRef:
   apiVersion: apps/v1
   kind: Deployment
   name: podinfo
 # HPA reference (optional)
 autoscalerRef:
   apiVersion: autoscaling/v2beta2
   kind: HorizontalPodAutoscaler
   name: podinfo
 # the maximum time in seconds for the canary deployment
 # to make progress before it is rollback (default 600s)
 progressDeadlineSeconds: 60
 service:
   # ClusterIP port number
   port: 9898
   # container port number or name (optional)
   targetPort: 9898
 analysis:
   # schedule interval (default 60s)
   interval: 30s
   # max number of failed metric checks before rollback
   threshold: 5
   # max traffic percentage routed to canary
   # percentage (0-100)
   maxWeight: 50
   # canary increment step
   # percentage (0-100)
   stepWeight: 5
   # Prometheus checks
   metrics:
   - name: osm-request-success-rate
     # minimum req success rate (non 404 responses)
     # percentage (0-100)
     thresholdRange:
       min: 99
     interval: 1m
   - name: osm-request-duration
     # maximum req duration P99
     # milliseconds
     thresholdRange:
       max: 500
     interval: 30s
   # testing (optional)
   webhooks:
     - name: acceptance-test
       type: pre-rollout
       url: http://flagger-loadtester.test/
       timeout: 30s
       metadata:
         type: bash
         cmd: "curl -sd 'test' http://podinfo-canary.test:9898/token | grep token"
     - name: load-test
       type: rollout
       url: http://flagger-loadtester.test/
       metadata:
         cmd: "hey -z 2m -q 10 -c 2 http://podinfo-canary.test:9898/"
EOF
```
