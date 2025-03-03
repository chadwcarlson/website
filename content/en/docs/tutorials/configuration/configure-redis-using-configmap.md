---
reviewers:
- eparis
- pmorie
title: Configure Redis using a ConfigMap
content_type: tutorial
weight: 30
---

<!-- overview -->
You can use 
{{< glossary_tooltip text="ConfigMaps" term_id="configmap" >}}
to decouple configuration artifacts from image content.
This keeps containerized applications portable by allowing you to inject configuration data from a data 
object within a file into application 
{{< glossary_tooltip text="pods" term_id="pod" >}}.

In this tutorial, you'll build upon [your previous experience configuring Pods with ConfigMaps](/docs/tasks/configure-pod-container/configure-pod-configmap/) 
to create and configure a Redis cache Pod using the data stored in a ConfigMap.

Additionally, you'll have a chance to better understand not only the relationship between the configuration defined by a ConfigMap for a Pod, 
but also what's required in order to update that configuration at runtime.

## What you'll learn

In this tutorial, you'll learn how to do the following tasks:

* Create a ConfigMap containing Redis configuration.
* Create a Redis Pod that mounts and uses the ConfigMap you create.
* Verify whether the configuration was correctly applied to a Pod from a ConfigMap.
* Update a Pod's configuration when there are changes to the ConfigMap.

## Requirements

