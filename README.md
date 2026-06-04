# Kubernetes on `kind` — Hands-on Lab with a Java App

A guided, copy-paste-able lab that takes you from **zero** to **HPA, Ingress,
RBAC, Taints, Affinity, Blue/Green & Canary** on a local **kind** cluster, using
a tiny Spring Boot app as the sample workload.

> Everything is local. No cloud, no cost.

---

## 0. Prerequisites

You need these installed (one-time):

| Tool | Why | macOS install |
|---|---|---|
| Docker Desktop / OrbStack / Colima | kind nodes ARE docker containers | `brew install --cask docker` |
| `kind` | spin up local k8s clusters | `brew install kind` |
| `kubectl` | talk to the cluster | `brew install kubectl` |
| `helm` (optional) | install ingress-nginx faster | `brew install helm` |

Verify:

```bash
docker version
kind version
kubectl version --client
```

The lab folder layout:

```
k8s-learning/
├── app/                              # Spring Boot source (pom.xml + src/)
├── docker/Dockerfile                 # multi-stage build
├── kind/kind-cluster.yaml            # 1 control-plane + 2 workers
├── manifests/
│   ├── 01-namespace/                 # Namespace, ResourceQuota, LimitRange
│   ├── 02-pods/                      # Pod, probes, sidecar
│   ├── 03-deployments/               # Deployment
│   ├── 04-services/                  # ClusterIP, NodePort, Headless
│   ├── 05-config-secrets/            # ConfigMap, Secret
│   ├── 06-resources-hpa/             # requests/limits, HPA
│   ├── 07-rbac/                      # SA, Role, RoleBinding, ClusterRole, ClusterRoleBinding
│   ├── 08-taints-affinity/           # nodeSelector, taint/toleration, affinity
│   ├── 09-ingress/                   # Ingress (NGINX)
│   ├── 10-deployment-strategies/     # RollingUpdate, Recreate, Blue/Green, Canary
│   ├── 11-statefulset/               # StatefulSet + headless service + PVC template
│   ├── 12-daemonset/                 # DaemonSet (one pod per node)
│   ├── 13-jobs/                      # Job + CronJob
│   ├── 14-storage/                   # StorageClass, PVC, Deployment with PVC
│   ├── 15-init-containers/           # initContainers (run-before-main)
│   ├── 16-network-policy/            # default deny + selective allow
│   └── 17-pdb/                       # PodDisruptionBudget
├── applications/                     # WHAT kinds of apps to deploy + recipes
│   ├── README.md                     #   - 13 workload patterns + decision tree
│   ├── catalog.html                  #   - interactive workload catalog
│   └── examples/                     #   - 7 runnable example manifests
├── dashboard.html                    # interactive learning dashboard
├── architecture.html                 # how a kubectl apply flows through k8s
└── docs/CHEATSHEET.md
```

---

## 1. Create the kind cluster

```bash
cd k8s-learning

kind create cluster --config kind/kind-cluster.yaml
```

Verify (you should see 3 nodes — 1 control-plane + 2 workers):

```bash
kubectl cluster-info --context kind-k8s-learn
kubectl get nodes -o wide
kubectl get nodes --show-labels
```

What you should see:

- `k8s-learn-control-plane`  — Ready, label `ingress-ready=true`
- `k8s-learn-worker`         — Ready, labels `tier=frontend, disktype=ssd`
- `k8s-learn-worker2`        — Ready, labels `tier=backend, disktype=hdd`

> Switch context (if you have other clusters): `kubectl config use-context kind-k8s-learn`

Tear down at any time with `kind delete cluster --name k8s-learn`.

---

## 2. Build & load the Java image into kind

kind nodes have their own container runtime. Images on your laptop are NOT
visible to them automatically, so we build, then `kind load` the image
into every node.

> **Run these from `k8s-learning/`, NOT from `k8s-learning/docker/`.**
> The Dockerfile does `COPY app/pom.xml ...`, and `app/` lives one level up
> from the Dockerfile. The trailing `.` below is the *build context* — it must
> be `k8s-learning/` so `COPY` can find the sources. `-f docker/Dockerfile`
> just tells Docker where the Dockerfile is.

