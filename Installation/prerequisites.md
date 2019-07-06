# Prequisites
I used 7 nodes for OpenStack and Kubernetes cluster, and use 2 network interface each node.

#### OpenStack Nodes
- 1 Controller Node
- 1 Network Node
- 2 Compute Node
- CentOS based
- OpenStack Stein
- Octavia need to Installed as Load Balancing

#### Kubernetes Nodes
- 1 Master Node
- 2 Worker Node
- Ubuntu based
- Kubernetes v1.13.3
- Pod Network 10.1.0.0/16
- Service Network 10.2.0.0/17

## Topology
With 2 network, 1 for management node, and 1 for external access, This is the design:
![network-topology](/assets/kuryr-kubernetes-topology-1.png)

The internal bridge structure is:
![network-topology](/assets/internal-diagram-bridge-kuryr-kubernetes.png)

