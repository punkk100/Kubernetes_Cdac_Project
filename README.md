# TrueNAS Storage Role

This role configures Kubernetes to use TrueNAS as an external storage backend via NFS or iSCSI.

## Description

TrueNAS provides enterprise-grade network-attached storage (NAS) with ZFS filesystem benefits. This role integrates TrueNAS with Kubernetes for dynamic persistent volume provisioning.

## Features

- **NFS Storage**: Dynamic provisioning via nfs-subdir-external-provisioner
- **ReadWriteMany Support**: Multiple pods can access the same volume simultaneously
- **Centralized Storage**: No local storage requirements on Kubernetes nodes
- **Enterprise Features**: Leverage TrueNAS snapshots, replication, and compression
- **Easy Management**: Single TrueNAS UI for all storage operations
- **Flexible Configuration**: Customizable mount options and reclaim policies

## Prerequisites

### TrueNAS Server Setup

1. **Create NFS Dataset:**
   ```bash
   # On TrueNAS, create a dataset
   # Example: /mnt/tank/k8s-storage
   ```

2. **Create NFS Share:**
   - Navigate to: Sharing → Unix Shares (NFS)
   - Add new share:
     - Path: `/mnt/tank/k8s-storage`
     - Authorized Networks: Your Kubernetes subnet (e.g., `10.118.129.0/24`)
     - Maproot User: `root`
     - Maproot Group: `root`
   - Click "Submit"

3. **Enable NFS Service:**
   - Navigate to: Services
   - Enable "NFS" service
   - Set to start automatically

4. **Test NFS Export:**
   ```bash
   # From a Kubernetes node
   showmount -e <truenas-ip>
   # Should show: /mnt/tank/k8s-storage
   ```

### Network Requirements

- All Kubernetes nodes must be able to reach TrueNAS server
- NFS ports must be open (TCP/UDP 111, 2049, 20048)
- Recommended: 1Gbps+ network connection

## Role Variables

Available variables are listed below, along with default values (see `defaults/main.yml`):

### General Configuration

```yaml
# Enable/disable TrueNAS storage
truenas_storage_enabled: true

# Storage backend: nfs, iscsi, or both
storage_backend: "nfs"
```

### NFS Configuration

```yaml
# TrueNAS server IP or hostname
truenas_nfs_server: "192.168.1.100"

# NFS export path on TrueNAS
truenas_nfs_path: "/mnt/tank/k8s-storage"

# NFS mount options
truenas_nfs_mount_options: "nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2"

# StorageClass configuration
nfs_storage_class_name: "truenas-nfs"
nfs_archive_on_delete: "true"  # Keep data when PVC is deleted
nfs_reclaim_policy: "Retain"   # Retain or Delete
nfs_default_storage_class: true  # Set as default
```

### iSCSI Configuration (Future)

```yaml
# iSCSI support is planned for future releases
truenas_iscsi_portal: "192.168.1.100:3260"
truenas_iscsi_iqn: ""
truenas_api_url: "http://192.168.1.100/api/v2.0"
truenas_api_key: ""
```

## Dependencies

- Kubernetes cluster must be initialized
- Helm must be available on control plane
- NFS client packages will be installed automatically

## Example Playbook

```yaml
- name: Install TrueNAS Storage
  hosts: k8s_cluster
  become: true
  roles:
    - role: truenas-storage
      vars:
        truenas_nfs_server: "192.168.1.100"
        truenas_nfs_path: "/mnt/tank/k8s-storage"
        nfs_default_storage_class: true
```

## Configuration in group_vars

Add to `ansible/inventories/lab/group_vars/all.yml`:

```yaml
# TrueNAS Storage Configuration
truenas_storage_enabled: true
storage_backend: "nfs"
truenas_nfs_server: "192.168.1.100"  # Your TrueNAS IP
truenas_nfs_path: "/mnt/tank/k8s-storage"  # Your NFS export
```

## Usage

### Deploy Storage

```bash
ansible-playbook -i inventories/lab/hosts.yml playbooks/site.yml --tags storage
```

### Verify Installation

```bash
# Check StorageClass
kubectl get storageclass

# Check NFS provisioner pod
kubectl get pods -n kube-system -l app=nfs-subdir-external-provisioner

# Check default StorageClass
kubectl get storageclass -o jsonpath='{.items[?(@.metadata.annotations.storageclass\.kubernetes\.io/is-default-class=="true")].metadata.name}'
```

### Create Test PVC

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: truenas-nfs
EOF

# Check PVC status
kubectl get pvc test-pvc
# Should show: Bound
```

### Test with Pod

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: default
spec:
  containers:
  - name: test
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: test-pvc
EOF

# Write test file
kubectl exec test-pod -- sh -c "echo 'Hello TrueNAS' > /data/test.txt"

# Read test file
kubectl exec test-pod -- cat /data/test.txt

# Verify on TrueNAS
# Check /mnt/tank/k8s-storage/ for created subdirectory
```

## StorageClass Details

The role creates a StorageClass with these characteristics:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: truenas-nfs
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: nfs-subdir-external-provisioner
parameters:
  archiveOnDelete: "true"
