# CKAD Practice Questions & Answers

A collection of practical Kubernetes (CKAD-style) questions with complete solutions.

---

## 1. NetworkPolicy – Restrict Database Access

### Question

A backend database deployment is currently running in the `prod` namespace. For security hardening, all inbound traffic to this database must be blocked by default, with only explicit microservices permitted to reach it.

**Context & Requirements**

- **Target Namespace:** `prod`
- **Target Pods:** Pods with label `app=db-backend`
- **NetworkPolicy Name:** `db-backend-netpol`
- **Traffic Rules:**
  - Deny all incoming traffic to `app=db-backend` pods by default.
  - Allow incoming TCP traffic on port `5432` only from pods in the `prod` namespace with the label `role=api`.
  - Allow incoming TCP traffic on port `5432` only from pods in the `reporting` namespace with the label `role=analytics`.  
    *(Assume the `reporting` namespace already has the label `kubernetes.io/metadata.name=reporting`.)*
- Outgoing (egress) traffic from `app=db-backend` must remain completely unrestricted.

### Answer

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-backend-netpol
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: db-backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: reporting
      podSelector:
        matchLabels:
          role: analytics
    ports:
    - protocol: TCP
      port: 5432
  - from:
    - podSelector:
        matchLabels:
          role: api
    ports:
    - protocol: TCP
      port: 5432
```

---

## 2. CronJob + Manual Job Trigger

### Question

A development team requires a scheduled data cleanup task in the `maintenance` namespace. Additionally, they need an immediate verification run executed right away without waiting for the scheduled trigger.

**Context & Requirements**

- **Target Namespace:** `maintenance` (create it if it doesn't exist)
- **CronJob Name:** `log-cleaner-cj`
- **Schedule:** Run every 30 minutes (`*/30 * * * *`)
- **Container Image:** `busybox:1.36`
- **Command:** `/bin/sh -c "echo Cleanup completed at $(date)"`
- **Job History Limits:** Keep 3 successful completed jobs and 1 failed job
- **Restart Policy:** `OnFailure`
- **Manual Execution:** Immediately trigger an ad-hoc Job named `log-cleaner-manual` from this CronJob.

### Answer

```bash
kubectl create ns maintenance
```

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: log-cleaner-cj
  namespace: maintenance
spec:
  schedule: "*/30 * * * *"
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: log-cleaner-cj
            image: busybox:1.36
            command:
            - /bin/sh
            - -c
            - echo Cleanup completed at $(date)
```

```bash
kubectl apply -f cj.yaml
kubectl create job log-cleaner-manual --from=cronjob/log-cleaner-cj -n maintenance
```

---

## 3. ResourceQuota + Compliant Pod

### Question

A new tenant namespace named `dev-team` requires strict compute resource controls to prevent resource starvation on the cluster. You are tasked with creating the quota and testing deployment compliance.

**Context & Requirements**

- **Namespace:** `dev-team` (create it if missing)
- **ResourceQuota Name:** `compute-quota`
- **Hard Limits:**
  - Maximum number of Pods: `4`
  - Total CPU requests: `1` (or `1000m`)
  - Total CPU limits: `2` (or `2000m`)
  - Total Memory requests: `1Gi`
  - Total Memory limits: `2Gi`
- **Verification / Compliance:**
  - Deploy a single Pod named `test-worker` using image `nginx:1.25-alpine`
  - Requests: `250m` CPU, `256Mi` Memory
  - Limits: `500m` CPU, `512Mi` Memory
  - Ensure the pod reaches `Running` status.

### Answer

```bash
kubectl create ns dev-team

kubectl create quota compute-quota \
  --hard=requests.cpu=1,limits.cpu=2,requests.memory=1Gi,limits.memory=2Gi,pods=4 \
  -n dev-team

kubectl run test-worker -n dev-team \
  --image=nginx:1.25-alpine \
  --requests="cpu=250m,memory=256Mi" \
  --limits="cpu=500m,memory=512Mi"
```

---

## 4. Secret + Environment Variable + Volume Mount

### Question

A web backend in the `auth` namespace needs to securely retrieve sensitive credentials. You must create the Secret and mount it into a Pod using two different mechanisms.

**Context & Requirements**

- **Namespace:** Create the namespace `auth` if it does not exist.
- **Secret:** Create a generic Secret named `db-credentials` with:
  - `DB_USER=admin`
  - `DB_PASS=secret123`
- **Pod:** Deploy a Pod named `auth-backend` using `nginx:1.25-alpine`:
  - Inject the key `DB_USER` from the Secret as an environment variable named `DATABASE_USER`.
  - Mount the entire `db-credentials` Secret as a volume at `/etc/db-secret` (read-only).
- Verify that the Pod transitions to `Running`.

### Answer

```bash
kubectl create ns auth

kubectl create secret generic db-credentials \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASS=secret123 \
  -n auth
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: auth-backend
  namespace: auth
spec:
  containers:
  - name: auth-backend
    image: nginx:1.25-alpine
    env:
    - name: DATABASE_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: DB_USER
    volumeMounts:
    - name: db-secret-volume
      mountPath: /etc/db-secret
      readOnly: true
  volumes:
  - name: db-secret-volume
    secret:
      secretName: db-credentials
```

---

## 5. Deployment Rollout + Rollback

### Question

An existing production deployment named `frontend-app` in the `prod` namespace needs an image update. However, the new release introduces a fatal misconfiguration, requiring you to monitor the rollout, detect the failure, and revert cleanly to the working revision.

**Context & Requirements**

1. Create namespace `prod` if it doesn't exist.
2. Create a deployment named `frontend-app` with 3 replicas using image `nginx:1.24-alpine`.
3. Update the container image to `nginx:1.999-broken`.
4. Monitor the rollout status until it fails to progress.
5. Inspect the rollout history.
6. Roll back the deployment to the previous stable revision.
7. Verify that all 3 replicas return to the `Running` state on image `nginx:1.24-alpine`.

### Answer

```bash
# 1. Create namespace & initial deployment
kubectl create ns prod
kubectl create deploy frontend-app --image=nginx:1.24-alpine --replicas=3 -n prod

# 2. Update the image
kubectl set image deploy/frontend-app frontend-app=nginx:1.999-broken -n prod

# 3. Watch status fail
kubectl rollout status deploy/frontend-app -n prod

# 4. View history
kubectl rollout history deploy/frontend-app -n prod

# 5. Roll back
kubectl rollout undo deploy/frontend-app -n prod

# 6. Verify
kubectl get pods -n prod
```

---

## 6. Build, Test and Save Container Image

### Question

