# Storage Design Doc
This document focuses on two categories of storage for Cassandra, data and logs. Data refers to `/var/lib/cassandra`. Logs refers to `/var/log/Cassandra` but more generally any sort of log files or output.

We primarily need to configure storage for Cassandra's data directory and for the commit log. We may also want to configure storage for any of the following:

* Cassandra logs
* Cassandra debug logs
* GC logs
* Heap dumps
* Profiler
* Sidecars

## Background
Some familiarity with the following topics is recommended for this document:

* [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
* [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
* [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
* [Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/)

## Data 
By default Cassandra has the following in `/var/lib/cassandra`:

* `/var/lib/cassandra/commitlog`
* `/var/lib/cassandra/data`
* `/var/lib/cassandra/cdc_raw`
* `/var/lib/cassandra/saved_caches`
* `/var/lib/cassandra/hints`

## Logs
By default Cassandra has the following in `/var/log/cassandra`:

* `system.log`
* `debug.log`
* `gc.log`

There are other files that can included in the logs category such as:

* heap dumps
* profiler output
* thread dumps
* sidecar logs

# Current State of the World
This section briefly looks at how what is exposed with Cass Operator and with CassKop.

## Cass Operator
Cass Operator directly exposes a `PersistentVolumeClaimSpec` to configure a volume for Cassandra's data directory:

```yaml
apiVersion: cassandra.datastax.com/v1beta1
kind: CassandraDatacenter
metadata:
  name: dc1
spec:
  ...
  storageConfig:
    cassandraDataVolumeClaimSpec:
      storageClassName: server-storage
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 5Gi
```

## CassKop
CassKop on the other hand only exposes capacity and the storage class for configuring the data directory volume:

```yaml
apiVersion: "db.orange.com/v1alpha1"
kind: "CassandraCluster"
metadata:
  name: cassandra-demo
spec:
  ...
  dataCapacity: "200Mi"
  dataStorageClass: "local-storage"
```

CassKop also supports configuring additional volumes. In this example below, we see two volume mounts and two sidecars for exposing GC logs and Cassandra's logs.

**Note:** Cass Operator also uses a sidecar to expose Cassandra logs. It is deployed automatically and is not configurable in the CRD currently.

```yaml
apiVersion: "db.orange.com/v1alpha1"
kind: "CassandraCluster"
metadata:
  name: cassandra-demo
spec:
  ...
  storageConfigs:
  - mountPath: "/var/lib/cassandra/log"
    name: "gc-logs"
    pvcSpec:
      accessModes:
        - ReadWriteOnce
      storageClassName: standard-wait
      resources:
        requests:
          storage: 10Gi
  - mountPath: "/var/log/cassandra"
    name: "cassandra-logs"
    pvcSpec:
      accessModes:
        - ReadWriteOnce
      storageClassName: standard-wait
      resources:
        requests:
          storage: 10Gi
  sidecarConfigs:
    - args: ["tail", "-F", "/var/log/cassandra/system.log"]
      image: alpine
      imagePullPolicy: Always
      name: cassandra-logs
      resources: &sidecar_resources
        limits:
          cpu: 50m
          memory: 50Mi
        requests:
          cpu: 10m
          memory: 10Mi
      volumeMounts:
        - mountPath: /var/log/cassandra
          name: cassandra-logs
    - args: ["tail", "-F", "/var/log/cassandra/gc.log.0.current"]
      image: alpine
      imagePullPolicy: Always
      name: gc-logs
      <<: *sidecar_resources
      volumeMounts:
        - mountPath: /var/log/cassandra
          name: gc-logs

```

# Goals
## Data
Existing operators create a PersistentVolumeClaim (PVC) for the data directory, i.e., `/var/lib/cassandra`. This is a reasonable default. The CRD should allow for separate PVCs to be specified and created for the different data directories. Let's consider what needs to be done today to use a separate PV for the commit log. I need to do the following:

* Update the value of `commitlog_directory` in `cassandra.yaml` (however that is done for the particular operator we choose to use)
* Create a PVC
* Create a volume mount in the Cassandra container for `commitlog_directory`

We would need to repeat similar steps if we want a separate PV for any of the other subdirectories in `/var/lib/cassandra`. Ideally, the CRD and operator should abstract away these steps to the extent possible.

## Logs
A robust logging solution would require cluster-level logging with something like [Elasticsearch and Kibana](https://kubernetes.io/docs/tasks/debug-application-cluster/logging-elasticsearch-kibana/); however, that is beyond the current scope and therefore not further discussed in this document.

Logs should be accessible using standard tools, namely `kubectl`. This does however does not imply that there *has* to be a logging sidecar for each log file, but rather the CRD and operator should provide flexibility to confire what logging is exposed. Furthermore, the CRD should provide flexibility to use different VolumeSources. Consider a scenario in which we need to capture a heap dump to troubleshoot a Cassandra node. We may want to store the heap dumps on a persistent volume if the node is unstable.

# Proposal
```go
// A PersistentData object will result in the creation of a 
// PersistentVolumeClaim in the cassandra pod and a VolumeMount
// in the cassandra container.
type PersistentData struct {
	// The volume name
	Name string `json:"name"`
	
	// The path within the container where the volume
	// should be mounted.
	MountPath string `json:"mountPath"`
	
	// The requested storage capacity. This will be used in the
	// storage requirements in the generated PersistenVolumeClaim.
	Capacity string `json:"capacity,omitempty"`
	
	// The storage class to use for the volume. This will be used in
	// the generated PersistentVolumeClaim.
	StorageClass string `json:"storageClass,omitempty"`
	
	// An option volume claim template that will override default one.
	// This will also override the Capacity and StorageClass fields 
	// if they are set.
	VolumeClaimTemplate *corev1.PersistentVolumeClaimTemplate
}

// A LogData will result in the creation of a Volume in the cassandra
// pod, a VolumeMount in the cassandra container, and optionally a logging
// sidecar.
type LogData struct {
	// When true the operator will deploy a logging sidecar 
	// similar to the example above. 
	SidecarEnabled bool `json:"sidecarEnabled,omitEmpty"`
	
	// The volume mount name.
	Name string `json:"name,omitempty"`
	
	// The path within the container where the volume
	// should be mounted.
	MountPath string `json:"mountPath,omitempty"`
	
	// This allows you to mount log files in a different
	// volume which could be persistent or simply to make 
	// available to another container for processing.
	VolumeSource *corev1.VolumeSource `json:"volumeSource,omitempty"`
}

typs StorageConfig struct {
	// The default data volume, i.e., /var/lib/cassandra.
	CassandraDataDir *PersistentData `json:"cassandraDataDir,omitempty"`
	
	// An optional, persistent volume for the data directory,
	// i.e., SSTables. The operator will update the value of
	// data_file_directories in cassandra.yaml to 
	// PersistentData.MountPath. Note that this does not currently
	// support multiple data directories.
	DataDir *PersistentData `json:"dataDir,omitempty"`
	
	// An optional, persistent volume for the commit log. The
	// operator will update the value of commitlog_directory 
	// in cassandra.yaml to PersistentData.MountPath. 
	CommitLogDir *PersistentData `json:"commitLogDir,omitempty"`
	
	// An optional, persistent volume for the hints directory.
	// The operator will update the value of hints_directory
	// in cassandra.yaml to PersistentData.MountPath.
	HintsDir *PersistentData `json:"hintsDir,omitempty"`
	
	// An optional, persistent volume for the saved_caches 
	// directory. The operator will update the value of
	// saved_caches_directory in cassandra.yaml to 
	// PersistentData.MountPath.
	SavedCachesDir *PersistentData `json:"savedCachesDir,omitempty"`
	
	// An optional, persistent volume for the cdc_raw directory.
	// The operator will update the value of cdc_raw_directory in
	// cassandra.yaml to PersistentData.MountPath.
	CdcRawDir *PersistentData `json:"cdcRawDir,omitempty"`
	
	// The default log directory, i.e., /var/log/cassandra
	CassandraLogDir *LogData `json:"cassandraLogDir,omitempty"`
	
	// An optional volume for the system.log files. If this property
	// is set, the operator should update logback.xml accordingly.
	SystemLogDir *LogData `json:"systemLogDir,omitempty"`
	
	// An optional volume for the debug.log files. If this property
	// is set, the operator should update logback.xml accordingly.
	DebugLogDir *LogData `json:"debugLogDir,omitempty"`
	
	// An optional volume for heap dumps. If this property is set,
	// then the operator should set -xx:HeapDumpPath in jvm.options
	// accordingly.
	HeapDumpDir *LogData `json:"heapDumpDir,omitempy"`
	
	AdditionalVolumes []corev1.Volume `json:"additionalVolumes,omitempty"`
}
```

**Note:** None of the properties in `StorageConfig` have to be set.

## PersistentData
The operator will by default initialize `CassandraDataDir`. This will result in the creation of a PersistenVolumeClaim for `/var/lib/cassandra` and a VolumeMount in the cassandra container.

Let's say we have enabled CDC in our Cassandra cluster. We can set the `CdcRawDir` property so that commit log segments that are flushed to `/var/lib/cassandra/cdc_raw` instead get stored on a separate volume. It might look something like this:

```go
StorageConfig{
	CdcRawDir: &PersistentData{
		Name: "cdc_raw"
		MountPath: "/cdc_raw"
		Capacity: "10Gi"
		StorageClass: fast-storage
	}
}
```

In addition to creating the default PersistentData for `CassandraDataDir`, the operator will also create one for `CdcRawDir`. The operator will create a 10 Gi PersistentVolumeClaim with a mount point at `/cdc_raw`. The operator should also update the value of `cdc_raw_directory` in `cassandra.yaml`.

## LogData
The operator will by default initialize `CassandraLogDir`. This will result in the creation of a Volume for `/var/log/cassandra` and a VolumeMount in the cassandra container. 

The operator should also by default initialize `SystemLogDir` such that it has a logging sidecar but no additional volume.

Now let's say we to send debug logs to a different volume for processing by another container. We could set `DebugLogDir` as follows:

```go
StorageConfig{
	DebugLogDir:
		Name: "cassandra-debug-log"
		MountPath: "/cassandra-debug-log"
		VolumeSource: corev1.VolumeSource{
			EmptyDir: &corev1.EmptyDirVolumeSource{}
		}
}
```

The operator should create an EmptyDir volume with a mount point of `/cassandra-debug-log` in the cassandra container. The operator should update `logback.xml` so that debug logs are written to `/cassandra-debug-log` instead of `/var/log/cassandra`.

## AdditionalVolumes
We should provide support for specifying additional volumes that may needed for sidecars.

Rather than a slice of raw volumes, would it make more sense to have something like this:

```go
AdditionalPersistentData []PersistentData `json:"additionalPersistentData,omitempty"`

AdditionalLogData []LogData `json:"additionalLogData,omitempty"`
```