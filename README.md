# calico-learning
Project Calico Crash Course

I am assuming you mean Project Calico, the Kubernetes networking and network-security platform—not Calico Linux, Calico cat, or Google’s biotechnology company.

The current official documentation is Calico Open Source 3.32, with installation examples using v3.32.1 manifests.

1. What is Calico?

Calico provides three important capabilities for Kubernetes:

Pod networking — assigns IP addresses and connects pods across nodes.
Network security — controls which pods, namespaces, hosts, services, and external networks can communicate.
IP address management, or IPAM — manages pod IP pools and allocation.

Calico can act as the complete Kubernetes CNI, or it can provide only network-policy enforcement while another CNI supplies networking.

A simple mental model is:

Kubernetes creates a pod
        ↓
Calico CNI connects the pod
        ↓
Calico IPAM assigns an IP
        ↓
Calico programs routes
        ↓
Felix programs security rules
        ↓
The pod can communicate only as permitted
2. Kubernetes networking concepts you need first
CNI

CNI means Container Network Interface.

Kubernetes itself does not directly build the pod network. When Kubernetes creates a pod, it calls a CNI plugin such as Calico, Cilium, Flannel or another networking solution.

The CNI plugin normally:

Creates the pod network interface.
Connects the pod to the host.
Assigns an IP address.
Adds required routes.
Removes networking when the pod is deleted.

Calico connects pods using a virtual Ethernet pair, commonly called a veth pair, and uses Layer 3 routing rather than putting every pod behind a traditional Layer 2 bridge.

Pod IP

Every Kubernetes pod normally receives its own IP address.

Example:

Node 1: 192.168.0.0/26
  frontend-pod: 192.168.0.10
  backend-pod:  192.168.0.11

Node 2: 192.168.0.64/26
  database-pod: 192.168.0.70

Calico must make sure that:

192.168.0.10 → 192.168.0.70

reaches the correct node and pod.

Service IP

A Kubernetes Service IP is virtual. It is not normally assigned directly to a physical interface.

For example:

Service: backend-service
ClusterIP: 10.96.45.10
Backend pods:
  192.168.0.11
  192.168.0.75

The networking data plane translates or load-balances traffic from the Service IP to one of the backend pod IPs.

Depending on configuration, this may be handled by:

kube-proxy with iptables or nftables.
Calico’s eBPF data plane.
Another service implementation.
3. What Calico installs

A production Calico deployment contains several components.

3.1 Tigera Operator

The Tigera Operator installs and manages Calico components through Kubernetes custom resources.

It handles:

Installation.
Configuration.
Component lifecycle.
Scaling.
Upgrades.

The operator installation method is recommended over manually applying raw manifests because the operator manages the lifecycle of the installation.

3.2 Calico CNI plugin

The CNI plugin runs when pods are created or removed.

It performs tasks such as:

Create veth pair
Move one side into the pod namespace
Assign the pod IP
Configure routes
Register the pod as a Calico workload endpoint

You can examine CNI configuration on a node under:

ls -l /etc/cni/net.d/

Typical files may include:

10-calico.conflist
calico-kubeconfig
3.3 Felix

Felix is one of the most important Calico components.

It runs on every Kubernetes node, normally as part of the calico-node DaemonSet.

Felix watches the Calico datastore and programs the node’s data plane.

Depending on the selected data plane, Felix can configure:

Linux routes.
iptables rules.
nftables rules.
IP sets.
eBPF programs.
Workload interface settings.
Network-policy enforcement.
NAT and forwarding behaviour.

Think of Felix as:

Calico configuration → Felix → Linux networking configuration
3.4 BIRD

BIRD is a routing daemon used for Calico’s BGP-based networking.

It exchanges pod network routes between:

Kubernetes nodes.
Route reflectors.
Top-of-rack routers.
External BGP peers.

A useful mental model is that each Calico node can behave like a virtual router.

Example:

Node A knows:
192.168.0.0/26 is local

Node B advertises:
192.168.0.64/26 via 10.0.0.12

Node A routing table:
192.168.0.64/26 via 10.0.0.12

BIRD is relevant mainly when BGP networking is enabled. A VXLAN-only configuration may not require BGP for internal cluster routes.

3.5 confd

confd monitors Calico networking configuration and generates configuration used by BIRD.

