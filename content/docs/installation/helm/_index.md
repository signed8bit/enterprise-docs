---
title: "Install with Helm on Kubernetes"
linkTitle: "Kubernetes"
weight: 3
---

## Introduction

The preferred method for deploying Anchore Enterprise on Kubernetes is with [Helm](https://helm.sh). The [Anchore Engine Helm Chart](https://github.com/helm/charts/tree/master/stable/anchore-engine) now includes configuration options for a full Enterprise deployment. This deploys an Anchore Engine 0.3.2 system as well as the enterprise extensions and services.

This chart deploys the Anchore Engine docker container image analysis system. Anchore Engine requires a PostgreSQL database (>=9.6) which may be handled by the chart or supplied externally, and executes in a service based architecture utilizing the following Anchore Engine services: External API, Simplequeue, Catalog, Policy Engine, and Analyzer.

This chart can also be used to install the following Anchore Enterprise services: GUI, RBAC, On-prem Feeds. Enterprise services require a valid Anchore Enterprise License as well as credentials with access to the private dockerhub repository hosting the images. These are not enabled by default.

Each of these services can be scaled and configured independently.

### Chart Details

The chart is split into global and service specific configurations for the OSS Anchore Engine, as well as global and services specific configurations for the Enterprise components.

  * The `anchoreGlobal` section is for configuration values required by all Anchore Engine components.
  * The `anchoreEnterpriseGlobal` section is for configuration values required by all Anchore Engine Enterprise components.
  * Service specific configuration values allow customization for each individual service.

For a description of each component, view the official documentation at: [Anchore Enterprise Service Overview](../../overview/architecture)

### Installation Steps

Enterprise services require an Anchore Enterprise license, as well as credentials with
permission to the private docker repositories that contain the enterprise images.

To use this Helm chart with the enterprise services enabled, perform these steps.

1. Create a kubernetes secret containing your license file.

    `kubectl create secret generic anchore-enterprise-license --from-file=license.yaml=<PATH/TO/LICENSE.YAML>`

1. Create a kubernetes secret containing dockerhub credentials with access to the private anchore enterprise repositories.

    `kubectl create secret docker-registry anchore-enterprise-pullcreds --docker-server=docker.io --docker-username=<DOCKERHUB_USER> --docker-password=<DOCKERHUB_PASSWORD> --docker-email=<EMAIL_ADDRESS>`

1. Install the helm chart using a custom anchore_values.yaml file (see examples below)

    `helm install --name <release_name> -f /path/to/anchore_values.yaml stable/anchore-engine`

#### Example anchore_values.yaml file for installing Anchore Enterprise
*Note: This installs with chart managed PostgreSQL & Redis databases. This is not a production ready config.*

  ```
  ## anchore_values.yaml

  postgresql:
    postgresPassword: <PASSWORD>
    persistence:
      size: 50Gi

  anchoreGlobal:
    defaultAdminPassword: <PASSWORD>
    defaultAdminEmail: <EMAIL>
    enableMetrics: True

  anchoreEnterpriseGlobal:
    enabled: True

  anchore-feeds-db:
    postgresPassword: <PASSWORD>

  anchore-ui-redis:
    password: <PASSWORD>
  ```

### Configuration

All configurations should be appended to your custom `anchore_values.yaml` file and utilized when installing the chart. While the configuration options of Anchore Engine are extensive, the options provided by the chart are:


#### Install using chart managed PostgreSQL service with custom passwords.
  ```
  ## anchore_values.yaml

  postgresql:
    postgresPassword: <PASSWORD>
    persistence:
      size: 50Gi

  anchoreGlobal:
    defaultAdminPassword: <PASSWORD>
    defaultAdminEmail: <EMAIL>
  ```

#### Exposing the service outside the cluster:

##### Using Ingress

This configuration allows SSL termination using your chosen ingress controller.

###### NGINX Ingress Controller
```
ingress:
  enabled: true
```

###### GCE Ingress Controller
  ```
  ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: gce
    apiPath: /v1/*
    uiPath: /*
    apiHosts:
      - anchore-api.example.com
    uiHosts:
      - anchore-ui.example.com

  anchoreApi:
    service:
      type: NodePort

  anchoreEnterpriseUi:
    service
      type: NodePort
  ```

###### Using Service Type
  ```
  anchoreApi:
    service:
      type: LoadBalancer
  ```

#### Install using an existing/external PostgreSQL instance
*Note: it is recommended to use an external Postgresql instance for production installs*

  ```
  postgresql:
    postgresPassword: <PASSWORD>
    postgresUser: <USER>
    postgresDatabase: <DATABASE>
    enabled: false
    externalEndpoint: <HOSTNAME:5432>

  anchoreGlobal:
    dbConfig:
      ssl: true
  ```

#### Archive Driver
*Note: it is recommended to use an external archive driver for production installs.*

The archive subsystem of Anchore Engine is what stores large json documents and can consume quite a lot of storage if
you analyze a lot of images. A general rule for storage provisioning is 10MB per image analyzed, so with thousands of
analyzed images, you may need many gigabytes of storage. The Archive drivers now support other backends than just postgresql,
so you can leverage external and scalable storage systems and keep the postgresql storage usage to a much lower level.

##### Configuring Compression:

The archive system has compression available to help reduce size of objects and storage consumed in exchange for slightly
slower performance and more cpu usage. There are two config values:

To toggle on/off (default is True), and set a minimum size for compression to be used (to avoid compressing things too small to be of much benefit, the default is 100):

  ```
  anchoreCatalog:
    archive:
      compression:
        enabled=True
        min_size_kbytes=100
  ```

##### The supported archive drivers are:

* S3 - Any AWS s3-api compatible system (e.g. minio, scality, etc)
* OpenStack Swift
* Local FS - A local filesystem on the core pod. Does not handle sharding or replication, so generally only for testing.
* DB - the default postgresql backend

#### S3:
  ```
  anchoreCatalog:
    archive:
      storage_driver:
        name: 's3'
        config:
          access_key: 'MY_ACCESS_KEY'
          secret_key: 'MY_SECRET_KEY'
          #iamauto: True
          url: 'https://S3-end-point.example.com'
          region: null
          bucket: 'anchorearchive'
          create_bucket: True
      compression:
      ... # Compression config here
  ```

#### Using Swift:

The swift configuration is basically a pass-thru to the underlying pythonswiftclient so it can take quite a few different
options depending on your swift deployment and config. The best way to configure the swift driver is by using a custom values.yaml

The Swift driver supports three authentication methods:

* Keystone V3
* Keystone V2
* Legacy (username / password)

##### Keystone V3:
  ```
  anchoreCatalog:
    archive:
      storage_driver:
        name: swift
        config:
          auth_version: '3'
          os_username: 'myusername'
          os_password: 'mypassword'
          os_project_name: myproject
          os_project_domain_name: example.com
          os_auth_url: 'foo.example.com:8000/auth/etc'
          container: 'anchorearchive'
          # Optionally
          create_container: True
      compression:
      ... # Compression config here
  ```

##### Keystone V2:
  ```
  anchoreCatalog:
    archive:
      storage_driver:    
        name: swift
        config:
          auth_version: '2'
          os_username: 'myusername'
          os_password: 'mypassword'
          os_tenant_name: 'mytenant'
          os_auth_url: 'foo.example.com:8000/auth/etc'
          container: 'anchorearchive'
          # Optionally
          create_container: True
      compression:
      ... # Compression config here
  ```

##### Legacy username/password:
  ```
  anchoreCatalog:
    archive:
      storage_driver:
        name: swift
        config:
          user: 'user:password'
          auth: 'http://swift.example.com:8080/auth/v1.0'
          key:  'anchore'
          container: 'anchorearchive'
          # Optionally
          create_container: True
      compression:
      ... # Compression config here
  ```

#### Postgresql:

This is the default archive driver and requires no additional configuration.

### Prometheus Metrics

Anchore Engine supports exporting prometheus metrics form each container. To enable metrics:
  ```
  anchoreGlobal:
    enableMetrics: True
  ```

When enabled, each service provides the metrics over the existing service port so your prometheus deployment will need to
know about each pod and the ports it provides to scrape the metrics.

### Event Notifications

Anchore Engine in v0.2.3 introduces a new events subsystem that exposes system-wide events via both a REST api as well
as via webhooks. The webhooks support filtering to ensure only certain event classes result in webhook calls to help limit
the volume of calls if you desire. Events, and all webhooks, are emitted from the core components, so configuration is
done in the coreConfig.

To configure the events:
  ```
  anchoreCatalog:
    events:
      notification:
        enabled:true
      level=error
  ```

### Scaling Individual Components

As of Chart version 0.9.0, all services can now be scaled-out by increasing the replica counts. The chart now supports
this configuration.

To set a specific number of service containers:
  ```
  anchoreAnalyzer:
    replicaCount: 5

  anchorePolicyEngine:
    replicaCount: 3
  ```

To update the number in a running configuration:

`helm upgrade --set anchoreAnalyzer.replicaCount=2 <releasename> stable/anchore-engine -f anchore_values.yaml`

### Next Steps

Now that you have Anchore Enterprise running, you can begin to learning more about Anchore Enterprise Architecture, Anchore Concepts and Anchore Usage.

- To learn more about Anchore Enterprise, go to [Overview]({{< ref "/docs/overview" >}})
- To learn more about Anchore Concepts, go to [Concepts]({{< ref "/docs/overview/concepts" >}})
- To learn more about using Anchore Usage, go to [Usage]({{< ref "/docs/using" >}})