* You have [minikube](https://minikube.sigs.k8s.io/docs/start), the local Kubernetes toolkit, [installed locally](https://minikube.sigs.k8s.io/docs/start). 
  This tutorial works with version `v1.35.0`. Check your current and the upstream's latest version with the command `minikube update-check`.
* You have [Docker Desktop](https://docs.docker.com/desktop/) installed on your computer, which you will use as the container manager for minikube. 
  This tutorial works with version 27.5.1 and above. Check your version with the command `docker version`.
* You've [installed](/docs/tasks/tools/#kubectl) the `kubectl` command-line tool to communicate with the minikube cluster.
  This tutorial works with `kubectl` version 1.14 and above. Check your version with the command `kubectl version`.

<!-- lessoncontent -->

## Step 1: Set up the cluster

Before you can define and apply configuration from a ConfigMap, you'll need to first set up the Kubernetes cluster.

1. **Start Docker Desktop**

   If you skip this step, you'll get the error `Unable to pick a default driver` on the next step.

1. **Start your cluster**

   From a terminal with administrator access (but not logged in as root), run the following command to start your cluster:

    ```shell
    minikube start
    ```

1. **Verify configuration**

   When completed, your `kubectl` CLI tool will automatically be configured to communicate with the minikube cluster.

    ```shell
    $ kubectl get services
    NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
    kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   2m35s
    ```
  
   Notice that the tool is communicating to the Cluster `kubernetes`, but that it at this point contains no pods:

    ```shell
    $ kubectl get pods
    No resources found in default namespace.
    ```

## Step 2: Create Redis Pod with an initial ConfigMap

1. **Create Redis initial ConfigMap**

   Create an initial ConfigMap in a local file (`example-redis-config.yaml`) with an _empty configuration block_ (`data.redis-config` is empty; more on this in the next step) with the following command:

    ```shell
    cat <<EOF >./example-redis-config.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: example-redis-config
    data:
      redis-config: ""
    EOF
    ```

1. **Apply the ConfigMap**

    Run the following command to apply the ConfigMap defined in `example-redis-config.yaml`:

    ```shell
    kubectl apply -f example-redis-config.yaml
    ```

    When applied successfully, you will see the message `configmap/example-redis-config created`, and can verify is was created in the default namespace with the command below:

    ```shell
    $ kubectl get configmap example-redis-config
    NAME                   DATA   AGE
    example-redis-config   1      2m35s
    ```

    Take a moment to notice that the `example-redis-config` ConfigMap contains no configuration due to the empty `redis-config` key in the original file:

    ```shell
    $ kubectl describe configmap example-redis-config
    Name:         example-redis-config
    Namespace:    default
    Labels:       <none>
    Annotations:  <none>

    Data
    ====
    redis-config:
    ----
    ```

1. **Apply manifest to create a Redis pod**

   `example-redis-config.yaml` contains configuration for a key-value store of configuration we can leverage when starting up a Pod. 
   While `example-redis-config.yaml` is set up to do this through the `kind: ConfigMap` configuration on line 2 of that file, we can use 
   [a _Pod manifest_ file](https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml) that 
   does the same thing for a resource of `kind: Pod`. 

   Take a moment to inspect this file:

   {{% code_sample file="pods/config/redis-pod.yaml" %}}

   Examine the contents of the Redis pod manifest and note the following:

    * A volume named `config` is created by `spec.volumes[1]`.
    * The `key` and `path` under `spec.volumes[1].configMap.items[0]` exposes the `redis-config` key from the 
      `example-redis-config` ConfigMap as a file named `redis.conf` on the `config` volume.
    * The `config` volume is then mounted at `/redis-master` by `spec.containers[0].volumeMounts[1]`.

   This has the net effect of exposing the data in `data.redis-config` from the `example-redis-config`
   ConfigMap above as `/redis-master/redis.conf` inside the Pod.

   Apply this manifest file to the cluster to create the Redis cache Pod.

   ```shell
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
   ```

   Examine the objects you've created up to now:

   ```shell
   kubectl get pod/redis configmap/example-redis-config
   ```

   You should see the following output:

   ```
   NAME        READY   STATUS    RESTARTS   AGE
   pod/redis   1/1     Running   0          8s

   NAME                             DATA   AGE
   configmap/example-redis-config   1      14s
   ```

## Step 3: Verify Redis configuration

1. **Verify Redis configuration using the Redis CLI**

   Use `kubectl exec` to enter the pod and run the `redis-cli` tool to check the current configuration:

   ```shell
   kubectl exec -it redis -- redis-cli
   ```

   Check `maxmemory`:

   ```shell
   127.0.0.1:6379> CONFIG GET maxmemory
   ```

   Since you've included no configuration for Redis in the ConfigMap, it should show the default value of 0:

   ```shell
   1) "maxmemory"
   2) "0"
   ```

   Similarly, check `maxmemory-policy`:

   ```shell
   127.0.0.1:6379> CONFIG GET maxmemory-policy
   ```

   Which should also yield its default value of `noeviction`:

   ```shell
   1) "maxmemory-policy"
   2) "noeviction"
   ```

{{< note >}}
Before moving onto the next step, be sure to exit the Redis CLI session with the command `Ctrl+D`.
{{< /note >}}

## Step 4: Update the ConfigMap

1. **Update Redis configuration within ConfigMap**

   Add some configuration to the ConfigMap file `example-redis-config.yaml` you created locally with the following command:

    ```shell
    cat <<EOF >./example-redis-config.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: example-redis-config
    data:
      redis-config: |
        maxmemory 2mb
        maxmemory-policy allkeys-lru   
    EOF
    ```

1. **Apply the updated ConfigMap**

    At this point, you might expect that this new configuration would change `maxmemory` from `0` to `2mb`, and `maxmemory-policy` from `noeviction` to `allkeys-lru` when applied.

    {{< tabs name="updated_redis_configmap" >}}

    {{% tab name="Updated ConfigMap" %}}
  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: example-redis-config
  data:
    redis-config: |
      maxmemory 2mb
      maxmemory-policy allkeys-lru 
  ```
    {{% /tab %}}

    {{% tab name="Previous configuration" %}}

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: example-redis-config
  data:
    redis-config: ""



  ```
    {{% /tab %}}
    {{< /tabs >}}

    Apply the changes with the command below to test this expectation in the next step:

    ```shell
    kubectl apply -f example-redis-config.yaml
    ```

## Step 5: Verify the updated configuration

1. **Confirm the update to the ConfigMap**

    Confirm that the ConfigMap was updated:

    ```shell
    kubectl describe configmap/example-redis-config
    ```

    You should see the configuration values you just added:

    ```shell
    Name:         example-redis-config
    Namespace:    default
    Labels:       <none>
    Annotations:  <none>

    Data
    ====
    redis-config:
    ----
    maxmemory 2mb
    maxmemory-policy allkeys-lru
    ```

1. **Config the update to the Redis Pod**

    Check the Redis Pod again using `redis-cli` via `kubectl exec` to see if the configuration was applied:

    ```shell
    kubectl exec -it redis -- redis-cli
    ```

    Check `maxmemory`:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory
    ```

    It remains at the default value of 0:

    ```shell
    1) "maxmemory"
    2) "0"
    ```

    Similarly, `maxmemory-policy` remains at the `noeviction` default setting:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory-policy
    ```

    Returns:

    ```shell
    1) "maxmemory-policy"
    2) "noeviction"
    ```

