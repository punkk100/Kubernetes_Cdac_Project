# Flannel CNI Role

This role installs and configures Flannel as the Container Network Interface (CNI) for Kubernetes.

## Description

Flannel is a simple and easy-to-configure overlay network that provides a Layer 3 network fabric for Kubernetes. It uses VXLAN as the default backend, which works across most network topologies without requiring special configuration.

## Features

- **Simple Configuration**: No complex BGP or routing protocols
- **VXLAN Backend**: Works across different network types and topologies
- **Automatic Setup**: Minimal configuration required
- **Cross-Node Networking**: Enables pod-to-pod communication across nodes
- **Lightweight**: Lower resource usage compared to more complex CNI solutions

## Requirements

- Kubernetes cluster initialized with `kubeadm`
- `kubectl` installed and configured on the control plane
- Pod network CIDR configured during `kubeadm init` (default: `10.244.0.0/16`)

## Role Variables

Available variables are listed below, along with default values (see `defaults/main.yml`):

```yaml
# Flannel manifest URL
flannel_manifest_url: "https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml"

# Pod network CIDR (must match kubeadm init --pod-network-cidr)
pod_network_cidr: "10.244.0.0/16"
```

## Dependencies

- `kubernetes` role must be run first to initialize the cluster

## Example Playbook

```yaml
- name: Install Flannel CNI
  hosts: k8s_control_plane
  become: true
  roles:
    - role: flannel
```

## Tasks Overview

1. **Prerequisite Checks**: Verify kubectl availability
2. **API Server Readiness**: Wait for Kubernetes API server to be ready
3. **Cleanup**: Remove old CNI configurations (e.g., from previous Calico installation)
4. **Download Manifest**: Fetch the latest Flannel manifest
5. **CIDR Configuration**: Update pod network CIDR if custom value is used
6. **Apply CNI**: Deploy Flannel to the cluster
7. **Verification**: Wait for Flannel pods to be running and nodes to be Ready

## Network Backend

Flannel uses **VXLAN** as the default backend:
- Encapsulates pod traffic in UDP packets
- Works across NAT and firewalls
- No special network requirements
- Port used: UDP 8472

## Troubleshooting

### Pods Not Running
```bash
kubectl get pods -n kube-flannel
kubectl logs -n kube-flannel -l app=flannel
```

### Nodes Not Ready
```bash
kubectl get nodes
kubectl describe node <node-name>
```

### Network Connectivity Issues
```bash
# Check Flannel interface
ip addr show flannel.1

# Check routes
ip route | grep flannel
```

## Migration from Calico

If migrating from Calico:
1. Reset the cluster: `kubeadm reset -f` on all nodes
2. Clean CNI directories: `/etc/cni/net.d/`, `/var/lib/cni/`
3. Update `pod_network_cidr` to `10.244.0.0/16` (or keep existing if preferred)
4. Run the playbook with this role

## License

MIT

## Author Information

Created as part of the Kubernetes Ansible automation project.