A new microservice Dockerfile has been placed on the host at `/opt/app/Dockerfile`. You must build the image locally, verify it functions by running a temporary container, and export the resulting image into an archive.

**Context & Requirements**

- Detect whether `docker` or `podman` is available and use the existing runtime.
- Build the image from `/opt/app` and tag it as `custom-app:v1.0`.
- Run a temporary container named `test-container` in detached mode, mapping host port `8088` → container port `80`.
- Confirm it responds on `http://localhost:8088` using `curl`.
- Stop and remove the test container.
- Export/save the image as `/opt/app/custom-app.tar`.

### Answer

```bash
# 1. Detect runtime
which podman || which docker

# 2. Build and tag
podman build -t custom-app:v1.0 /opt/app
# (or use docker if podman is not available)

# 3. Run detached on port 8088
podman run -d --name test-container -p 8088:80 custom-app:v1.0

# 4. Verify
curl http://localhost:8088

# 5. Cleanup
podman rm -f test-container

# 6. Save the image
podman save -o /opt/app/custom-app.tar custom-app:v1.0
```

---

## 7. Deployment + Manual Scale + HPA

### Question

Create a deployment and configure Horizontal Pod Autoscaler for it.

**Context & Requirements**

- **Namespace:** Create `ecommerce` if it does not exist.
- **Deployment:** `order-api` using `nginx:1.25-alpine`, initial replicas = 2.
- Set container CPU requests to `100m` (required for HPA metric calculations).
- Manually scale the deployment to 5 replicas.
- Create an HPA targeting `order-api`:
  - min pods = 2
  - max pods = 8
  - target CPU utilization = 75%
- Verify the HPA exists under the `ecommerce` namespace.

### Answer

```bash
kubectl create ns ecommerce

kubectl create deploy order-api --replicas=2 --image=nginx:1.25-alpine -n ecommerce
kubectl set resources deploy order-api --requests=cpu=100m -n ecommerce

kubectl scale deploy order-api --replicas=5 -n ecommerce

kubectl autoscale deploy order-api --min=2 --max=8 --cpu-percentage=75 -n ecommerce

kubectl get hpa -n ecommerce
```

---

## 8. ConfigMaps + Volume Mount

### Question

Create two ConfigMaps and mount one of them into a Pod.

**Context & Requirements**

- **Static ConfigMap:** Create `app-config` in the `default` namespace with key `special.how` and value `very`.
- **File-based ConfigMap:** Create `ui-properties` from a local file.
- Deploy a Pod named `config-web` (`nginx:1.25-alpine`) that mounts the `ui-properties` ConfigMap as a file at `/var/lib/web-config`.

### Answer

```bash
kubectl create configmap app-config --from-literal=special.how=very

kubectl create configmap ui-properties --from-file=ui.properties=/tmp/ui.properties
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-web
  namespace: default
spec:
  containers:
  - name: config-web
    image: nginx:1.25-alpine
    volumeMounts:
    - name: ui-properties-volume
      mountPath: /var/lib/web-config
      readOnly: true
  volumes:
  - name: ui-properties-volume
    configMap:
      name: ui-properties
```

---

## 9. Security Context – Non-root + Capabilities

### Question

Demonstrate how to strip a container of unnecessary privileges (defense against the "Root-by-Default" vulnerability).

**Context & Requirements**

- **Namespace:** `security-lab`
- **Pod Name:** `hardened-svc`
- **Image:** `busybox:1.36`
- Run as User ID `1001`
- Add Linux capability `SYS_TIME`
- Set `allowPrivilegeEscalation: false`

### Answer

```bash
kubectl create ns security-lab
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened-svc
  namespace: security-lab
spec:
  securityContext:
    runAsUser: 1001
  containers:
  - name: hardened-svc
    image: busybox:1.36
    command:
    - /bin/sh
    - -c
    - sleep 3600
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        add:
        - SYS_TIME
```

---

## 10. ServiceAccount + Role + RoleBinding

### Question

Follow the principle of least privilege by assigning a workload a specific identity with only the required permissions.

**Context & Requirements**

- **Namespace:** `lead-developer`
- **ServiceAccount:** `pod-reader-sa`
- **Role:** `pod-read-access` that allows `get`, `watch`, `list` on `pods` and `secrets`
- **RoleBinding:** Bind the Role to the ServiceAccount
- Deploy a Pod that explicitly uses this ServiceAccount.

### Answer

```bash
kubectl create ns lead-developer
kubectl create sa pod-reader-sa -n lead-developer

kubectl create role pod-read-access \
  --verb=get,list,watch \
  --resource=pods,secrets \
  -n lead-developer

kubectl create rolebinding read-pods-binding \
  --role=pod-read-access \
  --serviceaccount=lead-developer:pod-reader-sa \
  -n lead-developer
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-reader
  namespace: lead-developer
spec:
  serviceAccountName: pod-reader-sa
  containers:
  - name: nginx
    image: nginx:1.25-alpine
```

---

## Notes

- All commands assume `kubectl` is available and configured for the target cluster.
- Some questions intentionally use slightly different wording or minor variations of the same core task (common in real exam scenarios).
- Always verify resources after creation with `kubectl get`, `kubectl describe`, and `kubectl logs` as appropriate.

### Question
## 11. 
You have an existing deployment named api-server in the backend namespace. Using a single kubectl set command, update its container (also named api-server) to have:

CPU request: 150m

Memory request: 256Mi

CPU limit: 300m

Memory limit: 512Mi

### Answer
```
k set resources deployment api-server -n backend -c api-server --requests=cpu=150m,memory=256Mi --limits=cpu=300m,memory=512Mi
```
### Question
## 12.
Create an Ingress resource named web-ingress in namespace frontend that routes host app.example.com on path /static to a Service named static-svc on port 8080.

### Answer
```
k create ingress web-ingress -n frontend --rule="app.example.com/static=static-svc:8080"
```

### Question
## 13.
In namespace testing, spin up a pod named temp-worker using image busybox:1.36 in one command:

Pass an environment variable: MODE=debug

Set CPU request: 50m

Set Memory limit: 64Mi

Command to run inside: sleep 3600

### Answer
```
k run temp-worker --image=busybox:1.36 -n testing --env="MODE=debug" --requests=cpu=50m --limits=memory=64Mi --command -- sleep 3600
```
### Question
## 14.
Create a ConfigMap named app-config in namespace prod:

Literal key APP_COLOR set to dark-blue

Literal key APP_RETRIES set to 5

