# Manual Deployment: Longhorn Distributed Block Storage

This guide provides instructions for deploying **Longhorn**, a lightweight, highly available distributed block storage system for Kubernetes, on your local K3s homelab cluster.

> [!NOTE]
> Longhorn is an **optional infrastructure addon** for the homelab. By default, K3s provisions storage using `local-path-provisioner`, which pins data to a single node. Longhorn enables true cross-node replication, snapshotting, and dynamic failover for stateful workloads like GitLab and KubeVirt.

---

## Why Distributed Storage for Homelabs?

| Feature | K3s Default (`local-path`) | Longhorn (`longhorn`) |
| :--- | :--- | :--- |
| **Replication** | Single node (no replication) | Multi-node synchronous block replication |
| **Pod Failover** | Pod must run on the original node | Pod can reschedule to any worker node seamlessly |
| **Snapshots & Backups** | Manual filesystem copies | Built-in volume snapshots, backup to NFS/S3 |
| **Management UI** | None (CLI only) | Rich web UI for volume, node, and disk status |

---

## Prerequisites

1. **Active K3s Cluster**: With MetalLB configured.
2. **OS Packages**: Ensure `open-iscsi` and `cryptsetup` are installed on all worker nodes:
   ```bash
   ansible -i inventory/inventory.yaml worker_nodes -m apt -a "name=open-iscsi,cryptsetup state=present" -b
   ```
3. **`iscsid` Service Active**:
   ```bash
   ansible -i inventory/inventory.yaml worker_nodes -m systemd -a "name=iscsid state=started enabled=yes" -b
   ```

---

## Option A: Automated Deployment via Ansible Playbook

Run the provided optional deployment playbook:

```bash
ansible-playbook -i inventory/inventory.yaml playbooks/deploy-longhorn.yaml
```

This installs Longhorn using Helm, waits for the driver rollout, and exposes the Longhorn UI via a MetalLB LoadBalancer IP.

---

## Option B: Manual Deployment via Helm

### 1. Add and Update Helm Repository
```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
```

### 2. Install Longhorn
For homelab environments with 4 worker nodes, a 2-replica setup is optimal to balance high availability with disk usage:

```bash
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --set defaultSettings.defaultReplicaCount=2 \
  --set persistence.defaultClassReplicaCount=2
```

### 3. Verify Rollout
```bash
kubectl get pods -n longhorn-system -w
```

Wait until all Longhorn manager, driver, and engine pods are `Running`.

### 4. Expose the Longhorn Management UI
Expose the web dashboard via MetalLB:

```bash
kubectl patch svc longhorn-frontend -n longhorn-system -p '{"spec":{"type":"LoadBalancer"}}'
kubectl get svc longhorn-frontend -n longhorn-system
```

Access the IP in your browser at `http://<LONGHORN_LOADBALANCER_IP>`.

---

## Verifying Storage Functionality

Create a test PVC and Pod to confirm distributed volume binding:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: longhorn-test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: storage-test-pod
spec:
  containers:
  - name: test
    image: busybox
    command: ["sh", "-c", "echo 'Hello from Longhorn' > /data/test.txt && sleep 3600"]
    volumeMounts:
    - mountPath: /data
      name: test-vol
  volumes:
  - name: test-vol
    persistentVolumeClaim:
      claimName: longhorn-test-pvc
```

Apply and verify:
```bash
kubectl apply -f test-storage.yaml
kubectl get pod storage-test-pod
kubectl exec storage-test-pod -- cat /data/test.txt
```

Clean up the test pod:
```bash
kubectl delete pod storage-test-pod
kubectl delete pvc longhorn-test-pvc
```

---

## Cleanup / Uninstallation

To remove Longhorn completely from the cluster:

```bash
helm uninstall longhorn -n longhorn-system
kubectl delete namespace longhorn-system
```
