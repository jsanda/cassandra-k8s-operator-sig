# Topology Design Doc
The purpose of this document is to identify and define data types that are needed for specifying a Cassandra cluster's topology.

# Racks
A `Rack` maps directly to a Cassandra rack. A Cassandra rack is a logical grouping of C* nodes that can be co-located within a zone. Cassandra designates replicas across racks. 

A `Rack` object should minimally be expressive enough to pin C* pods to a specific zone.

All four operator have a well-defined type for racks. A `Rack` object should provide the ability to do the following for the Cassandra nodes that constitute a rack:

* Constrain C* pods to be scheduled to a specific zone
* Constrain C* pods to particular k8s worker nodes
* Constrain C* pods to particular k8s worker nodes based on the pods already running on those nodes
* Prevent C* pods from being scheduled on k8s worker nodes that do  meet certain criteria, e.g., the node does not have local SSDs.

All of these things can be accomplished using node affinity, pod anti-affinity, and taints and tolerations.

```go
// A Rack has a 1-to-1 mapping to a StatefulSet. Its properties
// are primarily intended to help determine where pods should
// or should not be scheduled.
type Rack struct {
	// The rack name
	Name string `json:"name"`
	
	// Used to establish node affinity and/or pod anti-affinity
	// rules.
	Labels map[string]string `json:"labels,omitempty"`
	
	// Annotations to apply to each pod in the rack.
	Annotations map[string]string `json:"annotations,omitempty"`
	
	// Allows the pods to be scheduled onto nodes with matching
	// taints.
	Tolerations []corev1.Toleration `json:"tolerations,omitempty"`
}
```

## Questions
### Where is Cassandra configuration?

The primary purpose of a `Rack` is to specify topology which includes where pods will be scheduled. Cassandra configured will be addressed elsewhere.

### How come there is no property to specify the number of C* nodes?
We want balanced racks. Instead of specifying nodes at the rack level, it should be a setting that applies across racks to help enforce balance.

### Should other parts of the StatefulSet should be exposed at the rack level?

### Does there need to be a rack-level status?


# Datacenter
A `Datacenter` maps to a C* datacenter. A C* datacenter is a logical grouping of C* nodes that is configured for replication purposes.

A `Datacenter` is comprised of `Racks` and settings that are shared among them.

```go
type PodAntiAffintyType string

const (
	// denotes requiredDuringSchedulingIgnoredDuringExecution
	Required PodAntiAffintyType = "required"
	
	// denotes preferredDuringSchedulingIgnoredDuringExecution
	Preferred PodAntiAffintyType = "preferred"
	
	// Denotes that pod anti-affinity should not be used
	Disabled PodAntiAffinityType = "disabled"
)

// A Cassandra cluster  consists of one or more datacenters that are comprised of
// racks. Balanced racks are enforced by only allowing the number of nodes per rack
// to be specified as opposed to the totl number of nodes in the DC.
type Datacenter struct {
	// The cluster to which the DC belongs.
	Cluster string `json:"name"`

	// The DC name.
	Name string `json:"name"`
	
	NodesPerRack int32 `json:"nodesPerRack,omitempty`
	
	Racks []Rack `json:"racks,omitempty"`
	
	PodAntiAffinity PodAntiAffinityType `json:"podAntiAffinity,omitempty"
	
	// Used to establish node affinity and/or pod anti-affinity
	// rules. These will be merged with rack-level labels.
	Labels map[string]string `json:"labels,omitempty"`
	
	// Annotations to apply to each pods in each rack. These will be merged with 
	// rack-level annotations.
	Annotations map[string]string `json:"annotations,omitempty"`
	
	// Allows the pods to be scheduled onto nodes with matching taints. These will be
	// merged with rack-level tolerations.
	Tolerations []corev1.Toleration `json:"tolerations,omitempty"`
	
	// TODO add property for PodDisruptionBudget
}
```

## Questions
### How is the snitch configured?
The operator will automatically set the snitch to `GossipingPropertyFileSnitch`.