For example, when a BGP peer or autonomous system number changes:

Calico datastore
       ↓
confd detects change
       ↓
BIRD configuration is updated
       ↓
BIRD reloads routing configuration
3.6 Calico IPAM

Calico IPAM allocates pod addresses from one or more IP pools.

An IP pool could be:

apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: production-pool
spec:
  cidr: 192.168.0.0/16
  natOutgoing: true
  vxlanMode: CrossSubnet

Calico divides IP pools into smaller blocks and associates blocks with nodes. The default IPv4 block size is generally /26, which contains 64 addresses, although the block size can be configured.

Example:

Main pool: 192.168.0.0/16

Node 1 receives: 192.168.0.0/26
Node 2 receives: 192.168.0.64/26
Node 3 receives: 192.168.0.128/26

Pods are then allocated addresses from their node’s block.

3.7 kube-controllers

Calico’s Kubernetes controllers synchronize Kubernetes state with Calico state.

They can manage areas such as:

Nodes.
Workload endpoints.
IPAM.
Kubernetes service accounts.
Namespace information.
Pod-related Calico resources.
3.8 Typha

Typha is mainly useful in large clusters.

Without Typha:

Every Felix instance → Kubernetes API/datastore

With Typha:

Many Felix instances → Typha → Kubernetes API/datastore

Typha reduces the number of direct datastore connections and distributes updates efficiently.

3.9 Calico API server

The Calico API server exposes Calico resources through Kubernetes-style APIs, allowing administrators to manage supported Calico resources through tools such as kubectl.

For example:

kubectl get ippools
kubectl get globalnetworkpolicies
kubectl get networkpolicies.projectcalico.org -A
3.10 calicoctl

calicoctl is Calico’s administrative CLI.

It is useful for:

calicoctl get nodes
calicoctl get ippool -o wide
calicoctl get bgppeer
calicoctl get globalnetworkpolicy
calicoctl node status
calicoctl ipam show

It can manage Calico configuration and display lower-level networking status that may not be obvious through regular Kubernetes commands.

4. How a packet travels through Calico

Suppose:

frontend-pod
IP: 192.168.0.10
Node: worker-1

backend-pod
IP: 192.168.0.70
Node: worker-2

The frontend sends traffic to the backend:

192.168.0.10 → 192.168.0.70:8080

A simplified flow is:

frontend container
       ↓
pod eth0
       ↓
veth pair
       ↓
worker-1 Linux network stack
       ↓
Calico policy evaluation
       ↓
routing decision
       ↓
BGP route or VXLAN/IPIP tunnel
       ↓
worker-2
       ↓
Calico policy evaluation
       ↓
backend pod veth
       ↓
backend container

On the traditional Linux data plane, packets are handled through Linux routing and iptables or nftables. In eBPF mode, Calico attaches eBPF programs at kernel hooks and can process packets earlier in the path.

5. Calico networking modes

Choosing the networking mode is one of the most important Calico design decisions.

5.1 Non-overlay BGP networking

Pods use routable IP addresses, and routes are exchanged through BGP.

Pod on Node A
     ↓
Normal routed packet
     ↓
Physical network
     ↓
Node B
     ↓
Destination pod

There is no additional tunnel header.

Advantages
Better performance.
Lower encapsulation overhead.
Pod IPs can become first-class routable network addresses.
Easier integration with BGP-capable physical networks.
Considerations
The underlying network must know how to reach pod CIDRs.
BGP peering may need to be configured.
Cloud networks may not permit the required routing architecture.

Calico generally recommends non-overlay networking when the underlying network can route workload addresses.

5.2 IP-in-IP

IP-in-IP places the original IPv4 packet inside another IPv4 packet.

Original:
Pod A IP → Pod B IP

Encapsulated:
Node A IP → Node B IP
  inside:
  Pod A IP → Pod B IP
Useful when
The underlay cannot route pod IPs.
You still want BGP route distribution.
Your infrastructure supports IP protocol 4.
Limitation

IP-in-IP is an IPv4-oriented encapsulation option and may be blocked by some networks or cloud environments.

5.3 VXLAN

VXLAN encapsulates pod traffic inside UDP.

Default Calico VXLAN traffic commonly uses:

UDP 4789

VXLAN is particularly useful when the physical network does not understand pod routes or does not support Calico BGP integration. A VXLAN-only Calico network does not require BGP for internal workload routes.

