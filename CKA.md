Yes — this is a well-known (and tricky) CKA troubleshooting scenario involving a broken control plane after a migration from external etcd.
What the question almost certainly was
Something along the lines of:
The cluster was recently migrated from an external etcd. The API server is down / the control plane is not functional. Fix it so that the cluster becomes healthy again. (Sometimes it also mentions that kube-system pods aren’t coming up or that CPU usage should stay under ~10% of the node’s capacity.)
The three problems that are usually present together
Wrong etcd endpoint in the API server static pod
In /etc/kubernetes/manifests/kube-apiserver.yaml the --etcd-servers flag is still pointing at the old external etcd address (or a wrong IP/port).
Fix: Change it to the local (stacked) etcd:
--etcd-servers=https://127.0.0.1:2379
(This matches what the commenter “Xuupu” said.)
kube-controller-manager CPU request is ridiculously high
In its static pod manifest the resources.requests.cpu is set to something like "2" or "4". On the small exam nodes this prevents the static pod from ever starting.
kube-scheduler has the same problem
Same inflated CPU request, so it also fails to start.
Why the original poster got stuck
The API server manifest looked fine (port 2379 was already correct in their case, or they only checked the port).
They only lowered the CPU requests on controller-manager and scheduler → nothing changed because the API server still couldn’t talk to etcd.
journalctl and crictl showed little useful output because the static pods were either not starting or crashing immediately.
Correct order of fixes
Fix the --etcd-servers line in kube-apiserver.yaml → API server comes back.
Lower the CPU requests (usually to 200m) in both kube-controller-manager.yaml and kube-scheduler.yaml.
Wait a few seconds (or restart kubelet) and verify with kubectl get pods -n kube-system and kubectl get nodes.
This exact multi-issue “migration from external etcd + oversized CPU requests” pattern has been reported by many candidates in 2025–2026 and matches both the original post and the helpful reply you screenshotted.