### How are seeds configured?
The operator will automatically configure seed nodes.

# Cluster
```go
type Cluster struct {
	Name string `json:"name"`
	
	// Used to establish node affinity and/or pod anti-affinity rules. These will be
	// merged with datacenter-level labels.
	Labels map[string]string `json:"labels,omitempty"`
	
	// Annotations to apply to pods in each racks. These will be merged with
	// datacenter-level annotations.
	Annotations map[string]string `json:"annotations,omitempty"`
	
	// Allows the pods to be scheduled onto nodes with matching
	// taints. These will be merged with datacenter-level tolerations.
	Tolerations []corev1.Toleration `json:"tolerations,omitempty"`
	
	Datacenters []Datacenter `json:"datacenters,omitempty
}
```

## Questions
### Should Datacenter be its own resource type?


## Examples
### One DC and One Rack
```yaml
apiVersion: cassandra/v1alpha1
kind: Cluster
metadata:
  name: one-dc-one-rack
spec:
  name: one-dc-one-rack
  datacenters:
    - name: dc1
      nodesPerRack: 3
```

### One DC and Multiple Racks
```yaml
apiVersion: cassandra/v1alpha1
kind: Cluster
metadata:
  name: one-dc-multiple-racks
spec:
  name: one-dc-multiple-racks
  datacenters:
    - name: dc1
      nodesPerRack: 3
      racks:
        - name: rack1
        - name: rack2
        - name: rack3
```

### One DC and Multiple Racks with Zone-Awareness
```yaml
apiVersion: cassandra/v1alpha1
kind: Cluster
metadata:
  name: one-dc-zone-aware-racks
spec:
  name: one-dc-zone-aware-racks
  datacenters:
    - name: dc1
      nodesPerRack: 3
      racks:
        - name: rack1
          labels:
            topology.kubernetes.io/zone: us-east-1c
        - name: rack2
          labels:
            topology.kubernetes.io/zone: us-east-1b
        - name: rack3
          labels:
            topology.kubernetes.io/zone: us-east-1d
```

### Two DCs and Multiple Racks with Zone-Awareness

```yaml
apiVersion: cassandra/v1alpha1
kind: Cluster
metadata:
  name: two-dcs-zone-aware-racks
spec:
  name: two-dcs-zone-aware-racks
  datacenters:
    - name: dc1
      nodesPerRack: 3
      racks:
        - name: rack1
          labels:
            topology.kubernetes.io/zone: us-east-1c
        - name: rack2
          labels:
            topology.kubernetes.io/zone: us-east-1b
        - name: rack3
          labels:
            topology.kubernetes.io/zone: us-east-1d
    - name: dc2
      nodesPerRack: 3
      racks:
        - name: rack1
          labels:
            topology.kubernetes.io/zone: us-west-1c
        - name: rack2
          labels:
            topology.kubernetes.io/zone: us-west-1b
        - name: rack3
          labels:
            topology.kubernetes.io/zone: us-west-1d                        
```

# Controllers
There will be two controllers - one for `Datacenters` and one for `Clusters`. I will refer to them as `datacenter_controller` and `cluster_controller` respectively.

`datacenter_controller` will watch for changes to `Datacenter` resources. Creation of a `Datacenter`.

`cluster_controller` will watch for changes to both `Cluster` and `Datacenter` resources. When a `Cluster` resource is created, `cluster_controller` should create `Datacenter` resources for any datacenters defined inline. `Datacenters` should be created in the order in which they are declared. `cluster_controller` should follow the prescribed process for adding a datacenter to the cluster. For example, after the nodes in the new DC are online, `cluster_controller` should trigger a `rebuild` (i.e., `nodetool rebuild`).

There is more involved with adding a datacenter, like updating replication of keyspaces so that the new DC starts receiving client requests. There is also the issue of updating clients. These topics can be addressed later.

If `cluster_controller` is notified of a new `Datacenter` and there is no parent `Cluster`, it should go ahead and create the `Cluster` resource.