# Federating System and User metrics to Azure Files in Azure RedHat OpenShift

**Paul Czarkowski**

*06/04/2021*

By default Azure RedHat OpenShift (ARO) stores metrics in Ephemeral volumes, and its advised that users do not change this setting. However its not unreasonable to expect that metrics should be persisted for a set amount of time.

This guide shows how to set up Thanos to federate both System and User Workload Metrics to a Thanos gateway that stores the metrics in Azure Files and makes them available via a Grafana instance (managed by the Grafana Operator).

> ToDo - Add Authorization in front of Thanos APIs

## Pre-Prequsites

1. An ARO cluster

1. clone this repo down locally

    ```bash
    git clone https://github.com/rh-mobb/documentation
    cd docs/aro/federated-metrics
    ```

## Azure Preperation

1. Create an Azure storage account

    > modify the arguments to suit your environment

    ```bash
    az storage account create \
      --name thanosreceiver \
      --resource-group openshift \
      --location eastus \
      --sku Standard_RAGRS \
      --kind StorageV2
    ```

1. Get the account key and update the secret in `thanos-store-credentials.yaml`

    ```bash
    az storage account keys list -g openshift -n thanosreceiver
    ```

1. Create the Thanos Store Credentials Secret

    ```bash
    oc new-project thanos-receiver
    oc apply -f thanos-store-credentials.yaml
    ```

## Enabling User Workload Monitoring

> See [docs](https://docs.openshift.com/container-platform/4.7/monitoring/enabling-monitoring-for-user-defined-projects.html) for more indepth details.

1. Check the cluster-monitoring-config ConfigMap object

    ```bash
    oc -n openshift-monitoring get configmap cluster-monitoring-config -o yaml
    ```

1. Enable User Workload Monitoring by doing one of the following

    **If the `data.config.yaml` is not `{}` you should edit it and add the `enableUserWorkload: true` line manually.**

    ```bash
    oc -n openshift-monitoring edit configmap cluster-monitoring-config
    ```

    **Otherwise if its `{}` then you can run the following command safely.**

    ```bash
    oc patch configmap cluster-monitoring-config -n openshift-monitoring \
       -p='{"data":{"config.yaml": "enableUserWorkload: true\n"}}'
    ```

1. Check that the User workload monitoring is starting up

    ```bash
    oc -n openshift-user-workload-monitoring get pods
    ```

## Deploy Thanos Store Gateway

<!--

> we should be able to skip this because its in the yaml now.

1. Create a service account for the Thanos Store Gateway

    ```bash
    oc -n thanos-receiver create serviceaccount thanos-store-gateway
    oc -n thanos-receiver adm policy add-scc-to-user anyuid -z thanos-store-gateway
    ```
-->

1. Deploy the thanos store

    ```bash
    oc apply -f thanos-store.yaml
    ```

1. Deploy Thanos Receiver

    > Note we should be securing this via [OIDC / Bearer Tokens](https://www.openshift.com/blog/federated-prometheus-with-thanos-receive)

    ```bash
    oc -n thanos-receiver apply -f thanos-receive.yaml
    ```

1. Append remoteWrite settings to the cluster-monitoring config to forward cluster metrics to Thanos.

    ```bash
    oc -n openshift-monitoring edit configmaps cluster-monitoring-config
    ```

    ```yaml
      data:
        config.yaml: |
          ...
          prometheusK8s:
          ...
            remoteWrite:
              - url: "http://thanos-receive.thanos-receiver.svc.cluster.local:9091/api/v1/receive"
    ```

1. Append remoteWrite settings to the user-workload-monitoring config to forward user workload metrics to Thanos.

    **Check if the User Workload Config Map exists:**

    ```bash
    oc -n openshift-user-workload-monitoring get \
      configmaps user-workload-monitoring-config
    ```

    **If the config doesn't exist run:**

    ```bash
    oc apply -f user-workload-monitoring-config.yaml
    ```

    **Otherwise update it with the following:**

    ```bash
    oc -n openshift-user-workload-monitoring edit \
      configmaps user-workload-monitoring-config
    ```

    ```yaml
      data:
        config.yaml: |
          ...
          prometheus:
          ...
            remoteWrite:
              - url: "http://thanos-receive.thanos-receiver.svc.cluster.local:9091/api/v1/receive"
    ```


## Deploy Thanos Queryier

1. Deploy the thanos querier

    ```bash
    oc apply -f thanos-querier.yaml
    ```

## Deploy Grafana

1. create the grafana operator in the thanos-receiver namespace

    ```bash
    oc apply -f thanos-grafana-operator.yaml
    ```

1. create grafana instance and datasource for thanos

    > Change the password to something less default.

    ```bash
    oc -n thanos-receiver apply -f thanos-grafana.yaml
    ```

1. load up cluster metrics dashboards

    > Note: these were generated by the [generate-dashboards.sh](generate-dashboards.sh) script.

    ```bash
    oc -n thanos-receiver apply -f dashboards.yaml
    ```

1. get the Route URL for Grafana (remember its https) and login using username `root` and the password you updated to (or the default of `secret`).

    ```bash
    oc -n thanos-receiver get route grafana-route
    ```

1. Once logged in go to **Dashboards->Manage** and expand the **thanos-receiver** group and you should see the cluster metrics dashboards.  Click on the **Use Method / Cluster** Dashboard and you should see metrics.  \o/.

![screenshot of grafana with federated cluster metrics](./grafana-metrics.png)