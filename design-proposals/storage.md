# Storage Design Doc
The purpose of this document is to identify and define data types to satisfy Cassandra storage requirements.

We primarily need to configure storage for Cassandra's data directory and for the commit log. We may also want to configure storage for any of the following:

* Cassandra logs
* Cassandra debug logs
* GC logs
* Heap dumps
* Profiler
* Sidecars

## Data Directory
By default Cassandra stores the following in the *data* directory:

* `/var/lib/cassandra/commitlog`
* `/var/lib/cassandra/data`
* `/var/lib/cassandra/cdc_raw`
* `/var/lib/cassandra/saved_caches`
* `/var/lib/cassandra/hints`

## Logs Directory
By default Cassandra stores the following the *logs* directory:

* `system.log`
* `debug.log`
* `gc.log`

# Current State of the World
This section briefly looks at how what is exposed with Cass Operator and with CassKop.

## Cass Operator
Cass Operator directly exposes a `PersistentVolumeClaimSpec` to configure a volume for Cassandra's data directory and commit log directory:

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
CassKop on the other hand only exposes capacity and the storage class for configuring the volume that will store Cassandra's data directoty and commit log directory:

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

CassKop also supports configuring additional volumes:

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

