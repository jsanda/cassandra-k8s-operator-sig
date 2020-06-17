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

# Datacenter
```go
type Datacenter struct {
	Name string `json:"name"`
	
	NodesPerRack int32 `json:"nodesPerRack,omitempty`
	
	Racks []Rack `json:"racks,omitempty"`
}
```