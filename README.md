# Bookkeeper Operator

 [![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0) [![GoDoc](https://godoc.org/github.com/pravega/bookkeeper-operator?status.svg)](https://godoc.org/github.com/pravega/bookkeeper-operator) [![Build Status](https://travis-ci.org/pravega/bookkeeper-operator.svg?branch=master)](https://travis-ci.org/pravega/bookkeeper-operator) [![Go Report](https://goreportcard.com/badge/github.com/pravega/bookkeeper-operator)](https://goreportcard.com/report/github.com/pravega/bookkeeper-operator) [![Version](https://img.shields.io/github/release/pravega/bookkeeper-operator.svg)](https://github.com/pravega/bookkeeper-operator/releases)

### Project status: alpha

The project is currently alpha. While no breaking API changes are currently planned, we reserve the right to address bugs and change the API before the project is declared stable.

## Table of Contents

 * [Overview](#overview)
 * [Requirements](#requirements)
 * [Quickstart](#quickstart)    
    * [Install the Operator](#install-the-operator)
        * [Install the Operator in Test Mode](#install-the-operator-in-test-mode)
    * [Install a sample Bookkeeper Cluster](#install-a-sample-bookkeeper-cluster)
    * [Scale a Bookkeeper Cluster](#scale-a-bookkeeper-cluster)
    * [Upgrade a Bookkeeper Cluster](#upgrade-a-bookkeeper-cluster)
    * [Uninstall the Bookkeeper Cluster](#uninstall-the-bookkeeper-cluster)
    * [Uninstall the Operator](#uninstall-the-operator)
    * [Manual installation](#manual-installation)
 * [Configuration](#configuration)
 * [Development](#development)
* [Releases](#releases)
* [Upgrade the Operator](#upgrade-the-operator)

## Overview

[Bookkeeper](https://bookkeeper.apache.org/) A scalable, fault-tolerant, and low-latency storage service optimized for real-time workloads.

The Bookkeeper Operator manages Bookkeeper clusters deployed to Kubernetes and automates tasks related to operating a Bookkeeper cluster.

- [x] Create and destroy a Bookkeeper cluster
- [x] Resize cluster
- [x] Rolling upgrades

## Requirements

- Kubernetes 1.9+
- Helm 2.10+
- An existing Apache Zookeeper 3.5 cluster. This can be easily deployed using our [Zookeeper operator](https://github.com/pravega/zookeeper-operator)

## Quickstart

### Install the Operator

> Note: If you are running on Google Kubernetes Engine (GKE), please [check this first](doc/development.md#installation-on-google-kubernetes-engine).

Use Helm to quickly deploy a Bookkeeper operator with the release name `pravega-bk`.

```
$ helm install charts/bookkeeper-operator --name pr
```

Verify that the Bookkeeper Operator is running.

```
$ kubectl get deploy
NAME                          DESIRED   CURRENT   UP-TO-DATE   AVAILABLE     AGE
pr-bookkeeper-operator           1         1         1            1          17s
```

#### Install the Operator in Test Mode
 The Operator can be run in `test mode` if we want to deploy the Bookkeeper Cluster on minikube or on a cluster with very limited resources by setting `testmode: true` in `values.yaml` file. Operator running in test mode skips the minimum replica requirement checks. Test mode provides a bare minimum setup and is not recommended to be used in production environments.

### Install a sample Bookkeeper cluster
> Note that the Bookkeeper cluster must be installed in the same namespace as the Zookeeper cluster.

If the Bookkeeper cluster is expected to work with Pravega, we need to create a ConfigMap which needs to have the following values

| KEY | VALUE |
|---|---|
| *PRAVEGA_CLUSTER_NAME* | Name of Pravega Cluster using this BookKeeper Cluster |
| *WAIT_FOR* | Zookeeper URL |

The name of this ConfigMap needs to be mentioned in the field `envVars` present in the BookKeeper Spec. For more details about this ConfigMap refer to [this](doc/bookkeeper-options.md#bookkeeper-custom-configuration).

Helm can be used to install a sample Bookkeeper cluster.

```
$ helm install charts/bookkeeper --name pravega-bk --set zookeeperUri=[ZOOKEEPER_HOST]
```

where:

- `[ZOOKEEPER_HOST]` is the Zookeeper service endpoint of your Zookeeper deployment (e.g. `zookeeper-client:2181`). It expects the zookeeper service URL in the given format `<service-name>:<port-number>`

Check out the [Bookkeeper Helm Chart](charts/bookkeeper) for more a complete list of installation parameters.

Verify that the cluster instances and its components are being created.

```
$ kubectl get bk
NAME                   VERSION   DESIRED MEMBERS   READY MEMBERS      AGE
pravega-bk             0.7.0     3                 1                  25s
```

After a couple of minutes, all cluster members should become ready.

```
$ kubectl get bk
NAME                   VERSION   DESIRED MEMBERS   READY MEMBERS     AGE
pravega-bk             0.7.0     3                 3                 2m
```

```
$ kubectl get all -l bookkeeper_cluster=pravega-bk
NAME                                              READY   STATUS    RESTARTS   AGE
pod/pravega-bk-bookie-0                           1/1     Running   0          2m
pod/pravega-bk-bookie-1                           1/1     Running   0          2m
pod/pravega-bk-bookie-2                           1/1     Running   0          2m

NAME                                            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)              AGE
service/pravega-bk-bookie-headless              ClusterIP   None          <none>        3181/TCP             2m

NAME                                            DESIRED   CURRENT     AGE
statefulset.apps/pravega-bk-bookie              3         3           2m
```

By default, a `BookkeeperCluster` is reachable using this kind of headless service URL for each pod:
```
http://pravega-bk-bookie-0.pravega-bk-bookie-headless.pravega-bk-bookie:3181
```

### Scale a Bookkeeper cluster

You can scale Bookkeeper cluster by updating the `replicas` field in the BookkeeperCluster Spec.

Example of patching the Bookkeeper Cluster resource to scale the server instances to 4.

```
kubectl patch bk pravega-bk --type='json' -p='[{"op": "replace", "path": "/spec/replicas", "value": 4}]'
```

### Upgrade a Bookkeeper cluster

Check out the [upgrade guide](doc/upgrade-cluster.md).

### Uninstall the Bookkeeper cluster

```
$ helm delete pravega-bk --purge
```

Once the Bookkeeper cluster has been deleted, make sure to check that the zookeeper metadata has been cleaned up before proceeding with the deletion of the operator. This can be confirmed with the presence of the following log message in the operator logs.
```
zookeeper metadata deleted
```

However, if the operator fails to delete this metadata from zookeeper, you will instead find the following log message in the operator logs.
```
failed to cleanup <bookkeeper-cluster-name> metadata from zookeeper (znode path: /pravega/<pravega-cluster-name>): <error-msg>
```
The operator additionally sends out a `ZKMETA_CLEANUP_ERROR` event to notify the user about this failure. The user can check this event by doing `kubectl get events`. The following is the sample describe output of the event that is generated by the operator in such a case

```
Name:             ZKMETA_CLEANUP_ERROR-nn6sd
Namespace:        default
Labels:           app=bookkeeper-cluster
                  bookkeeper_cluster=pravega-bk
Annotations:      <none>
API Version:      v1
Event Time:       <nil>
First Timestamp:  2020-04-27T16:53:34Z
Involved Object:
  API Version:   app.k8s.io/v1beta1
  Kind:          Application
  Name:          bookkeeper-cluster
  Namespace:     default
Kind:            Event
Last Timestamp:  2020-04-27T16:53:34Z
Message:         failed to cleanup pravega-bk metadata from zookeeper (znode path: /pravega/pravega): failed to delete zookeeper znodes for (pravega-bk): failed to connect to zookeeper: lookup zookeeper-client on 10.100.200.2:53: no such host
Metadata:
  Creation Timestamp:  2020-04-27T16:53:34Z
  Generate Name:       ZKMETA_CLEANUP_ERROR-
  Resource Version:    864342
  Self Link:           /api/v1/namespaces/default/events/ZKMETA_CLEANUP_ERROR-nn6sd
  UID:                 5b4c3f80-36b5-43e6-b417-7992bc309218
Reason:                ZK Metadata Cleanup Failed
Reporting Component:   bookkeeper-operator
Reporting Instance:    bookkeeper-operator-6769886978-xsjx6
Source:
Type:    Error
Events:  <none>
```

>In case the operator fails to delete the zookeeper metadata, the user is expected to manually delete the metadata from zookeeper prior to reinstall.

### Uninstall the Operator
> Note that the Bookkeeper clusters managed by the Bookkeeper operator will NOT be deleted even if the operator is uninstalled.

```
$ helm delete pr --purge
```

### Manual installation

You can also manually install/uninstall the operator and Bookkeeper with `kubectl` commands. Check out the [manual installation](doc/manual-installation.md) document for instructions.

## Configuration

Check out the [configuration document](doc/configuration.md).

## Development

Check out the [development guide](doc/development.md).

## Releases  

The latest Bookkeeper releases can be found on the [Github Release](https://github.com/pravega/bookkeeper-operator/releases) project page.

## Upgrade the Bookkeeper Operator
Bookkeeper operator can be upgraded by modifying the image tag using
```
$ kubectl edit <operator deployment name>
```