```bash
cd /path/to/k8s-learning            # IMPORTANT: must be the lab root, not docker/
docker build -t k8s-demo:1.0.0 -f docker/Dockerfile .

# v1.0.1 — used later for rolling-update demo. Just a re-tag, no rebuild.
docker tag k8s-demo:1.0.0 k8s-demo:1.0.1

kind load docker-image k8s-demo:1.0.0 --name k8s-learn
kind load docker-image k8s-demo:1.0.1 --name k8s-learn
```

Verify the image is on the nodes:

```bash
docker exec -it k8s-learn-worker crictl images | grep k8s-demo
```

> Why `imagePullPolicy: IfNotPresent` everywhere? Because we don't push to a
> registry — kubelet must use the locally-loaded image and not try to pull.

> **Build troubleshooting**
> - `failed to compute cache key: "/app/pom.xml": not found`
>   → you ran the build from inside `docker/`. `cd` back to `k8s-learning/`
>   (see the warning above).
> - `no match for platform in manifest: not found`
>   → the base image's tag isn't published for your CPU architecture
>   (common on Apple Silicon with `*-alpine` tags). The Dockerfile here uses
>   `eclipse-temurin:17-jre` (Debian-based, multi-arch). If you want Alpine
>   AND arm64, swap the runtime stage to `amazoncorretto:17-alpine`.

---

## 3. Namespaces, ResourceQuota & LimitRange

```bash
kubectl apply -f manifests/01-namespace/namespace.yaml
kubectl apply -f manifests/01-namespace/resource-quota.yaml
```

Inspect:

```bash
kubectl get ns
kubectl describe ns demo
kubectl get resourcequota -n demo
kubectl describe resourcequota demo-quota -n demo
kubectl get limitrange -n demo
kubectl describe limitrange demo-limits -n demo
```

> `ResourceQuota` caps the namespace total. `LimitRange` sets per-pod defaults
> and bounds. Both are namespace-scoped.

---

## 4. Pods (the smallest unit)

```bash
kubectl apply -f manifests/02-pods/01-pod-simple.yaml
kubectl apply -f manifests/02-pods/02-pod-with-probes.yaml
kubectl apply -f manifests/02-pods/03-pod-multi-container.yaml
```