Advantages
Works in many cloud and virtualized environments.
Uses normal UDP transport.
Does not require the physical network to know pod CIDRs.
Disadvantages
Adds tunnel overhead.
Reduces available MTU.
Requires correct firewall and MTU configuration.
5.4 Cross-subnet encapsulation

Instead of encapsulating every packet, Calico can encapsulate only packets crossing subnet boundaries.

Example:

Node A and Node B in subnet 10.0.1.0/24
→ No encapsulation

Node A in 10.0.1.0/24
Node C in 10.0.2.0/24
→ Encapsulation

Cross-subnet mode reduces unnecessary tunnelling while still supporting networks that cannot route pod addresses between subnets. Calico recommends it where suitable.

5.5 eBPF data plane

Calico eBPF is an alternative to the standard Linux iptables-based data plane.

It can:

Enforce network policies.
Route pod traffic.
Implement Kubernetes Service handling.
Reduce dependence on kube-proxy.
Preserve source IPs in supported configurations.
Improve service-processing efficiency.

Calico’s official documentation describes higher throughput, lower CPU use per unit of traffic, and native Kubernetes Service handling as major eBPF benefits. Current installation documentation requires at least Linux kernel 5.10 and recommends newer kernels for the complete feature set.

Do not enable eBPF merely because it sounds faster. First verify:

Kernel compatibility
Cloud platform limitations
kube-proxy configuration
VXLAN connectivity
Existing IPIP configuration
Host networking requirements
6. Choosing the right mode
Environment	Practical starting choice
Local learning with kind	Default operator configuration
Simple cloud cluster	VXLAN or platform-supported mode
Multiple cloud subnets	VXLAN CrossSubnet
On-premises BGP-capable network	Non-overlay BGP
Underlay cannot route pod IPs	VXLAN
Performance-focused modern Linux	Evaluate eBPF
Need encrypted node-to-node pod traffic	WireGuard
Existing CNI already supplies networking	Calico policy-only deployment

Calico supports both overlay and non-overlay networking, allowing the design to be matched to the capabilities of the underlying infrastructure.

7. Hands-on Calico laboratory
Prerequisites

Install:

Docker
kind
kubectl

The following lab creates one control-plane and two worker nodes.

Step 1: Create a kind configuration

Create kind-calico.yaml:

kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
  - role: control-plane
  - role: worker
  - role: worker

networking:
  disableDefaultCNI: true
  podSubnet: 192.168.0.0/16

Create the cluster:

kind create cluster \
  --name calico-cluster \
  --config kind-calico.yaml

Initially, the nodes may appear NotReady because no CNI is installed:

kubectl get nodes

This is expected until Calico establishes cluster networking.

Step 2: Install Calico

Install the Calico CRDs:

kubectl create -f \
https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/v1_crd_projectcalico_org.yaml

Install the Tigera Operator:

kubectl create -f \
https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/tigera-operator.yaml

Create the Calico installation resources:

kubectl create -f \
https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/custom-resources.yaml

Monitor status:

kubectl get tigerastatus

Watch continuously:

watch kubectl get tigerastatus

The official operator installation uses these resources and reports component availability through tigerastatus.

Verify:

kubectl get nodes
kubectl get pods -n calico-system
kubectl get pods -n tigera-operator

The nodes should eventually become:

Ready
8. Deploy a test application

Create calico-lab.yaml:

apiVersion: v1
kind: Namespace
metadata:
  name: calico-lab

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: calico-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
        role: api
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: calico-lab
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 80

---
apiVersion: v1
kind: Pod
metadata:
  name: client
  namespace: calico-lab
  labels:
    app: client
    access: approved
spec:
  containers:
    - name: client
      image: curlimages/curl
      command: ["sleep", "3600"]

Apply it:

kubectl apply -f calico-lab.yaml

Check pod placement and IPs:

kubectl get pods -n calico-lab -o wide

Test access:

kubectl exec -n calico-lab client -- \
  curl -s --connect-timeout 3 http://backend

It should initially work because Kubernetes pods are normally unrestricted when no network policy selects them.

9. Kubernetes NetworkPolicy

A Kubernetes NetworkPolicy is namespace-scoped and selects pods through labels.

Important idea:

No policy selects pod
→ Traffic is allowed

