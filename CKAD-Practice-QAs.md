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
  