Debug commands (memorise these — they're the same for every workload):

```bash
kubectl get pods -n demo -o wide
kubectl describe pod java-pod -n demo
kubectl logs java-pod -n demo
kubectl logs java-pod-sidecar -n demo -c log-sidecar      # specific container
kubectl exec -it java-pod -n demo -- sh
kubectl port-forward -n demo pod/java-pod 8080:8080       # then curl localhost:8080
```

Then in another terminal:

```bash
curl localhost:8080/
curl localhost:8080/info
```

Clean up the bare pods (we'll use Deployments from now on):

```bash
kubectl delete -f manifests/02-pods/
```

---

## 5. Deployments

```bash
kubectl apply -f manifests/03-deployments/01-deployment.yaml

kubectl get deploy,rs,pods -n demo
kubectl rollout status deploy/k8s-demo -n demo
kubectl describe deploy k8s-demo -n demo
```

**Scaling** — manual:

```bash
kubectl scale deploy/k8s-demo -n demo --replicas=5
kubectl get pods -n demo -w           # Ctrl-C to stop watching
kubectl scale deploy/k8s-demo -n demo --replicas=2
```

**Self-healing test** — kill a pod and watch it come back:

```bash
POD=$(kubectl get pods -n demo -l app=k8s-demo -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod $POD -n demo
kubectl get pods -n demo -w
```

**Rollout / rollback**:

```bash
kubectl set image deploy/k8s-demo java-app=k8s-demo:1.0.1 -n demo
kubectl rollout status deploy/k8s-demo -n demo
kubectl rollout history deploy/k8s-demo -n demo
kubectl rollout undo deploy/k8s-demo -n demo
```

---

## 6. Services (ClusterIP, NodePort, Headless)

```bash
kubectl apply -f manifests/04-services/01-service-clusterip.yaml
kubectl apply -f manifests/04-services/02-service-nodeport.yaml
kubectl apply -f manifests/04-services/03-service-headless.yaml

kubectl get svc -n demo
kubectl get endpoints -n demo            # the actual pod IPs behind each svc
kubectl describe svc k8s-demo -n demo
```

**Test ClusterIP from inside the cluster:**

```bash
kubectl run -n demo tmp --rm -it --restart=Never --image=curlimages/curl -- \
  sh -c 'curl -s http://k8s-demo/info'
```

**Test NodePort from your laptop** (we mapped 30080 in kind config):

```bash
curl http://localhost:30080/info
```

**Inspect headless DNS:**

```bash
kubectl run -n demo tmp --rm -it --restart=Never --image=busybox -- \
  nslookup k8s-demo-headless.demo.svc.cluster.local
```

You will see one A record per pod, instead of one cluster IP.

---

## 7. ConfigMaps & Secrets

```bash
kubectl apply -f manifests/05-config-secrets/01-configmap.yaml
kubectl apply -f manifests/05-config-secrets/02-secret.yaml
kubectl apply -f manifests/05-config-secrets/03-deployment-with-config.yaml

kubectl get cm,secret -n demo
kubectl describe cm k8s-demo-config -n demo
kubectl get secret k8s-demo-secret -n demo -o yaml         # values base64-encoded
kubectl get secret k8s-demo-secret -n demo -o jsonpath='{.data.DB_PASSWORD}' | base64 -d ; echo
```

Verify the env vars and the mounted file inside the pod:

```bash
POD=$(kubectl get pods -n demo -l app=k8s-demo-cfg -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n demo $POD -- env | grep -E 'APP_|DB_'
kubectl exec -n demo $POD -- cat /config/application.yaml
```

Hit the app's `/info` endpoint to see config + secret reflected:

```bash
kubectl port-forward -n demo svc/k8s-demo-cfg 8081:80
# new terminal:
curl localhost:8081/info
```

> Editing a ConfigMap does NOT auto-restart pods consuming it as env vars.
> Force a rollout: `kubectl rollout restart deploy/k8s-demo-cfg -n demo`

---

## 8. Resource requests/limits + HPA

Install metrics-server (kind doesn't ship it):

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# kind uses self-signed kubelet certs - patch metrics-server to accept them
kubectl patch -n kube-system deploy metrics-server --type=json -p='[
  {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}
]'

kubectl rollout status -n kube-system deploy/metrics-server
kubectl top nodes
kubectl top pods -n demo
```

Apply the deployment + HPA:

```bash
kubectl apply -f manifests/06-resources-hpa/01-deployment-resources.yaml
kubectl apply -f manifests/06-resources-hpa/02-hpa.yaml

kubectl get hpa -n demo
kubectl describe hpa k8s-demo-hpa -n demo
```

**Generate load** (the `/cpu` endpoint burns CPU for 3s per call):

```bash
# Terminal 1: watch HPA + pods
watch -n 2 'kubectl get hpa,pods -n demo -l app=k8s-demo-hpa'

# Terminal 2: hammer it
kubectl run -n demo loadgen --rm -it --restart=Never --image=busybox -- \
  sh -c 'while true; do wget -q -O- http://k8s-demo-hpa/cpu; done'
```

Within 1–2 minutes you should see replicas climb 1 → 2 → 3 → 5 (max).
Stop the loadgen (Ctrl-C). After ~5 min stabilization, replicas drop back.

> If `kubectl top` says "metrics not available", wait 60s after metrics-server
> rolls out, OR re-check the `--kubelet-insecure-tls` patch was applied.

---

## 9. RBAC — ServiceAccount, Role, RoleBinding, ClusterRole, ClusterRoleBinding

```bash
kubectl apply -f manifests/07-rbac/01-serviceaccount.yaml
kubectl apply -f manifests/07-rbac/02-role-rolebinding.yaml
kubectl apply -f manifests/07-rbac/03-clusterrole-clusterrolebinding.yaml
kubectl apply -f manifests/07-rbac/04-deployment-with-sa.yaml
```

Inspect:

```bash
kubectl get sa,role,rolebinding -n demo
kubectl get clusterrole cluster-readonly
kubectl get clusterrolebinding cluster-readonly-binding
kubectl describe rolebinding demo-reader-binding -n demo
```

**Verify what the SA can / cannot do** with `kubectl auth can-i`:

```bash
# Should be YES (granted by Role demo-reader):
kubectl auth can-i list pods       -n demo --as=system:serviceaccount:demo:k8s-demo-sa
kubectl auth can-i get  configmaps -n demo --as=system:serviceaccount:demo:k8s-demo-sa
# Should be NO (not granted):
kubectl auth can-i create pods     -n demo --as=system:serviceaccount:demo:k8s-demo-sa
kubectl auth can-i list pods       -n demo-prod --as=system:serviceaccount:demo:k8s-demo-sa
# Should be YES (granted by ClusterRole cluster-readonly):
kubectl auth can-i list nodes      --as=system:serviceaccount:demo:k8s-demo-sa
kubectl auth can-i list namespaces --as=system:serviceaccount:demo:k8s-demo-sa
```

**Use the SA token from inside a pod** to call the API:

```bash
POD=$(kubectl get pods -n demo -l app=k8s-demo-rbac -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n demo $POD -- sh -c '
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
curl --cacert $CACERT -H "Authorization: Bearer $TOKEN" \
  https://kubernetes.default.svc/api/v1/namespaces/demo/pods | head -40
'
```

> Mental model:
> - **Role / RoleBinding** = namespace-scoped permissions
> - **ClusterRole / ClusterRoleBinding** = cluster-scoped permissions
> - You can also *RoleBind* a ClusterRole to give a reusable role inside ONE namespace.

---

## 10. Taints, Tolerations & Pod Affinity (multi-node)

**A) `nodeSelector`** — schedule only on `tier=frontend` worker:

```bash
kubectl apply -f manifests/08-taints-affinity/01-deployment-nodeselector.yaml
kubectl get pods -n demo -l app=k8s-demo-frontend -o wide
# all pods land on the worker labelled tier=frontend
```

**B) Taint + Toleration** — dedicate `worker2` to backend pods:

```bash
# 1. taint the node (ONLY pods with matching toleration may run there)
kubectl taint nodes k8s-learn-worker2 dedicated=backend:NoSchedule

# 2. apply the deployment carrying the toleration
kubectl apply -f manifests/08-taints-affinity/02-taint-toleration.yaml

kubectl get pods -n demo -l app=k8s-demo-backend -o wide
kubectl describe node k8s-learn-worker2 | grep -A2 Taints
```

To remove the taint later:

```bash
kubectl taint nodes k8s-learn-worker2 dedicated=backend:NoSchedule-
```

**C) Pod anti-affinity (HA spread)**:

```bash
kubectl apply -f manifests/08-taints-affinity/03-pod-affinity.yaml
kubectl get pods -n demo -l app=k8s-demo-spread -o wide
```

You should see each replica on a DIFFERENT node (anti-affinity by hostname).
With only 2 worker nodes and 3 replicas, the 3rd will be `Pending`.
That's the tradeoff: a hard `requiredDuringScheduling` rule will refuse to
schedule rather than break the rule. Switch to
`preferredDuringScheduling` for soft spreading.

```bash
kubectl describe pod -n demo -l app=k8s-demo-spread | grep -A3 Events
```

---

## 11. Ingress (NGINX)

Install ingress-nginx (the kind-flavoured manifest):

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=180s

kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

Apply the Ingress:

```bash
kubectl apply -f manifests/09-ingress/01-ingress.yaml
kubectl get ingress -n demo
kubectl describe ingress k8s-demo-ingress -n demo
```

Test (port 80 of kind container is mapped to your laptop):

```bash
curl -H "Host: demo.local" http://localhost/
curl -H "Host: demo.local" http://localhost/cfg/info
```

Or add to `/etc/hosts`:

```
127.0.0.1   demo.local
```

…and just `curl http://demo.local/`.

---

## 12. Deployment Strategies

**RollingUpdate** (default):

```bash
kubectl apply -f manifests/10-deployment-strategies/01-rolling-update.yaml
kubectl rollout status deploy/k8s-demo-rolling -n demo

# Trigger a rollout to v1.0.1
kubectl set image deploy/k8s-demo-rolling java-app=k8s-demo:1.0.1 -n demo
kubectl rollout status deploy/k8s-demo-rolling -n demo
kubectl rollout history deploy/k8s-demo-rolling -n demo

# Pause / resume / rollback
kubectl rollout pause  deploy/k8s-demo-rolling -n demo
kubectl rollout resume deploy/k8s-demo-rolling -n demo
kubectl rollout undo   deploy/k8s-demo-rolling -n demo
```

**Recreate**:

```bash
kubectl apply -f manifests/10-deployment-strategies/02-recreate.yaml
kubectl set image deploy/k8s-demo-recreate java-app=k8s-demo:1.0.1 -n demo
kubectl get pods -n demo -l app=k8s-demo-recreate -w
# you'll see all old pods Terminate first, THEN new ones Pending → Running
```

**Blue/Green** (manual flip via service selector):

```bash
kubectl apply -f manifests/10-deployment-strategies/03-bluegreen.yaml

# right now Service selects version=blue
kubectl run -n demo tmp --rm -it --restart=Never --image=curlimages/curl -- \
  sh -c 'for i in 1 2 3 4 5; do curl -s http://k8s-demo-bluegreen/ ; echo; done'

# flip to green
kubectl patch svc k8s-demo-bluegreen -n demo \
  -p '{"spec":{"selector":{"app":"k8s-demo-bluegreen","version":"green"}}}'

kubectl run -n demo tmp --rm -it --restart=Never --image=curlimages/curl -- \
  sh -c 'for i in 1 2 3 4 5; do curl -s http://k8s-demo-bluegreen/ ; echo; done'
```

**Canary** (replica-weighted):

```bash
kubectl apply -f manifests/10-deployment-strategies/04-canary.yaml

kubectl run -n demo tmp --rm -it --restart=Never --image=curlimages/curl -- \
  sh -c 'for i in $(seq 1 20); do curl -s http://k8s-demo-canary/ ; echo; done'

# About 4 of 5 should say "stable v1", 1 of 5 "canary v2".
# Promote canary by scaling stable down and canary up:
kubectl scale deploy/k8s-demo-stable -n demo --replicas=0
kubectl scale deploy/k8s-demo-canary -n demo --replicas=4
```

---

## 13. StatefulSet + Headless Service

```bash
kubectl apply -f manifests/11-statefulset/01-statefulset.yaml
kubectl rollout status sts/k8s-demo-sts -n demo
kubectl get sts,pods,pvc,svc -n demo -l app=k8s-demo-sts
```

Inspect the **stable, ordered identity** each Pod has:

```bash
kubectl get pods -n demo -l app=k8s-demo-sts -o wide
# you should see:
#   k8s-demo-sts-0   <node>
#   k8s-demo-sts-1   <node>
#   k8s-demo-sts-2   <node>

# and PVCs created automatically by volumeClaimTemplates:
kubectl get pvc -n demo
# data-k8s-demo-sts-0, data-k8s-demo-sts-1, data-k8s-demo-sts-2
```

Resolve each replica's stable DNS name:

```bash
kubectl run -n demo dns-tmp --rm -it --restart=Never --image=busybox:1.36 -- \
  sh -c 'for i in 0 1 2; do
           echo "--- k8s-demo-sts-$i ---";
           nslookup k8s-demo-sts-$i.k8s-demo-sts.demo.svc.cluster.local;
         done'
```

Prove the volume is **per-pod and persistent**:

```bash
# write a unique marker INTO pod 0's volume
kubectl exec -n demo k8s-demo-sts-0 -- sh -c 'echo "hello from pod-0 at $(date)" > /var/data/marker.txt'

# delete pod 0
kubectl delete pod -n demo k8s-demo-sts-0
kubectl wait --for=condition=Ready pod/k8s-demo-sts-0 -n demo --timeout=60s

# the marker is STILL THERE - same PVC reattached
kubectl exec -n demo k8s-demo-sts-0 -- cat /var/data/marker.txt
```

> Mental model: a Deployment treats Pods as cattle (interchangeable). A
> StatefulSet treats them as pets — each one has a name, a slot, and its own
> disk. Use it for **databases, queues, anything where pod identity matters**.

---

## 14. DaemonSet

A Pod **per node** — for log collectors, metrics agents, CNI, kube-proxy.

```bash
kubectl apply -f manifests/12-daemonset/01-daemonset.yaml

kubectl get ds -n demo
kubectl get pods -n demo -l app=node-info-agent -o wide
# one pod on each WORKER node (control-plane is excluded by its taint by default)
```

Watch each agent's heartbeat (it logs node + pod identity every 30s):

```bash
kubectl logs -f -n demo -l app=node-info-agent --max-log-requests=10 --prefix=true
```

Make it run on the control-plane too — uncomment the `tolerations:` block in
`manifests/12-daemonset/01-daemonset.yaml`, then `kubectl apply -f` again.

---

## 15. Job & CronJob

**Job** — run-to-completion (data migration, backup, batch).

```bash
kubectl apply -f manifests/13-jobs/01-job.yaml

kubectl get job,pods -n demo -l app=data-migration
kubectl logs -n demo -l app=data-migration --tail=50

# Wait for it to finish:
kubectl wait --for=condition=complete -n demo job/data-migration --timeout=120s
kubectl get job data-migration -n demo
# COMPLETIONS = 3/3, parallelism limited active to 2
```

**CronJob** — Job on a schedule (every 2 minutes in this demo):

```bash
kubectl apply -f manifests/13-jobs/02-cronjob.yaml

kubectl get cronjob -n demo
kubectl get jobs    -n demo --sort-by=.metadata.creationTimestamp

# trigger one manually instead of waiting:
kubectl create job --from=cronjob/hello-cron hello-once -n demo
kubectl logs -n demo -l job-name=hello-once
```

> `concurrencyPolicy: Forbid` skips a tick if the previous run is still going.
> `successfulJobsHistoryLimit: 3` keeps the last 3 successful Jobs around so you
> can inspect them; older ones are garbage-collected.

---

## 16. Storage — PV, PVC, StorageClass

kind ships with a default `standard` StorageClass (local-path-provisioner) so
this works out of the box.

```bash
kubectl get storageclass                              # 'standard' is the default
kubectl describe storageclass standard

kubectl apply -f manifests/14-storage/02-pvc.yaml
kubectl apply -f manifests/14-storage/03-deployment-with-pvc.yaml

kubectl get pvc,pv -n demo
kubectl get pods -n demo -l app=k8s-demo-storage -o wide
```

Persist data, kill the pod, see the data survive:

```bash
POD=$(kubectl get pods -n demo -l app=k8s-demo-storage -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n demo $POD -- sh -c 'echo "saved at $(date)" > /var/data/it-survived.txt'

kubectl delete pod $POD -n demo
kubectl wait --for=condition=Ready -l app=k8s-demo-storage pod -n demo --timeout=60s

POD=$(kubectl get pods -n demo -l app=k8s-demo-storage -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n demo $POD -- cat /var/data/it-survived.txt
```

> kind uses `WaitForFirstConsumer` binding — the PVC stays Pending until a Pod
> using it gets scheduled. Then a PV is provisioned on that pod's node. This is
> why a kind PVC sometimes looks "stuck" until you `kubectl apply` the
> Deployment.

---

## 17. Init Containers

initContainers run **before** main containers, in order, until they all
exit successfully.

```bash
kubectl apply -f manifests/15-init-containers/01-init-pod.yaml

kubectl get pod java-pod-init -n demo -w
# you'll see STATUS go: Init:0/2 -> Init:1/2 -> Init:2/2 -> PodInitializing -> Running
```

Inspect the init phases:

```bash
kubectl describe pod java-pod-init -n demo | grep -A5 "Init Containers:"
kubectl logs java-pod-init -n demo -c wait-for-service
kubectl logs java-pod-init -n demo -c prepare-data
kubectl exec  java-pod-init -n demo -- cat /work/seed.txt   # written by init, read by main
```

> initContainer = "do something, then exit". Sidecar = "stay running alongside
> the app". Use init for **DB migrations, dependency-waits, generating config
> files, fixing volume permissions**.

---

## 18. NetworkPolicy

By default, every pod can talk to every other pod. NetworkPolicy adds firewall
rules — but they're enforced by the **CNI plugin**.

> **kind's default kindnet does NOT enforce NetworkPolicy.** Install Calico (or
> Cilium) first, otherwise the policies you create are decorative.

```bash
# Install Calico (replaces kindnet for new pods)
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

kubectl wait --for=condition=Ready -n kube-system pods -l k8s-app=calico-node --timeout=180s
kubectl rollout restart deploy -n demo               # so existing pods get Calico-managed networking
```

Apply the policies:

```bash
kubectl apply -f manifests/16-network-policy/01-default-deny.yaml
kubectl apply -f manifests/16-network-policy/02-allow-from-frontend.yaml

kubectl get networkpolicy -n demo
kubectl describe networkpolicy default-deny-all -n demo
```

Verify with two probes — one from a pod that *should* be allowed, one that *shouldn't*:

```bash
# DENIED: tmp pod has no role=frontend label
kubectl run -n demo deny-tmp --rm -it --restart=Never --image=curlimages/curl -- \
  curl -sS --max-time 4 http://k8s-demo/info ; echo "exit=$?"
# expect: timeout / connection refused / non-zero exit

# ALLOWED: same pod with the right label
kubectl run -n demo allow-tmp --rm -it --restart=Never \
  --labels='role=frontend' --image=curlimages/curl -- \
  curl -sS --max-time 4 http://k8s-demo/info
# expect: a JSON response from the app
```

Common pitfall: forget `allow-egress-dns` and *nothing* in the namespace can
even resolve `k8s-demo`. Always allow DNS in egress when you go default-deny.

---

## 19. PodDisruptionBudget

Protects you during **voluntary** disruptions (`kubectl drain`, autoscaler).
Does NOT protect against node hardware failure or OOM — that's what replicas +
spread are for.

```bash
kubectl apply -f manifests/17-pdb/01-pdb.yaml
kubectl get pdb -n demo
kubectl describe pdb k8s-demo-pdb -n demo
# ALLOWED DISRUPTIONS = currentHealthy - minAvailable
```

Test it by draining a node — if draining would violate the PDB, kubectl drain
blocks (until you `--force` it):

```bash
# scale up first so we have something to evict
kubectl scale deploy/k8s-demo -n demo --replicas=3

NODE=$(kubectl get pods -n demo -l app=k8s-demo -o jsonpath='{.items[0].spec.nodeName}')
kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data --disable-eviction=false
# you'll see drain wait for new replicas before evicting old ones
kubectl uncordon $NODE
```

---

## 20. Cleanup

```bash
# Drop the whole demo namespace (deletes everything inside it):
kubectl delete ns demo demo-prod

# Or nuke the whole cluster:
kind delete cluster --name k8s-learn
```

---

## 21. What to read next

1. `applications/README.md` — **what kinds of apps you can deploy on K8s** + 13 patterns + decision tree.
2. `applications/catalog.html` — interactive, color-coded workload catalog (filter by stateful/HTTP/ML/...).
3. `docs/CHEATSHEET.md` — every debugging command grouped by topic.
4. `architecture.html` — how a `kubectl apply` flows through the control plane.
5. `dashboard.html` — interactive lab tracker.
6. **CSI drivers** & cloud-specific volumes (EBS, GCE PD, Azure Disk).
7. **Operators / CRDs** — extend k8s with your own kinds.
8. **Service Mesh** (Istio, Linkerd) — mTLS, traffic shifting, observability.
9. **Helm / Kustomize** — template & overlay your manifests.
10. **Argo Rollouts / Flagger** — real progressive delivery with metrics.
11. **OPA / Kyverno** — policy-as-code admission control.

Happy shipping.