Ingress policy selects pod
→ Only explicitly permitted ingress is allowed

Egress policy selects pod
→ Only explicitly permitted egress is allowed

Ingress and egress isolation operate independently.

9.1 Default-deny policy

Create default-deny.yaml:

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: calico-lab
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress

Apply:

kubectl apply -f default-deny.yaml

Test again:

kubectl exec -n calico-lab client -- \
  curl --connect-timeout 3 http://backend

The connection should fail.

A namespaced default-deny policy is a recommended starting point after required allow policies have been planned and tested.

However, this policy also blocks DNS egress. The client may not even resolve backend.

9.2 Allow DNS

CoreDNS normally uses UDP and TCP port 53.

Create allow-dns.yaml:

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: calico-lab
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53

Apply:

kubectl apply -f allow-dns.yaml

DNS should work, but backend access remains blocked because backend ingress and client egress have not yet been allowed.

9.3 Allow approved clients to reach backend

Create allow-client-to-backend.yaml:

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-client-to-backend
  namespace: calico-lab
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              access: approved
      ports:
        - protocol: TCP
          port: 80

This allows ingress to app=backend from pods with:

access=approved

But because the client is also egress-isolated, create an egress policy.

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-client-egress-to-backend
  namespace: calico-lab
spec:
  podSelector:
    matchLabels:
      access: approved
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 80

Apply both policies and test:

kubectl exec -n calico-lab client -- \
  curl -s --connect-timeout 3 http://backend

It should work again.

10. Calico NetworkPolicy versus Kubernetes NetworkPolicy

Calico supports standard Kubernetes NetworkPolicy, but also offers its own policy resources.

Kubernetes policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

Good for:

Portable Kubernetes policy.
Basic ingress and egress allow rules.
Pod and namespace selectors.
Port and CIDR-based controls.

Limitations include:

No explicit Deny action.
No policy ordering field.
Primarily pod-oriented.
Fewer selector and rule features.
Calico policy
apiVersion: projectcalico.org/v3
kind: NetworkPolicy

Calico policy adds capabilities such as:

Explicit Allow and Deny.
Policy ordering.
Rich Calico selectors.
Service-account selectors.
Host endpoint policies.
Global policies.
Broader endpoint support.

Calico’s policy model extends Kubernetes policy with ordering, deny rules and more flexible matching.

Example Calico policy
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: calico-lab
spec:
  order: 100
  selector: app == "backend"
  types:
    - Ingress
  ingress:
    - action: Allow
      source:
        selector: access == "approved"
      protocol: TCP
      destination:
        ports:
          - 80
    - action: Deny

Apply with:

kubectl apply -f backend-policy.yaml

Or:

calicoctl apply -f backend-policy.yaml
Understanding order

Lower numbers are considered earlier.

For example:

order: 10   platform policy
order: 100  security policy
order: 500  application policy

Do not depend only on ordering to create an understandable policy design. Labels, scopes and policy tiers should also be organised clearly.

11. GlobalNetworkPolicy

A regular Calico NetworkPolicy is namespaced.

A GlobalNetworkPolicy can apply across namespaces and may also protect other Calico endpoint types.

Example:

apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: block-malicious-range
spec:
  order: 50
  selector: all()
  types:
    - Ingress
  ingress:
    - action: Deny
      source:
        nets:
          - 203.0.113.0/24
    - action: Allow

Global policy is powerful and dangerous. A global deny that selects system namespaces or host endpoints can disrupt:

DNS.
Kubernetes API access.
Node health.
Calico itself.
Monitoring.
Admission webhooks.

Official guidance recommends carefully scoping global default-deny policy away from system workloads and testing allow behaviour before enforcing broad deny behaviour.

12. Policy tiers

Policy tiers provide administrative separation and evaluation precedence.

A possible design is:

platform tier
    ↓
security tier
    ↓
application tier
    ↓
default tier

Example responsibilities:

Tier	Owner	Purpose
Platform	Cluster administrators	Protect system infrastructure
Security	Security team	Enterprise-wide restrictions
Application	Application teams	Application communication
Default	General workloads	Remaining policies

Calico recommends thinking about tiers as groups of policies controlled by different organisational responsibilities.

13. Selectors

Calico selectors are more expressive than basic Kubernetes matchLabels.

Examples:

