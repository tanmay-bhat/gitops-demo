# GitOps Demo

## Prerequisite :

- Kind cluster
- Helm v3

### Directory Structure :

```bash
├── applications
│   ├── apps-of-apps.yaml
│   ├── argocd
│   │   └── argocd-server.yaml
│   ├── keda
│   │   ├── keda.yaml
│   │   └── scale-objects
│   │       └── keda-scale-object.yaml
│   ├── kyverno
│   │   ├── kyverno-policy
│   │   │   └── policy.yaml
│   │   └── kyverno.yaml
│   ├── metrics-server
│   │   └── metrics-server.yaml
│   ├── nginx-server
│   │   └── nginx-server.yaml
│   └── prometheus
│       └── prometheus-server.yaml
└── kubernetes-manifests
    ├── keda
    │   └── keda-scale-object.yaml
    ├── kyverno
    │   └── kyverno-pdb-enforce-policy.yaml
    └── nginx-server
        └── nginx-manifests.yaml
```

- `Applications` : Concist of ArgoCD application manifests.
- `kubernetes-manifests` :  Consist of kubernetes manifests which needs to applied to the cluster for example, kyverno policy, Keda auto-scale object etc.
- `apps-of-apps.yaml` : We’re using apps-of-apps model. Any app you define under the **Applications** directory will be automcailly onboarded via ArgoCD.

## 1.  Self managed ArgoCD

1. Create the cluster (local for demo) with [kind](https://kind.sigs.k8s.io/docs/user/quick-start/) : 

```bash
kind create cluster --name gitops
```

2. Get the kube-context and verify its set to correct cluster : 

```bash
kubectl config current-context

#expected output
kind-gitops
```

3. Clone the gitops repository : 

```bash
git clone https://github.com/tanmay-bhat/gitops-demo.git

cd gitops-demo/clusters/gitops-cluster/
```

4. Add the argocd helm repo & Install via helm chart :  

```bash

helm repo add argo https://argoproj.github.io/argo-helm && helm repo update

#create the values.yaml file : 

cat <<EOF >>values.yaml
server:
  configEnabled: true
  config:
    repositories: |
      - type: git
        url: https://github.com/tanmay-bhat/gitops-demo.git
      - name: argo-helm
        type: helm
        url: https://argoproj.github.io/argo-helm
    resource.compareoptions: |
      ignoreAggregatedRoles: true
  extraArgs:
  - "--disable-auth"
  metrics:
    enabled: false
  pdb:
    enabled: true
controller:
  pdb:
    enabled: true
  metrics:
    enabled: false
repoServer:
  pdb:
    enabled: true
  extraArgs:
    - "--parallelismlimit=1"
  metrics:
    enabled: false
applicationSet:
  enabled: false
dex:
  enabled: false
crds:
  install: true
  keep: false
EOF

#run the helm install command
helm install argocd -n argocd argo/argo-cd -f values.yaml --create-namespace --version 5.5.22
```

5. Onboard the rest of the apps GitOps way using argocd : 

```bash

kubectl apply -f apps-of-apps.yaml -n argocd
```

6. Access the ArgoCD UI by port-forwarding the service :  

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

7. In future, if we wanna make any change to ArgoCD configs or update its version, we just need to update the version in the argoCD application manifest `targetRevision`. (Ref Path : `applications/argocd/argocd-server.yaml`)
- ArgoCD is configured to manage itself, hence any change will be automatically synced.
- Same goes with other configs also, just edit the values section with the new changes.

<aside>
💡 For the demo purpose, argocd is configured with no auth. This should be done in Production.

</aside>

[https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/#kustomize](https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/#kustomize)

## 2. Enforcing PDB using Kyverno

- We’re using Kyverno as the policy engine for Kubernetes.
- If you look closely, we have applied a `ClusterPolicy` which will deny any incoming deployments which doesnt have any PodDisruptionBudget.
- This works by matching the labels of both PDB and deployments. if there’s no PDB with same set of labels, the deployment will be denied.

We can test it out by simply installing an example helm chart wihich doesnt have PDB enabled.

```bash
> helm install podinfo podinfo/podinfo

Error: admission webhook "validate.kyverno.svc-fail" denied the request:

policy Deployment/default/podinfo for resource violation:

require-pdb:
  require-pdb: There is no corresponding PodDisruptionBudget found for this Deployment.
```

## 3.  Nginx Auto-Scale based on CPU Load

- Nginx server is deployed via ArgoCD.
- Inorder to scale our pods based on CPU load of 50%, we’ll use  metrics server(installed via helm with ArgoCD) and fetch the CPU usage metrics.
- Once the usage is greater than 50% (specifed CPU request value), HPA controller will scale the pods.

We can test it out by running a simple load generator : 

```bash
kubectl run -i --tty load-generator --rm --image=curlimages/curl --restart=Never -- /bin/sh -c "while sleep 0.01; do seq 1 200 | xargs -n1 -P10  curl http://nginx; done"
```

- In the above command, we’re running 200 times GET request via CURL with 10 jobs parallely to the nginx kubernetes service.

Enable the HPA for nginx server by running the below command : 

```bash
kubectl autoscale deployment nginx-server --cpu-percent=50 --min=1 --max=10
```

- After a min, pods should get autoscaled :

```bash
kubectl get pod -l app=nginx
NAME                            READY   STATUS              RESTARTS   AGE
nginx-server-6f8b458d8f-fnqjd   0/2     ContainerCreating   0          1s
nginx-server-6f8b458d8f-t2xwp   2/2     Running             0          7m51s
nginx-server-6f8b458d8f-xjs9q   0/2     ContainerCreating   0          1s
nginx-server-6f8b458d8f-xl7sl   0/2     ContainerCreating   0          1s
```

- If you describe the HPA, you’ll see that HPA scaled because CPU usage was above 50%.

```bash
Normal   SuccessfulRescale             67s                    horizontal-pod-autoscaler  New size: 4; reason: cpu resource utilization (percentage of request) above target
```

### Scale Nginx server based on RPS

- We can use custom metrics with KEDA and scale the pods basd on number of requests we’re getting on nginx servers.
- The nginx server has an added container (nginx exporter) which will expose the prometheus exposition format metrics which will be later scraped by prometheus using its `scrape configs`.
- KEDA will query that mertrics and we can scale the pods once the rate of requests is greater than 10 or any number.
- The KEDA scale config looks like this :

```bash
triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus-server.default.svc.cluster.local:80
      metricName: nginx_http_requests_total
      threshold: '10'
      query: sum(rate(nginx_http_requests_total{job="nginx"}[1m]))
```

We can run the same above load-generator and the HPA (generated via KEDA) will display the below event : 

```bash
Normal   SuccessfulRescale     4m32s (x36 over 13m)  horizontal-pod-autoscaler  New size: 4; reason: external metric s0-prometheus-nginx_http_reques
ts_total above target
```

---

### Assumption :

- Nginx service is not exposed as NodePort or LB since its running in local cluster.
- If the same setup is replicated in a cloud cluster, service type can be changed to LB or using an Ingress (preferably nginx ingress controller)

### Extras : 
- Instead of manual curl requests, we can utilize https://locust.io/ as load test tool to visualize and automate too.
- Since we're using Kyverno with PDB deny policy, we can utilize ArgoCD notifications and send out email /slack message when a deployment is failed.