### Answer
```
k create cm app-config -n prod --from-literal=APP_COLOR=dark-blue --from-literal=APP_RETRIES=5
```
### Question
## 15.
Create a HorizontalPodAutoscaler named cache-hpa targeting deployment redis-cache in namespace cache-system:

Minimum replicas: 3

Maximum replicas: 10
### Answer
```
kubectl autoscale deployment redis-cache -n cache-system --min=3 --max=10 --cpu-percent=80 --name=cache-hpa
```
### Question
## 16.
The staging namespace requires a multi-container Pod where multiple configuration keys are loaded simultaneously into all containers.

- Create a ConfigMap named `app-config` in the `staging` namespace with the following keys:
  - `UI_COLOR=blue`
  - `LOG_LEVEL=info`
  - `FEATURE_TOGGLE=true`
- Create a Pod named `multi-app` with two containers (`c1` and `c2`) using the image `nginx:1.25-alpine`.
- Use the `envFrom` field in **both** containers to inject **all** keys from the ConfigMap as environment variables (do not map individual keys).

### Answer
```bash
kubectl create cm app-config \
  --from-literal=UI_COLOR=blue \
  --from-literal=LOG_LEVEL=info \
  --from-literal=FEATURE_TOGGLE=true \
  -n staging

apiVersion: v1
kind: Pod
metadata:
  name: multi-app
  namespace: staging
spec:
  containers:
    - name: c1
      image: nginx:1.25-alpine
      envFrom:
        - configMapRef:
            name: app-config
    - name: c2
      image: nginx:1.25-alpine
      envFrom:
        - configMapRef:
            name: app-config

kubectl apply -f scenario2.yaml
```
### Question
## 17.
Applications in the secure-ops namespace must run under a non-default ServiceAccount with restricted API token access.

Create a ServiceAccount named app-runner in the secure-ops namespace.
Create a Pod named secure-worker using the image busybox:1.36 that sleeps for 3600 seconds.
Set serviceAccountName: app-runner.
Set automountServiceAccountToken: false at the Pod level.
(Bonus) Create a Role named pod-reader that allows get and list on pods, and bind it to the ServiceAccount app-runner.

### Answer
```
kubectl create sa app-runner -n secure-ops

apiVersion: v1
kind: Pod
metadata:
  name: secure-worker
  namespace: secure-ops
spec:
  serviceAccountName: app-runner
  automountServiceAccountToken: false
  containers:
    - name: secure-worker
      image: busybox:1.36
      command:
        - /bin/sh
        - -c
        - sleep 3600

kubectl create role pod-reader --verb=get,list --resource=pods -n secure-ops
kubectl create rolebinding runner-binding \
  --role=pod-reader \
  --serviceaccount=secure-ops:app-runner \
  -n secure-ops
  ```
### Question
## 18.
You must implement process-level security at the Pod level and specific Linux capabilities at the Container level.
Create a Pod named security-layered using the image nginx with the following settings:
Pod-level securityContext:

runAsUser: 1000
runAsNonRoot: true
fsGroup: 2000

Container-level securityContext:

Add the NET_ADMIN capability
Set readOnlyRootFilesystem: true
### Answer
```
apiVersion: v1
kind: Pod
metadata:
  name: security-layered
spec:
  securityContext:
    runAsUser: 1000
    runAsNonRoot: true
    fsGroup: 2000
  containers:
    - name: security-layered
      image: nginx
      securityContext:
        capabilities:
          add:
            - NET_ADMIN
        readOnlyRootFilesystem: true
kubectl apply -f scenario4.yaml
```
### Question
## 20.
An operator manages AppVault resources. You must identify the API structure and deploy an instance.

Discover the short name and API version for AppVault.
Investigate the required fields under spec.
Create a file named vault-instance.yaml for a resource of Kind AppVault, named vault-instance, in the API group security.example.com/v1.

### Answer
```
kubectl api-resources | grep -i AppVault
kubectl explain appvault.spec
kubectl explain appvault.spec --recursive

apiVersion: security.example.com/v1
kind: AppVault
metadata:
  name: vault-instance
spec:
  # Add the required fields shown by:
  # kubectl explain appvault.spec
```
### Question
## 21.
A developer has provided a custom configuration file for an Nginx server. You must inject this file into the container without overwriting other existing configuration files in the target directory.

Namespace: web-prod
ConfigMap Name: nginx-custom-config
Key/Value: custom.conf = server_tokens off;
Pod Name: secure-web
Image: nginx:1.25-alpine
Target Path: /etc/nginx/conf.d/custom.conf

Use a volume mount with subPath so that only the single file is mounted and the rest of the directory is preserved.
### Answer
```
kubectl create ns web-prod
kubectl create cm nginx-custom-config \
  --from-literal=custom.conf='server_tokens off;' \
  -n web-prod
apiVersion: v1
kind: Pod
metadata:
  name: secure-web
  namespace: web-prod
spec:
  containers:
    - name: secure-web
      image: nginx:1.25-alpine
      volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d/custom.conf
          subPath: custom.conf
  volumes:
    - name: nginx-config
      configMap:
        name: nginx-custom-config
kubectl apply -f secure-web.yaml
```
### Question
## 22.
The dev-team namespace is governed by strict resource controls to prevent cluster exhaustion.

Create the namespace dev-team (if it does not exist).
Create a ResourceQuota named compute-quota with the following hard limits:
Pods: 4
CPU Requests: 1
CPU Limits: 2
Memory Requests: 1Gi
Memory Limits: 2Gi

Deploy a Pod named test-worker using nginx:1.25-alpine with:
Requests: 250m CPU, 256Mi Memory
Limits: 500m CPU, 512Mi Memory

### Answer
```
kubectl create ns dev-team

apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev-team
spec:
  hard:
    pods: "4"
    requests.cpu: "1"
    requests.memory: "1Gi"
    limits.cpu: "2"
    limits.memory: "2Gi"
kubectl apply -f compute-resources.yaml

kubectl run test-worker -n dev-team \
  --image=nginx:1.25-alpine \
  --requests='cpu=250m,memory=256Mi' \
  --limits='cpu=500m,memory=512Mi'
```
### Question
## 23.
Certain nodes in the cluster have been reserved for specific tiers. You need to deploy a Pod to a node that has been restricted with a taint.

Node Taint: tier=frontend:NoSchedule
Namespace: prod
Pod Name: frontend-app
Image: nginx:1.24-alpine

Configure the Pod with the necessary toleration so it can be scheduled on nodes with the above taint. The toleration must exactly match the key, value, and effect of the taint.

