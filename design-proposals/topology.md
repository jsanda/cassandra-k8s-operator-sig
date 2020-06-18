# Topology Design Doc
The purpose of this document is to identify and define data types that are needed for specifying a Cassandra cluster's topology.

# Racks
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

Some notable things missing from the `Rack` type include:

* Cassandra configuration, i.e., `cassandra.yaml`
* Number of C* nodes
* Configuration of the `PodTemplateSpec`

**Cassandra configuration**

The purpose of a `Rack` is help specify topology and where pods should be scheduled. Cassandra configuration will be addressed elsewhere in the CRD.

**Number of nodes**

We want balanced racks. Instead of specifying nodes at the rack level, it should be a setting that applies across racks to help enforce balance.

**Configuration of PodTemplateSpec**

Should any of this be exposed at the rack level?

**Affinity**

Should `Affinity` be exposed to give users more fine-grained control over node affinity, pod affinity, and pod anti-affinity in cases where only labels is not expressive enough?

# Datacenter
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
}
```

# Cluster
```go
type DatacenterRef corev1.ObjectReference

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
	
	Datacenters []Datacenter `json:"datacenters,omitempty"`
	
	DatacenterRefs []DatacenterRef `json:"datacenterRefs,omitempty"`
}
```

And here is example of how this might look in a YAML manifest:

```yaml
apiVersion: cassandra/v1alpha1
kind: Cluster
metadata:
  name: demo
spec:
  name: demo
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
  datacenterRefs:
    - name: dc3
    - name: dc4          
```

A `Cluster` allows for datacenters to be defined inline or as references. As the example illustrates, a combination of the two can be used. 

If the same datacenter appears in both `datacenters` and `datacenterRefs`, then it should result in a validation error.

# Controllers
There will be two controllers - one for `Datacenters` and one for `Clusters`. I will refer to them as `datacenter_controller` and `cluster_controller` respectively.

`datacenter_controller` will watch for changes to `Datacenter` resources. Creation of a `Datacenter`.

`cluster_controller` will watch for changes to both `Cluster` and `Datacenter` resources. When a `Cluster` resource is created, `cluster_controller` should create `Datacenter` resources for any datacenters defined inline. `Datacenters` should be created in the order in which they are declared. `cluster_controller` should follow the prescribed process for adding a datacenter to the cluster. For example, after the nodes in the new DC are online, `cluster_controller` should trigger a `rebuild` (i.e., `nodetool rebuild`).

There is more involved with adding a datacenter, like updating replication of keyspaces so that the new DC starts receiving client requests. There is also the issue of updating clients. These topics can be addressed later.

If `cluster_controller` is notified of a new `Datacenter` and there is no parent `Cluster`, it should go ahead and create the `Cluster` resource.