This is not the behavior you might initially expect.
While your updates are visible in the ConfigMap `example-redis-config`, they have not yet been retrieved by the running Redis Pod. 
First you will need to restart Redis in order to grab updated values from the associated `example-redis-config` ConfigMaps. 

{{< note >}}
Before moving onto the next step, be sure to exit the Redis CLI session with the command `Ctrl+D`.
{{< /note >}}

## Step 6: Retrieve updated configuration

1. **Restart the Redis Pod**

    First, delete the Redis pod:

    ```shell
    kubectl delete pod redis
    ```

    Then recreate it using the same Pod manifest you used previously in this tutorial:

    ```shell
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
    ```

1. **Verify configuration has been updated**

    Now re-check the configuration values one last time:

    ```shell
    kubectl exec -it redis -- redis-cli
    ```

    Check `maxmemory`:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory
    ```

    It should now return the updated value of 2097152:

    ```shell
    1) "maxmemory"
    2) "2097152"
    ```

    Similarly, `maxmemory-policy` has also been updated:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory-policy
    ```

    It now reflects the desired value of `allkeys-lru`:

    ```shell
    1) "maxmemory-policy"
    2) "allkeys-lru"
    ```

{{< note >}}
Before moving onto the next step, be sure to exit the Redis CLI session with the command `Ctrl+D`.
{{< /note >}}

## Summary

In this tutorial, you 

* Created a ConfigMap from a file (`example-redis-config.yaml`) which contained configuration for a Redis pod.
* Leveraged an externally defined Pod manifest file to create a Redis Pod.
  You also saw how Pod manifest's can mount an applied ConfigMap through the `volumes` definition.
* Found out how to inspect a running Pod, and leverage its built-in tools (in this case, the Redis CLI for Redis)
  to verify that configuration has been applied as expected.
  You used this again to troubleshoot when configuration had not changed following updates to the ConfigMap.
* Updated a Pod's configuration at runtime by changing its associated ConfigMap.

## Cleaning up

Clean up your work by deleting the resources you created during this tutorial:

```shell
kubectl delete pod/redis configmap/example-redis-config
```

You can also shut down minikube completely at this point:

```shell
minikube stop
minikube delete
```

## Next steps

* Learn more about working with [ConfigMaps](/docs/tasks/configure-pod-container/configure-pod-configmap/).
* Explore more on the topic of [updating configuration via a ConfigMap](/docs/tutorials/configuration/updating-configuration-via-a-configmap/) for a Deployment, including:
  * Updating configuration via ConfigMap mounted as a volume (like in this tutorial), [but with literal key-value literals](/docs/tutorials/configuration/updating-configuration-via-a-configmap/#rollout-configmap-volume)
  * [Updating environment variables of a Pod via ConfigMap](/docs/tutorials/configuration/updating-configuration-via-a-configmap/#rollout-configmap-env)
  * Updating configuration via a ConfigMap when there are [multiple Pod containers in a Deployment](/docs/tutorials/configuration/updating-configuration-via-a-configmap/#rollout-configmap-multiple-containers)
  * Updating configuration via a ConfigMap in a [Pod with a sidecar container](/docs/tutorials/configuration/updating-configuration-via-a-configmap/#rollout-configmap-sidecar)
  * Handling configuration updates to ConfigMaps mounted as volumes, but are [immutable](/docs/tutorials/configuration/updating-configuration-via-a-configmap/#rollout-configmap-immutable-volume)