### Answer
```
kubectl create ns prod

apiVersion: v1
kind: Pod
metadata:
  name: frontend-app
  namespace: prod
spec:
  containers:
  - name: nginx
    image: nginx:1.24-alpine
    imagePullPolicy: IfNotPresent
  tolerations:
  - key: "tier"
    operator: "Equal"
    value: "frontend"
    effect: "NoSchedule"
kubectl apply -f pod.yaml
```
### Question
## 24.
A multi-container Pod requires a shared workspace where both containers can write files. You must ensure proper permissions for group-level access.

Namespace: apps
Pod Name: multi-app-handler
Container 1 (writer): image busybox:1.36, command touch /data/test.log && sleep 3600
Container 2 (reader): image busybox:1.36, command sleep 3600
Volume: emptyDir named data-vol mounted at /data in both containers
Set fsGroup: 2000 at the Pod level

### Answer
```
kubectl create namespace apps
apiVersion: v1
kind: Pod
metadata:
  name: multi-app-handler
  namespace: apps
spec:
  securityContext:
    fsGroup: 2000
  containers:
    - name: writer
      image: busybox:1.36
      command:
        - /bin/sh
        - -c
        - touch /data/test.log && sleep 3600
      volumeMounts:
        - name: data-vol
          mountPath: /data
    - name: reader
      image: busybox:1.36
      command:
        - sleep
        - "3600"
      volumeMounts:
        - name: data-vol
          mountPath: /data
  volumes:
    - name: data-vol
      emptyDir: {}

kubectl apply -f multi-app-handler.yaml
```
### Question
## 25.
The security team requires that images be pulled from a private registry. You must configure a ServiceAccount so that all Pods using it can automatically pull these private images.

Namespace: auth
Secret Name: private-reg-cred (type kubernetes.io/dockerconfigjson)
Use placeholder credentials:
server: https://index.docker.io/v1/
username: admin
password: secret123
email: admin@example.com

ServiceAccount Name: internal-developer (link the imagePullSecrets to the secret)
Pod Name: private-app using image nginx:1.25-alpine and the ServiceAccount internal-developer
### Answer
```
kubectl create namespace auth

kubectl create secret docker-registry private-reg-cred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=admin \
  --docker-password=secret123 \
  --docker-email=admin@example.com \
  -n auth

kubectl create serviceaccount internal-developer -n auth

apiVersion: v1
kind: ServiceAccount
metadata:
  name: internal-developer
  namespace: auth
imagePullSecrets:
  - name: private-reg-cred

apiVersion: v1
kind: Pod
metadata:
  name: private-app
  namespace: auth
spec:
  serviceAccountName: internal-developer
  containers:
    - name: private-app
      image: nginx:1.25-alpine
```

### Question
## 26.
Demonstrate your understanding of how configuration changes propagate to running Pods. Compare Environment Variables (static) against Volume Mounts (dynamic).

Namespace: config-test
ConfigMap: app-settings with keys COLOR=blue and MODE=fast
Pod: updater-pod using busybox:1.36 (sleep 3600)
Map the key COLOR from the ConfigMap to an environment variable named APP_COLOR
Mount the entire ConfigMap as a volume at /etc/config

After the Pod is running, update the ConfigMap (COLOR=red, MODE=slow) and observe the difference between the environment variable and the mounted files.

### Answer
```
kubectl create ns config-test

kubectl create cm app-settings \
  --from-literal=COLOR=blue \
  --from-literal=MODE=fast \
  -n config-test
apiVersion: v1
kind: Pod
metadata:
  name: updater-pod
  namespace: config-test
spec:
  containers:
    - name: test-container
      image: busybox:1.36
      command: ["/bin/sh", "-c", "sleep 3600"]
      env:
        - name: APP_COLOR
          valueFrom:
            configMapKeyRef:
              name: app-settings
              key: COLOR
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: app-settings

kubectl edit cm app-settings -n config-test
```
### Question
## 27.
Applications in the secure-ops namespace must run under a non-default ServiceAccount with restricted API token access.
Configuration Instructions
ServiceAccount: Create a ServiceAccount named app-runner in the secure-ops namespace.
Speed String: kubectl create sa app-runner -n secure-ops
Token Hardening: Create a Pod named secure-worker using busybox:1.36.
Speed String: kubectl run secure-worker --image=busybox:1.36 -n secure-ops $do -- /bin/sh -c "sleep 3600" > scenario3.yaml
Logic: In the manifest, set serviceAccountName: app-runner. Crucially, set automountServiceAccountToken: false at the Pod level to prevent the default token mount.
Bonus (RBAC): Create a Role named pod-reader that allows get and list on pods. Bind it to app-runner.
Speed String: kubectl create role pod-reader --verb=get,list --resource=pods -n secure-ops
Speed String: kubectl create rolebinding runner-binding --role=pod-reader --serviceaccount=secure-ops:app-runner -n secure-ops

### Answer
```
kubectl create sa app-runner -n secure-ops

kubectl get sa app-runner -n secure-ops

export do='--dry-run=client -o yaml'
kubectl run secure-worker --image=busybox:1.36 -n secure-ops $do -- /bin/sh -c "sleep 3600" > scenario3.yaml

vi scenario3.yaml

apiVersion: v1
kind: Pod
metadata:
  name: secure-worker
  namespace: secure-ops
spec:
  serviceAccountName: app-runner
  automountServiceAccountToken: false
  containers:
    - name: secure-worker
      image: busybox:1.36
      command:
        - /bin/sh
        - -c
        - sleep 3600

kubectl create role pod-reader --verb=get,list --resource=pods -n secure-ops
kubectl get role pod-reader -n secure-ops
kubectl create rolebinding runner-binding --role=pod-reader --serviceaccount=secure-ops:app-runner -n secure-ops
```
### Question
## 28.
You must implement process-level security at the Pod level and specific Linux capabilities at the Container level.

Configuration Instructions

Initial Manifest: Generate a Pod named security-layered.
Speed String: kubectl run security-layered --image=nginx $do > scenario4.yaml
Pod-Level Settings: Apply a securityContext to the Pod spec:
runAsUser: 1000
runAsNonRoot: true
fsGroup: 2000
Container-Level Settings: Apply a securityContext to the container spec:
Add the NET_ADMIN capability.
Set readOnlyRootFilesystem: true.