app == "frontend"
environment in {"staging", "production"}
has(compliance)
app == "api" && environment == "production"
role != "database"

A selector identifies endpoints. It does not by itself allow traffic.

You still need a rule such as:

ingress:
  - action: Allow
    source:
      selector: role == "frontend"
14. Egress control

Egress policy controls where pods may connect.

Example: allow HTTPS only to a specific network.

apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: restricted-egress
  namespace: calico-lab
spec:
  selector: app == "client"
  types:
    - Egress
  egress:
    - action: Allow
      protocol: TCP
      destination:
        nets:
          - 198.51.100.0/24
        ports:
          - 443
    - action: Deny

Remember that external services may use:

Multiple IP addresses.
CDNs.
Frequently changing DNS results.
Separate authentication endpoints.
Different ports for telemetry and updates.

Therefore, IP-based egress allowlisting may require maintenance.

15. HostEndpoint

Most policies protect pods. Calico can also protect node interfaces using HostEndpoint.

Conceptually:

apiVersion: projectcalico.org/v3
kind: HostEndpoint
metadata:
  name: worker-1-eth0
  labels:
    node-role: worker
spec:
  node: worker-1
  interfaceName: eth0
  expectedIPs:
    - 10.0.0.11

You can then apply policy to node traffic.

Use cases include:

Restricting SSH.
Protecting kubelet ports.
Protecting node exporters.
Limiting management network access.
Controlling host-to-host communication.

Host endpoint policy should be introduced carefully because an incorrect rule can lock administrators out of nodes.

16. NAT and outbound access

Pods often use private IP addresses that are not routable on the public internet.

With natOutgoing: true, Calico can source-NAT traffic when it leaves the Calico IP pool.

Example:

Pod source: 192.168.0.10
Node source after SNAT: 10.0.0.11
External destination: 8.8.8.8

IPPool example:

apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-ipv4-pool
spec:
  cidr: 192.168.0.0/16
  natOutgoing: true
  vxlanMode: CrossSubnet
  nodeSelector: all()
17. MTU

MTU is the maximum packet size that can cross a network interface without fragmentation.

Typical Ethernet:

1500-byte MTU

Encapsulation adds headers:

Original packet
+ VXLAN header
+ UDP header
+ outer IP header

Therefore, the pod MTU must normally be smaller than the underlying network MTU.

Symptoms of an MTU problem include:

Ping works with small packets but fails with larger packets.
TCP connections establish but hang while transferring data.
DNS works while HTTP requests stall.
Cross-node communication fails, but same-node communication works.
TLS handshakes fail intermittently.

Testing:

ping -M do -s 1400 <destination-ip>

Inspect interfaces:

ip link
ip addr
18. BGP essentials
Autonomous system number

An AS number identifies a BGP routing domain.

A default private AS often appears as:

64512

Inspect BGP configuration:

calicoctl get bgpconfiguration -o yaml
Node-to-node mesh

In a small cluster, every node can peer directly with every other node.

For n nodes, the number of peer relationships grows approximately as:

n × (n - 1) / 2

Example:

3 nodes  → 3 sessions
10 nodes → 45 sessions
100 nodes → 4,950 sessions

This is why large BGP clusters commonly use route reflectors.

Route reflector

Instead of every node peering with every other node:

Nodes → Route reflectors → Other nodes

This reduces BGP session complexity.

Check BGP
sudo calicoctl node status

Healthy peers generally show:

Established

The command displays Calico node status and BGP peering state.

19. WireGuard encryption

Calico can configure WireGuard tunnels to encrypt inter-node pod traffic.

Simplified path:

Pod A
  ↓ unencrypted local host segment
Node A
  ↓ encrypted WireGuard host-to-host tunnel
Node B
  ↓ unencrypted local host segment
Pod B

Calico manages the tunnels automatically when the feature is enabled. Encryption covers the node-to-node portion, not necessarily every byte from inside one container to inside another container.

Relevant default ports include:

IPv4 WireGuard: UDP 51820
IPv6 WireGuard: UDP 51821

20. Essential troubleshooting workflow

Use a layer-by-layer approach.

Layer 1: Is Kubernetes healthy?
kubectl get nodes -o wide
kubectl get pods -A
kubectl get events -A --sort-by=.lastTimestamp

Look for:

