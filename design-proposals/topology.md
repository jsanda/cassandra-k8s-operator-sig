# Topology Design Doc
The purpose of this document is to identify and define data types that are needed for specifying a Cassandra cluster's topology.

# Racks
All four operator have a well-defined type for racks. A `Rack` object should provide the ability to do the following for the Cassandra nodes that constitute a rack:

* Constrain C* pods to be scheduled to a specific zone
* Constrain C* pods to particular k8s worker nodes
* Constrain C* pods to particular k8s worker nodes based on the pods already running on those nodes
* Prevent C* pods from being scheduled on k8s worker nodes that do  meet certain criteria, e.g., the node does not have local SSDs.

All of these things can be accomplished using node affinity, pod anti-affinity, and taints and tolerations.`

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