### Answer
```
export do='--dry-run=client -o yaml'
kubectl run security-layered --image=nginx $do > scenario4.yaml

vi scenario4.yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-layered
spec:
  securityContext:
    runAsUser: 1000
    runAsNonRoot: true
    fsGroup: 2000

  containers:
    - name: security-layered
      image: nginx
      securityContext:
        capabilities:
          add:
            - NET_ADMIN
        readOnlyRootFilesystem: true
kubectl apply -f scenario4.yaml
```
### Question
## 29.
Isolate a frontend application to production-designated nodes using strict scheduling rules.
Configuration Instructions
Initial Manifest:
Speed String: kubectl run frontend-prod --image=nginx $do > scenario5.yaml
Strict Isolation: Add a nodeAffinity block. Use the requiredDuringSchedulingIgnoredDuringExecution type.
Labels: The Pod must only land on nodes with both environment=production and tier=frontend.
Verification Criteria
kubectl describe pod frontend-prod
Verify that Node-Selectors is empty but the Affinity field contains the required expressions. 
### Answer
```
kubectl run frontend-prod --image=nginx --dry-run=client -o yaml > scenario5.yaml

vi scenario5.yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend-prod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: environment
            operator: In
            values:
            - production
          - key: tier
            operator: In
            values:
            - frontend
  containers:
  - name: frontend-prod
    image: nginx

kubectl apply -f scenario5.yaml

An operator manages AppVault resources. You must identify the API structure and deploy an instance.
```
### Question
## 30.
Context & Requirements A developer has provided a custom configuration file for an Nginx server. You must ensure this file is injected into the container without overwriting other existing configuration files in the target directory.
Namespace: web-prod
ConfigMap Name: nginx-custom-config
Key/Value in ConfigMap: custom.conf with the value server_tokens off;
Pod Name: secure-web
Image: nginx:1.25-alpine
Target Path: /etc/nginx/conf.d/custom.conf
Configuration Instructions
Create the namespace web-prod.
Create the ConfigMap nginx-custom-config with the specified key and value.
Deploy the Pod secure-web.
Configure a volume using the ConfigMap.
Mount only the specific key custom.conf to the file path /etc/nginx/conf.d/custom.conf using a subPath to prevent the directory /etc/nginx/conf.d/ from being cleared of other files.

### Answer
```
k create ns web-prod
k create cm -n web-prod --from-literal=custom.conf='server_tokens off;'
apiVersion: v1
kind: Pod
metadata:
  name: secure-web
  namespace: web-prod
spec:
  containers:
    - name: secure-web
      image: nginx:1.25-alpine
      volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d/custom.conf
          subPath: custom.conf

  volumes:
    - name: nginx-config
      configMap:
        name: nginx-custom-config

kubectl apply -f secure-web.yaml
```
### Question
## 31.
The dev-team namespace is governed by strict resource controls to prevent cluster exhaustion. You must deploy a workload that fits within the defined hard limits.
Namespace: dev-team (Create if missing)
ResourceQuota Name: compute-quota
Hard Limits:
Maximum Pods: 4
Total CPU Requests: 1
Total CPU Limits: 2
Total Memory Requests: 1Gi
Total Memory Limits: 2Gi
Configuration Instructions
Create the namespace and the ResourceQuota as defined above.
Deploy a Pod named test-worker using the image nginx:1.25-alpine.
Specify compute resources for the Pod that satisfy the quota and allow for additional Pods to be created later:
Requests: 250m CPU, 256Mi Memory.
Limits: 500m CPU, 512Mi Memory.
### Answer
```
k create ns dev-team
cat <<EOF > compute-resources.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namesapce: dev-team
spec:
  hard:
    requests.cpu: "1"
    requests.memory: "1Gi"
    limits.cpu: "2"
    limits.memory: "2Gi"
    pods: "4"
EOF

k run test-worker -n dev-team --image=nginx:1.25-alpine --requests='cpu=250m,memory=256Mi' --limits='cpu=500m,memory=512Mi'
```

### Question
## 31.
Certain nodes in the cluster have been reserved for specific tiers. You need to deploy a Pod to a node that has been restricted with a taint.
Node Taint: tier=frontend:NoSchedule
Namespace: prod
Pod Name: frontend-app
Image: nginx:1.24-alpine
Configuration Instructions
Create the namespace prod.
Configure the Pod frontend-app with the necessary toleration to allow it to be scheduled on nodes with the tier=frontend:NoSchedule taint.
Ensure the toleration exactly matches the key, value, and effect of the taint.
### Answer
```
k create ns prod
k run frontend-app --image=nginx:1.24-alpine -n prod --dry-run=client -o yaml > pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: frontend-app
  namespace: prod
spec:
  containers:
  - name: nginx
    image: nginx:1.24-alpine
    imagePullPolicy: IfNotPresent
  tolerations:
  - key: "tier"
    operator: "Equal"
    value: "frontend"
    effect: "NoSchedule"
```
### Question
## 32.
 Shared Volume & Group Ownership (fsGroup)