NotReady
CrashLoopBackOff
ImagePullBackOff
FailedCreatePodSandBox
NetworkPluginNotReady
Layer 2: Are Calico components healthy?
kubectl get tigerastatus
kubectl get pods -n calico-system -o wide
kubectl get pods -n tigera-operator

Check DaemonSet coverage:

kubectl get daemonset -n calico-system

There should normally be a Calico node pod on each relevant Kubernetes node.

Layer 3: Check logs
kubectl logs -n calico-system <calico-node-pod>

For a specific container:

kubectl logs -n calico-system <pod> -c calico-node

Previous container:

kubectl logs -n calico-system <pod> \
  -c calico-node --previous

Operator logs:

kubectl logs -n tigera-operator \
  deployment/tigera-operator

Calico’s official troubleshooting guidance recommends checking component logs and using calicoctl node diags for diagnostics.

Layer 4: Check pod IP allocation
kubectl get pods -A -o wide
calicoctl ipam show
calicoctl ipam show --show-blocks
calicoctl get ippool -o wide

Check whether:

The pool is exhausted.
A pool is disabled.
The pool overlaps the node network.
The pool overlaps the Service CIDR.
Pods are receiving unexpected IP ranges.
Blocks remain associated with removed nodes.
Layer 5: Check routing

On the affected node:

ip route
ip route get <destination-pod-ip>

Look for Calico interfaces:

ip link | grep cali

A pod interface often appears with a name beginning with:

cali

For BGP:

sudo calicoctl node status

For route inspection:

birdc show protocols
birdc show route

Command availability depends on the installation and container context.

Layer 6: Test same-node versus cross-node

Get pod placement:

kubectl get pods -A -o wide

Then compare:

Pod A → Pod B on same node
Pod A → Pod C on another node

Interpretation:

Result	Likely area
Same-node works, cross-node fails	BGP, VXLAN, IPIP, firewall or MTU
Both fail	Policy, pod interface, CNI or application
Pod IP works, Service IP fails	kube-proxy or eBPF service handling
IP works, DNS name fails	DNS or DNS egress policy
Only large transfers fail	MTU issue
Layer 7: Test DNS
kubectl exec -n calico-lab client -- \
  nslookup kubernetes.default.svc.cluster.local

Check CoreDNS:

kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

Check the pod’s DNS file:

kubectl exec -n calico-lab client -- cat /etc/resolv.conf
Layer 8: Test network policy

List policies:

kubectl get networkpolicy -A
kubectl get networkpolicies.projectcalico.org -A
kubectl get globalnetworkpolicies

Describe a Kubernetes policy:

kubectl describe networkpolicy \
  -n calico-lab default-deny

Inspect pod labels:

kubectl get pods -n calico-lab --show-labels

Many policy failures are actually selector failures:

Policy expects:
app=backend

Pod has:
app=back-end

Those labels do not match.

21. Common Calico problems
Nodes remain NotReady

Possible causes:

CNI configuration not created.
Calico node pod is failing.
Pod CIDR mismatch.
Kernel module missing.
Firewall blocks overlay or BGP traffic.
Operator installation is incomplete.

Check:

kubectl describe node <node-name>
kubectl get pods -n calico-system
kubectl get tigerastatus
FailedCreatePodSandBox

Possible causes:

CNI binary missing.
Invalid CNI configuration.
IPAM allocation failure.
Calico API unavailable.
Pod IP pool exhausted.
Stale CNI files from another plugin.

Check:

kubectl describe pod <pod-name>
journalctl -u kubelet
ls -l /etc/cni/net.d/
ls -l /opt/cni/bin/
Cross-node traffic fails

Check required networking:

BGP: TCP 179.
VXLAN: UDP 4789.
WireGuard IPv4: UDP 51820.
WireGuard IPv6: UDP 51821.
IP-in-IP: IP protocol 4.

Official Calico requirements list the relevant protocol and port requirements according to the selected networking mode.

DNS breaks after default deny

Cause:

Default-deny egress blocks DNS

Solution:

Permit UDP 53.
Permit TCP 53.
Select the correct CoreDNS pods or Service.
Confirm namespace labels.
Internet access fails

Check:

calicoctl get ippool -o yaml

Look for:

natOutgoing: true

Also inspect:

ip route
iptables -t nat -S

In eBPF mode, use eBPF-specific inspection and troubleshooting rather than relying only on iptables.

