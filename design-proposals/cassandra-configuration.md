# Cassandra Configuration Design Doc
Cassandra has several configuration files including:

* cassandra.yaml
* jvm.options
* cassandra-rackdc.properties
* cassandra-env.sh
* logback.xml
* cassandra-topology.properties
* commitlog_archiving.properties
* cassandra-jaas.config

## Background
Some familiarty with the following topics is recommended for this document:

* [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
* [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
* [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)

# Configuration Files
## cassandra.yaml
The main configuration file for Cassandra.

## jvm.options
jvm.options provides options for JVM settings like heap and garbage collection. It also allows you to override settings in cassandra.yaml.

## logback.xml
Configures Cassandra's logging.

## cassandra-rackdc.properties
Used by GossipingPropertyFileSnitch to specify racks and DCs. This file should not be directly exposed to users. This file should be managed by the operator and updated based on topology settings in the spec.

## cassandra-topology.properties
Set rack and datacenter info for the C* cluster. This file should be managed by the operator and updated based on the topology settings in the spec.

## cassandra-env.sh

A startup script that configures different environment variables.

## commitlog_archiving.properties
Sets archiving properties for the commit log.

## cassandra-jaas.config
**TODO**

# Goals
StatefulSets do not provide a way for setting up and managing config files like they do for persistent volumes. Users consequently have to get creative and implement custom solutions for setting up configuration files across pods in a StatefulSet.

We want to have the flexibility of directly editing config files as necessary while at the same time provide abstractions to make things easier and to encourage best practices.

Here is an example of how Cass Operator does it:

```yaml
apiVersion: cassandra.datastax.com/v1beta1
kind: CassandraDatacenter
metadata:
  name: dc1
# ...
spec:
# ...
  # Everything under the config key is passed to the cass-config-builder init
  # container that runs on pod creation, and then marshalled into config files.
  config:
    cassandra-yaml:
      num_tokens: 8    
      file_cache_size_in_mb: 1000
      authenticator: org.apache.cassandra.auth.PasswordAuthenticator
      authorizer: org.apache.cassandra.auth.CassandraAuthorizer
      role_manager: org.apache.cassandra.auth.CassandraRoleManager

    jvm-options:
      # Set the database to use 14 GB of Java heap
      initial_heap_size: "14G"
      max_heap_size: "14G"

      additional-jvm-opts:
        # As the database comes up for the first time, set system keyspaces to RF=3
        - "-Dcassandra.system_distributed_replication_dc_names=dc1"
        - "-Dcassandra.system_distributed_replication_per_dc=3"
```

## Questions
### Should all settings cassandra.yaml be exposed in `config` or should some be restricted?
For example, should users be allowed to change port settings like `storage_port` or `native_transport_port` here?

The same questions apply to `seed_provider`.

### Are there some settings that should be made more explicit in the spec?
Would it make sense for example to have an `Auth` property in the spec where we can:

* Enable/disable authentication and authorization
* Configure auth caches, e.g., `permissions_validity_in_ms` and `permissions_update_interval_in_ms`
* Create roles and permissions