Context & Requirements A multi-container Pod requires a shared workspace where both containers can write files. You must ensure proper permissions for group-level access.
Namespace: apps
Pod Name: multi-app-handler
Container 1: Name writer, image busybox:1.36, command /bin/sh -c "touch /data/test.log && sleep 3600"
Container 2: Name reader, image busybox:1.36, command sleep 3600
Volume: emptyDir named data-vol mounted at /data in both containers.
Security Context: Set the fsGroup at the Pod level to 2000.
Configuration Instructions
Create the namespace apps.
Define the Pod with the two containers and the shared emptyDir volume.
Apply a Pod-level securityContext setting the fsGroup to 2000.
Mount the volume at /data in both containers.
### Answer
```
kubectl create namespace apps
vi multi-app-handler.yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-app-handler
  namespace: apps
spec:
  securityContext:
    fsGroup: 2000

  containers:
    - name: writer
      image: busybox:1.36
      command:
        - /bin/sh
        - -c
        - touch /data/test.log && sleep 3600
      volumeMounts:
        - name: data-vol
          mountPath: /data

    - name: reader
      image: busybox:1.36
      command:
        - sleep
        - "3600"
      volumeMounts:
        - name: data-vol
          mountPath: /data

  volumes:
    - name: data-vol
      emptyDir: {}

kubectl apply -f multi-app-handler.yaml
```
### Question
## 33.
The security team requires that images be pulled from a private registry. You must configure a ServiceAccount so that all Pods using it can automatically pull these private images.
Namespace: auth
Secret Name: private-reg-cred
Secret Type: kubernetes.io/dockerconfigjson
ServiceAccount Name: internal-developer
Pod Name: private-app
Image: nginx:1.25-alpine (Simulating a private image pull)
Configuration Instructions
Create the namespace auth.
Create a docker-registry secret named private-reg-cred using placeholder credentials (user: admin, pass: secret123, email: admin@example.com, server: https://index.docker.io/v1/).
Create the ServiceAccount internal-developer and link the imagePullSecrets to the private-reg-cred secret.
Deploy the Pod private-app in the auth namespace.
Assign the internal-developer ServiceAccount to the Pod.

### Answer
```
kubectl create namespace auth
kubectl create secret docker-registry private-reg-cred --docker-server=https://index.docker.io/v1/ --docker-username=admin --docker-password=secret123 --docker-email=admin@example.com -n auth
kubectl create serviceaccount internal-developer -n auth
kubectl edit serviceaccount internal-developer -n auth

apiVersion: v1
kind: ServiceAccount
metadata:
  name: internal-developer
  namespace: auth
imagePullSecrets:
  - name: private-reg-cred


kubectl run private-app --image=nginx:1.25-alpine -n auth --dry-run=client -o yaml > private-app.yaml

vi private-app.yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-app
  namespace: auth
spec:
  serviceAccountName: internal-developer
  containers:
    - name: private-app
      image: nginx:1.25-alpine
```
### Question
## 34.
A microservice must run as a non-privileged user (UID 1000) for security hardening. However, the application requires a configuration file to be pre-populated in a shared directory that is initially owned by root. You must use an initContainer to bridge this gap.
Namespace: security-labs
Pod Name: data-prepper
Shared Volume: An emptyDir volume named app-data mounted at /var/lib/app in both containers.
InitContainer Configuration:
Name: setup-init
Image: busybox:1.36
Task: Create a file at /var/lib/app/config.txt containing the text "Initialised" and change the ownership of the entire /var/lib/app directory to user 1000.
Main Container Configuration:
Name: main-app
Image: nginx:1.25-alpine
SecurityContext: Must run as user 1000 (runAsUser: 1000).
Instructions
Create the security-labs namespace.
Define a Pod that implements the initContainer to perform the file creation and ownership change.
Ensure the main container runs as the non-root user and can read the file created by the initContainer.

### Answer
```
k create ns security-labs
k run data-prepper --image=nginx:1.25-alpine -n security-labs --dry-run=client -o yaml > data-prepper.yaml
vi data-prepper.yaml

apiVersion: v1
kind: Pod
metadata:
  name: data-prepper
  namespace: security-labs 

spec:
  securityContext:
    runAsUser: 1000
  containers:
  - name: main-app
    image: nginx:1.25-alpine
    securityContext:
      runAsUser: 1000 
    volumeMounts:
    - name: app-data
      mountPath: /var/lib/app
  initContainers:
  - name: setup-init
    image: busybox:1.36
    command:
    - /bin/sh
    - -c
    - echo "Initialised" > /var/lib/app/config.txt && chown -R 1000 /var/lib/app
    volumeMounts:
    - name: app-data
      mountPath: /var/lib/app
    securityContext:
      runAsUser: 0
  dnsPolicy: Default
  volumes:
  - name: app-data
    emptyDir: {}
```

### Question
## 35.
Strict data protection policies require that sensitive credentials mounted as volumes have the most restrictive permissions possible to prevent unauthorised access by other processes within the container.
Namespace: auth-system
Secret Name: ssh-key-secret
Secret Data: A key named id_rsa with any dummy string value.
Pod Name: secure-ssh-client
Mount Path: Mount the secret at /etc/ssh-keys.
Permission Requirement: Use defaultMode to set the file permissions to 0400 (octal equivalent 256) to ensure the file is read-only for the owner and inaccessible to others.
Instructions
Create the namespace and the generic secret with the specified key.
Configure the Pod to mount the secret as a volume.
Explicitly set the defaultMode in the volume definition.
Verification Criteria
Check the Pod logs or use ls -l /etc/ssh-keys inside the container.
Confirm the permissions for id_rsa are exactly -r--------.

### Answer
```
kubectl create namespace auth-system
kubectl create secret generic ssh-key-secret \
  --from-literal=id_rsa='dummy-private-key-value' \
  -n auth-system


apiVersion: v1
kind: Pod
metadata:
  name: secure-ssh-client
  namespace: auth-system

spec:
  containers:
    - name: ssh-client
      image: busybox:1.36
      command:
        - /bin/sh
        - -c
        - sleep 3600

      volumeMounts:
        - name: ssh-keys
          mountPath: /etc/ssh-keys
          readOnly: true

  volumes:
    - name: ssh-keys
      secret:
        secretName: ssh-key-secret
        defaultMode: 0400
```

### Question
## 36.
To enforce fine-grained security, you must allow a ServiceAccount in a development namespace to access specific resources in a shared resource namespace without granting broad cluster-wide permissions.
Namespace A: shared-resources
Namespace B: app-dev
Resource: A Role named config-reader in shared-resources that allows get, list, and watch on ConfigMaps.
Subject: A ServiceAccount named developer-sa in the app-dev namespace.
Binding: A RoleBinding named cross-ns-binding in the shared-resources namespace.
Instructions
Create both namespaces.
Create the ServiceAccount in app-dev.
Create the Role in shared-resources.
Create the RoleBinding in shared-resources to link the Role to the ServiceAccount located in app-dev.

### Answer
```
k create ns shared-resources
k create ns app-dev
k create role config-reader -n shared-resources --resource=configmaps --verb=get --verb=list --verb=watch
k create sa developer-sa  -n app-dev
k create rolebinding cross-ns-binding --role config-reader --serviceaccount=app-dev:developer-sa -n shared-resources

A critical frontend deployment in the production environment must remain available during voluntary disruptions (such as node maintenance or cluster upgrades).
Namespace: prod-apps
Target Deployment: A deployment named web-server with 3 replicas using nginx:1.24-alpine.
PDB Name: web-pdb
Requirement: Enforce that at least 2 replicas are always available during voluntary disruptions.
Selector: Ensure the PDB targets the pods of the web-server deployment via labels (app: web-server).

apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
  namespace: prod-apps
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-server
  template:
    metadata:
      labels:
        app: web-server
    spec:
      containers:
        - name: web-server
          image: nginx:1.24-alpine

apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
  namespace: prod-apps
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web-server

```
### Question
## 37.
To ensure high availability and physical node distribution, you must configure a workload to avoid concentrating all replicas on a single host.
Namespace: infrastructure
Deployment Name: distributed-app
Replicas: 4
Topology Spread Requirements:
maxSkew: 1
topologyKey: kubernetes.io/hostname
whenUnsatisfiable: DoNotSchedule
labelSelector: Match pods with app: distributed-app.
Instructions
Create the infrastructure namespace.
Create the deployment and include the topologySpreadConstraints in the Pod template spec.
Ensure the labelSelector in the constraint matches the Pod labels.

### Answer
```
apiVersion: v1
kind: Namespace
metadata:
  name: infrastructure
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: distributed-app
  namespace: infrastructure
spec:
  replicas: 4

  selector:
    matchLabels:
      app: distributed-app

  template:
    metadata:
      labels:
        app: distributed-app

    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: distributed-app

      containers:
        - name: distributed-app
          image: nginx:1.24-alpine
```
### Question
## 40.
You are required to manage an existing Helm release. You must update the configuration to meet new security and scaling standards using imperative CLI overrides.
Release Name: vault-backend
Chart: Assume a local chart or a common repository chart like bitnami/nginx.
Namespace: finance
Override Requirements:
Replica Count: Increase to 3.
Image Tag: Update to 1.25.3.
Security Context: Set containerSecurityContext.runAsUser to 1001.
Resource Limits: Set memory limit to 256Mi.

### Answer
```
helm upgrade vault-backend bitnami/nginx \
  -n finance \
  --set replicaCount=3 \
  --set image.tag=1.25.3 \
  --set containerSecurityContext.runAsUser=1001 \
  --set resources.limits.memory=256Mi
```
### Question
## 41.
A legacy application requires a sidecar container to stream its file-based logs to the standard output for collection by a cluster-wide logging agent.
Requirements:
Namespace: logging-infra
Pod Name: web-logger
Primary Container:
Name: app-container
Image: busybox:1.36
Command: /bin/sh, -c, "while true; do echo $(date) >> /var/log/app.log; sleep 5; done"
Sidecar Container:
Name: sidecar-log-streamer
Image: busybox:1.36
Command: /bin/sh, -c, "tail -f /var/log/app.log"
Storage: Implement a shared emptyDir volume mounted at /var/log in both containers.

### Answer
```
k create ns logging-infra
k run web-logger --image=busybox:1.36 -n logging-infra --dry-run=client -o yaml > web.logger.yaml
vi web-logger.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-logger
  namespace: logging-infra
spec:
  containers:
  - name: app-container
    image: busybox:1.36
    command: ["sh", "-c", "while true; do echo $(date) >> /var/log/app.log; sleep 5; done"]
    volumeMounts:
      - name: app-log
        mountPath: /var/log
  - name: sidecar-log-streamer
    image: busybox:1.36
    command: ["/bin/sh", "-c", "tail -f /var/log/app.log"]
    volumeMounts:
      - name: app-log
        mountPath: /var/log
  volumes:
  - name: app-log
    emptyDir: {}
```
### Question
## 42.
A microservice needs to bind to a privileged port (TCP 80) but must not run as the root user. You must harden the container by stripping all default Linux capabilities and adding only the minimum required.
Requirements:
Namespace: security-hardened
Pod Name: restricted-service
Container Image: nginx:1.25-alpine
Security Configuration:
Drop ALL Linux capabilities.
Add only the NET_BIND_SERVICE capability.
Configure the container to run as User ID 1001.
Ensure the container cannot gain additional privileges (allowPrivilegeEscalation: false).

### Answer
```
apiVersion: v1
kind: Namespace
metadata:
  name: security-hardened
---
apiVersion: v1
kind: Pod
metadata:
  name: restricted-service
  namespace: security-hardened
spec:
  containers:
    - name: restricted-service
      image: nginx:1.25-alpine
      securityContext:
        runAsUser: 1001
        allowPrivilegeEscalation: false
        capabilities:
          drop:
            - ALL
          add:
            - NET_BIND_SERVICE
```
### Question
## 43.
To prevent unauthorized persistent changes to a container's filesystem, you are tasked with deploying a workload with a completely read-only root filesystem. The application still requires temporary write access to specific directories to function.
Requirements:
Namespace: immutable-apps
Pod Name: stateless-worker
Container Image: busybox:1.36
Command: sleep 3600
Security Configuration: Set readOnlyRootFilesystem: true.
Volume Configuration:
Mount an emptyDir volume at /tmp.
Mount a separate emptyDir volume at /var/cache.

### Answer
```
apiVersion: v1
kind: Namespace
metadata:
  name: immutable-apps
---
apiVersion: v1
kind: Pod
metadata:
  name: stateless-worker
  namespace: immutable-apps
spec:
  containers:
    - name: stateless-worker
      image: busybox:1.36
      command:
        - sleep
        - "3600"
      securityContext:
        readOnlyRootFilesystem: true
      volumeMounts:
        - name: tmp-volume
          mountPath: /tmp
        - name: cache-volume
          mountPath: /var/cache

  volumes:
    - name: tmp-volume
      emptyDir: {}
    - name: cache-volume
      emptyDir: {}
```
### Question
## 44.
A backend application needs to consume data from multiple sources (Secrets, ConfigMaps, and ServiceAccount tokens) at a single filesystem path.
Requirements:
Namespace: unified-config
Existing Resources: (Create these first)
ConfigMap app-settings with key config.json.
Secret api-credentials with key api-token.
Pod Name: config-aggregator
Volume Mount: A single projected volume mounted at /etc/pod-config.
Sources to Project:
The ConfigMap app-settings.
The Secret api-credentials.
A service account token with an expiration of 3600 seconds and an audience of api-service.

### Answer
```
apiVersion: v1
kind: Pod
metadata:
  name: config-aggregator
  namespace: unified-config
spec:
  containers:
    - name: config-aggregator
      image: busybox:1.36
      command:
        - sleep
        - "3600"
      volumeMounts:
        - name: pod-config
          mountPath: /etc/pod-config

  volumes:
    - name: pod-config
      projected:
        sources:
          - configMap:
              name: app-settings

          - secret:
              name: api-credentials

          - serviceAccountToken:
              path: token
              expirationSeconds: 3600
              audience: api-service
```
### Question
## 45.
 The cluster administrator requires critical workloads to be protected from eviction during resource contention. You must define a PriorityClass and two Pods with different QoS tiers.
Requirements:
Namespace: resource-management
PriorityClass:
Name: critical-workload
Value: 1000000
Global Default: false
Pod 1 (Guaranteed QoS):
Name: guaranteed-pod
Image: nginx:1.25-alpine
Resources: CPU request and limit both set to 200m; Memory request and limit both set to 256Mi.
Priority: Use critical-workload.
Pod 2 (Burstable QoS):
Name: burstable-pod
Image: nginx:1.25-alpine
Resources: CPU request 100m, CPU limit 200m; Memory request 128Mi, Memory limit 256Mi.

### Answer
```
apiVersion: v1
kind: Namespace
metadata:
  name: resource-management
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-workload
value: 1000000
globalDefault: false
description: "Priority class for critical workloads"
---
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
  namespace: resource-management
spec:
  priorityClassName: critical-workload
  containers:
    - name: guaranteed-pod
      image: nginx:1.25-alpine
      resources:
        requests:
          cpu: 200m
          memory: 256Mi
        limits:
          cpu: 200m
          memory: 256Mi
---
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod
  namespace: resource-management
spec:
  containers:
    - name: burstable-pod
      image: nginx:1.25-alpine
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 200m
          memory: 256Mi
```
### Question
## 46.
Identify Node: Find the first worker node in your cluster (e.g., node01 or worker node).
Apply Taint: Taint this node with key workload, value maintenance, and effect NoSchedule.
Verify Node Isolation: Attempt to run a temporary pod named test-normal with image nginx without any tolerations. Confirm it lands on a different node or remains Pending if no other node is available. Then delete test-normal.
Deploy Tolerant Pod:
Create a pod named maintenance-agent using image nginx:1.25-alpine.
Configure the pod with a toleration matching the taint (key: workload, operator: Equal, value: maintenance, effect: NoSchedule).
Add a nodeSelector or nodeName explicitly pointing to that tainted node to ensure it schedules there.
Verification: Confirm maintenance-agent schedules and reaches Running status on the tainted node.

### Answer
```
k get nodes
k taint nodes node01 workload=maintenance:NoSchedule
k run test-normal --image=nginx
k run maintenance-agent --image=nginx:1.25-alpine --dry-run=client -o yaml > agent.yaml
vi agent.yaml

apiVersion: v1
kind: Pod
metadata:
  name: maintenance-agent
spec:
  containers:
  - name: nginx
    image: nginx:1.25-alpine
  tolerations:
  - key: "workload"
    operator: "Equal"
    value: "maintenance"
    effect: "NoSchedule"
```
### Question
## 47.
A sensitive backend database running in the database namespace needs strict traffic isolation. It must only accept incoming TCP traffic from authorized frontend clients.
Context & Requirements
Target Workload:
Namespace: database
Pod selector label: app=secure-db
Serving port: 5432 (TCP)
NetworkPolicy Specification:
Policy Name: allow-frontend-traffic
Target Namespace: database
Rule 1 (Cross-Namespace AND Pod Match): Allow incoming traffic on port 5432 only from Pods labeled role=frontend that reside inside Namespaces labeled env=production.
Rule 2 (Same-Namespace Match): Allow incoming traffic on port 5432 from any Pod located inside the same namespace (database) labeled role=monitoring.
Implicit Deny: Ensure all other inbound traffic to pods labeled app=secure-db is blocked.

### Answer
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-traffic
  namespace: database
spec:
  podSelector:
    matchLabels:
      app: secure-db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          env: production
      podSelector:
        matchLabels:
          role: frontend
    - podSelector:
        matchLabels:
          role: monitoring
    ports:
    - protocol: TCP
      port: 5432
```
### Question
## 48.
A payment processing worker deployed in the finance namespace must be locked down so it cannot access internal cluster networks. It is only permitted to communicate with an external banking gateway API and resolve cluster DNS.
Namespace: finance
Target Pods: labeled app=payment-processor
NetworkPolicy Specification:
Policy Name: lockdown-egress
Namespace: finance
Rule 1 (External Gateway): Allow outbound traffic on port 443 (TCP) strictly to the external CIDR range 198.51.100.0/24, but explicitly except the IP address 198.51.100.15/32.
Rule 2 (DNS Resolution): Allow outbound traffic to UDP port 53 and TCP port 53 to any destination (no peer restrictions) so internal DNS lookups continue to work.
Implicit Deny: Ensure all other outbound (egress) traffic from these pods is blocked. Ingress traffic is out of scope and should remain unaffected.

### Answer
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: lockdown-egress
  namespace: finance
spec:
  podSelector:
    matchLabels:
      app: payment-processor
  policyTypes:
  - Egress
  egress:
  # Rule 1: External gateway CIDR range (with exception) on port 443
  - to:
    - ipBlock:
        cidr: 198.51.100.0/24
        except:
        - 198.51.100.15/32
    ports:
    - protocol: TCP
      port: 443

  # Rule 2: DNS resolution (UDP + TCP 53) to any destination
  - ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

### Question
## 49.

Namespace: Create the namespace security-team if it does not exist.
Pod Name: hardened-worker
Namespace: security-team
Image: busybox:1.36
Command: sleep 3600
Pod-Level SecurityContext:
Ensure that all containers in the pod run with user ID 1000.
Set the pod-level filesystem group (fsGroup) to 2000.
Container-Level SecurityContext:
For the single container named worker:
Disallow privilege escalation (allowPrivilegeEscalation: false).
Ensure the container runs as a non-root user (runAsNonRoot: true).
Add the Linux capability NET_ADMIN.
Drop the Linux capability ALL.
k create ns security-team

### Answer
```
apiVersion: v1
kind: Pod
metadata:
  name: hardened-worker
  namespace: security-team
spec:
  securityContext:
    runAsUser: 1000
    fsGroup: 2000
    supplementalGroups: [4000]
  containers:
  - name: worker
    image: busybox:1.36
    command: [ "sh", "-c", "sleep 3600" ]
    securityContext:
      allowPrivilegeEscalation: false
      runAsNonRoot: true
      capabilities:
        add: ["NET_ADMIN"]
        drop: ["ALL"]
```
### Question
## 49.
Namespace: Create the namespace observability if it does not exist.
Shared Storage: Configure an emptyDir volume named log-volume.
Pod Specification:
Pod Name: transaction-service
Namespace: observability
Primary Container:
Container Name: app-producer
Image: busybox:1.36
Volume Mount: Mount log-volume at /var/log/app.
Command: Run a continuous loop that appends the current timestamp and a message to /var/log/app/transactions.log every 5 seconds:

### Answer
```
k create ns observability

apiVersion: v1
kind: Pod
metadata:
  name: transaction-service
  namespace: observability
spec:
  containers:
    - name: app-producer
      image: busybox:1.36
      command: ['sh', '-c', 'while true; do echo "$(date) - New transaction processed" >> /var/log/app/transactions.log; sleep 5; done']
      volumeMounts:
        - name: log-volume
          mountPath: /var/log/app
    - name: log-shipper
      image: busybox:1.36
      command: ['sh', '-c', 'tail -n+1 -f /var/log/app/transactions.log']
      volumeMounts:
        - name: log-volume
          mountPath: /var/log/app
          readOnly: true
  volumes:
    - name: log-volume
       emptyDir: {}
```