reclaimPolicy: Retain
volumeBindingMode: Immediate
allowVolumeExpansion: false
```

### Access Modes Supported

- **ReadWriteOnce (RWO)**: Single pod read-write
- **ReadOnlyMany (ROX)**: Multiple pods read-only
- **ReadWriteMany (RWX)**: Multiple pods read-write ✅ (NFS advantage!)

## Troubleshooting

### NFS Provisioner Not Running

```bash
# Check pod status
kubectl get pods -n kube-system -l app=nfs-subdir-external-provisioner

# Check logs
kubectl logs -n kube-system -l app=nfs-subdir-external-provisioner

# Common issues:
# 1. Cannot connect to NFS server
# 2. NFS export not accessible
# 3. Permission denied
```

### PVC Stuck in Pending

```bash
# Describe PVC
kubectl describe pvc <pvc-name>

# Check events
kubectl get events --sort-by='.lastTimestamp'

# Common causes:
# 1. NFS server unreachable
# 2. Export path doesn't exist
# 3. Permission issues
```

### Test NFS Connectivity

```bash
# From any Kubernetes node
showmount -e <truenas-ip>

# Try manual mount
sudo mkdir -p /mnt/test
sudo mount -t nfs <truenas-ip>:/mnt/tank/k8s-storage /mnt/test
ls -la /mnt/test
sudo umount /mnt/test
```

### Permission Issues

On TrueNAS, ensure:
- Maproot User: `root`
- Maproot Group: `root`
- Or set specific UID/GID matching your application

### Network Issues

```bash
# Check NFS ports are open
nc -zv <truenas-ip> 2049
nc -zv <truenas-ip> 111

# Check firewall on TrueNAS
# Ensure Kubernetes subnet is in authorized networks
```

## Migration from Longhorn

### Backup Data

```bash
# List all PVCs
kubectl get pvc -A

# Backup important data
kubectl exec <pod> -n <namespace> -- tar czf /backup.tar.gz /data
kubectl cp <namespace>/<pod>:/backup.tar.gz ./backup.tar.gz
```

### Remove Longhorn

```bash
# Uninstall Longhorn
helm uninstall longhorn -n longhorn-system

# Delete CRDs
kubectl delete crd $(kubectl get crd | grep longhorn | awk '{print $1}')

# Delete namespace
kubectl delete namespace longhorn-system
```

### Deploy TrueNAS Storage

```bash
ansible-playbook -i inventories/lab/hosts.yml playbooks/site.yml --tags storage
```

### Restore Data

```bash
# Create new PVC with truenas-nfs StorageClass
# Deploy pod with new PVC
# Restore data from backup
kubectl cp ./backup.tar.gz <namespace>/<pod>:/backup.tar.gz
kubectl exec <pod> -n <namespace> -- tar xzf /backup.tar.gz -C /
```

## Performance Tuning

### NFS Mount Options

Adjust in `defaults/main.yml`:

```yaml
# For better performance
truenas_nfs_mount_options: "nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noatime"

# For better reliability
truenas_nfs_mount_options: "nfsvers=4.1,hard,intr,timeo=600,retrans=3"
```

### TrueNAS Optimizations

1. **Enable Async Writes**: Storage → Pools → Edit Dataset → Sync: Standard
2. **Disable Access Time**: Set `atime=off` on dataset
3. **Compression**: Enable LZ4 compression for space savings
4. **Recordsize**: Set to 128K for general use

## Security Considerations

1. **Network Isolation**: Use dedicated VLAN for storage traffic
2. **Firewall Rules**: Restrict NFS access to Kubernetes nodes only
3. **Encryption**: Consider NFSv4 with Kerberos for encryption
4. **Snapshots**: Enable regular snapshots on TrueNAS
5. **Quotas**: Set dataset quotas to prevent runaway storage usage

## Advanced Features

### Snapshots (via TrueNAS)

```bash
# Create snapshot on TrueNAS
# Storage → Snapshots → Add
# Name: k8s-storage-<date>
# Dataset: tank/k8s-storage
```

### Replication (via TrueNAS)

Configure TrueNAS replication tasks for disaster recovery:
- Tasks → Replication Tasks → Add
- Source: Local dataset
- Destination: Remote TrueNAS or cloud

### Monitoring

Integrate TrueNAS with Prometheus (optional):
- Use TrueNAS SNMP exporter
- Monitor dataset usage, I/O, and health

## Limitations

- **No Volume Expansion**: NFS provisioner doesn't support volume expansion
- **Performance**: Slightly slower than local storage or iSCSI
- **Single Point of Failure**: TrueNAS server is critical (use HA for production)
- **Network Dependency**: Storage performance depends on network quality

## Future Enhancements

- [ ] iSCSI support via democratic-csi
- [ ] TrueNAS API integration
- [ ] Automated snapshot scheduling
- [ ] Volume expansion support
- [ ] Multiple TrueNAS servers (HA)

## License

MIT

## Author Information

Created as part of the Kubernetes Ansible automation project.

## References

- [NFS Subdir External Provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)
- [TrueNAS Documentation](https://www.truenas.com/docs/)
- [Democratic CSI](https://github.com/democratic-csi/democratic-csi) (for future iSCSI support)
