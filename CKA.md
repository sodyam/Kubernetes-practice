A Kubernetes cluster was recently migrated from an external etcd setup to a local (stacked) etcd configuration. Following the migration, the control plane is unhealthy and the API server is unavailable.

Investigation reveals that multiple control-plane components may be incorrectly configured.

Perform the necessary troubleshooting steps to restore the cluster to a healthy state.

Requirements
Ensure the kube-apiserver can communicate with the local etcd instance.
Ensure the kube-controller-manager and kube-scheduler can start successfully on the available node resources.
Verify that the control plane and cluster become healthy.


Step 1: Check the kube-apiserver etcd configuration

Inspect the API server static Pod manifest:

sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml

Locate the --etcd-servers argument.

If it still points to an old external etcd endpoint, update it to the local etcd endpoint:

- --etcd-servers=https://127.0.0.1:2379

Save the file. Since this is a static Pod, the kubelet automatically detects the manifest change and recreates the API server.

Step 2: Check the kube-controller-manager resource requests

Inspect:

sudo vi /etc/kubernetes/manifests/kube-controller-manager.yaml

Check whether the CPU request is unnecessarily high.

For example, change an excessive value such as:

resources:
  requests:
    cpu: "2"

to:

resources:
  requests:
    cpu: 200m

A very high CPU request can prevent the static Pod from being scheduled or started successfully on a small control-plane node.

Step 3: Check the kube-scheduler resource requests

Inspect:

sudo vi /etc/kubernetes/manifests/kube-scheduler.yaml

Similarly, reduce an excessively high CPU request:

resources:
  requests:
    cpu: 200m

Save the manifest.

The kubelet should automatically recreate the affected static Pods.

Step 4: Wait for the control-plane components to recover

Give the kubelet a few moments to detect the manifest changes and restart the static Pods.

If necessary, inspect the kubelet:

sudo systemctl status kubelet

You can also restart it if required:

sudo systemctl restart kubelet
Step 5: Verify the cluster

Once the API server is available again, check the nodes:

kubectl get nodes

Check the control-plane components:

kubectl get pods -n kube-system