22. Production design recommendations
Start with labels

Create a consistent label model:

app
component
environment
team
data-classification
internet-access
compliance-zone

Example:

labels:
  app: payments
  component: api
  environment: production
  data-classification: restricted

Policies become difficult to maintain when labels are inconsistent.

Use default deny gradually

A safe rollout sequence is:

Observe traffic
    ↓
Create explicit allow policies
    ↓
Test in staging
    ↓
Apply namespaced default deny
    ↓
Validate DNS, monitoring and backups
    ↓
Expand carefully

Avoid beginning with an unrestricted cluster-wide deny policy.

Separate responsibilities

A useful policy ownership model:

Platform team:
  system infrastructure policies

Security team:
  organisation-wide restrictions

Application teams:
  application-to-application rules
Treat policies as code

Store policies in Git and apply:

Code review.
Naming standards.
Automated validation.
Staging tests.
GitOps deployment.
Rollback procedures.
Monitor IP consumption

Regularly inspect:

calicoctl ipam show
calicoctl ipam show --show-blocks

Do not wait until the pod network is nearly exhausted.

23. Key commands cheat sheet
# Overall health
kubectl get nodes -o wide
kubectl get pods -A
kubectl get tigerastatus

# Calico components
kubectl get pods -n calico-system -o wide
kubectl logs -n calico-system <pod-name>

# Policies
kubectl get networkpolicy -A
kubectl get networkpolicies.projectcalico.org -A
kubectl get globalnetworkpolicies

# IPAM
calicoctl get ippool -o wide
calicoctl ipam show
calicoctl ipam show --show-blocks

# BGP
calicoctl node status
calicoctl get bgpconfiguration -o yaml
calicoctl get bgppeer -o wide

# Node networking
ip addr
ip link
ip route
ip route get <pod-ip>

# Connectivity
curl -v http://<service>
ping <pod-ip>
nslookup <service-name>
nc -vz <destination> <port>

# Diagnostics
sudo calicoctl node diags
24. Calico interview questions
What is Calico?

Calico is a Kubernetes networking and network-security solution that can provide CNI networking, IP address management, routing and network-policy enforcement.

What is Felix?

Felix is the per-node Calico agent that programs routes and policy rules into the Linux networking data plane.

What is BIRD?

BIRD is the routing daemon used by Calico for BGP route exchange.

What is Calico IPAM?

Calico IPAM allocates pod IP addresses from IP pools and assigns address blocks to nodes.

What is the difference between VXLAN and IP-in-IP?

VXLAN encapsulates traffic in UDP, while IP-in-IP places an IP packet inside another IP packet. VXLAN can be used without BGP for internal overlay routing; Calico IP-in-IP commonly works with BGP.

What happens when no NetworkPolicy selects a pod?

Traffic is allowed by default.

What happens after an ingress policy selects a pod?

Only ingress traffic allowed by one or more applicable policies is permitted.

Kubernetes NetworkPolicy versus Calico NetworkPolicy?

Kubernetes policy provides portable allow-based pod controls. Calico policy adds explicit deny actions, ordering, advanced selectors, global policy and broader endpoint support.

What is a GlobalNetworkPolicy?

A cluster-scoped Calico policy capable of applying across namespaces and, depending on its selector, other endpoint types.

Why use route reflectors?

They reduce the number of BGP peer sessions required in a large cluster.

Why does DNS stop after default deny?

Because DNS traffic on TCP/UDP port 53 was not explicitly permitted.

Why does same-node traffic work while cross-node traffic fails?

The pod interfaces and local policy may be correct, but BGP, VXLAN, IP-in-IP, firewall, routing or MTU configuration may be broken.

25. What you should remember
CNI connects the pod.
IPAM assigns the IP.
Felix programs routes and policies.
BIRD exchanges BGP routes.
VXLAN/IPIP handle overlay transport.
NetworkPolicy controls communication.
GlobalNetworkPolicy applies wider controls.
Typha supports scalability.
The operator manages the installation.
calicoctl provides operational visibility.

The best learning sequence is:

Kubernetes networking basics
→ Calico architecture
→ Pod routing
→ IPAM
→ Kubernetes NetworkPolicy
→ Calico policy
→ BGP and overlays
→ Troubleshooting
→ Production policy design
