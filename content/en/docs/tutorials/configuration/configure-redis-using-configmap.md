---
reviewers:
- eparis
- pmorie
title: Configuring Redis using a ConfigMap
content_type: tutorial
weight: 30
---

<!-- overview -->

This page provides a real-world example of using a ConfigMap to configure Redis, building upon the concepts in our [Configure a Pod to Use a ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/) task.



## {{% heading "objectives" %}}


* Create a ConfigMap with Redis configuration values
* Create a Redis Pod that uses the created ConfigMap
* Verify that the configuration was applied correctly



## {{% heading "prerequisites" %}}


1. {{< include "task-tutorial-prereqs.md" >}} 
2. You need a basic understanding of how to [Configure a Pod to Use a ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/).
3. You need to be running Kubernetes 1.14 or higher
    * {{< version-check >}}


<!-- lessoncontent -->


## Real World Example: Configuring Redis using a ConfigMap

Follow the steps below to configure a Redis cache using data stored in a ConfigMap.

### Step 1: Create a ConfigMap and Redis pod
1. In your kubectl command-line tool, add the ConfigMap below:
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
2. Apply the ConfigMap you created
    ```shell
    kubectl apply -f example-redis-config.yaml
    ```
3. Apply the Redis pod manifest
    ```shell
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
    ```
For reference, here's a look at the YAML file you are applying:

{{% code_sample file="pods/config/redis-pod.yaml" %}}

### Step 2: Verify the ConfigMap has been added

1. Use the snippet below to view the full details of the Redis pod YAML.
    ```shell
    kubectl get pod redis -o yaml
    ```
2. Under ```spec.containers.volumeMounts``` verify it matches the following:
    ```shell
        - mountPath: /redis-master
          name: config
    ```
3. Under ```spec.volumes``` verify that the second specification matches the following:
    ```shell
      - configMap:
          defaultMode: 420
          items:
          - key: redis-config
            path: redis.conf
          name: example-redis-config
        name: config
    ```
If the Redis pod matches the snippets above, the ConfigMap has been correctly added as `redis.conf`.

### Step 3: View current configuration

Before making changes, let's see how Redis is currently configured.
First, check that the ConfigMap is running:
```shell
kubectl get pod/redis configmap/example-redis-config 
```

You should get the output below:

```
NAME        READY   STATUS    RESTARTS   AGE
pod/redis   1/1     Running   0          8s

NAME                             DATA   AGE
configmap/example-redis-config   1      14s
```
Now review the ConfigMap's details:

```shell
kubectl describe configmap/example-redis-config
```
You should get the output below. Note that since we left the `redis-config` key blank, it will be empty here. 
```shell
Name:         example-redis-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
redis-config:
```
Next, let's look at how Redis is currently configured:
```shell
kubectl exec -it redis -- redis-cli
```
First check `maxmemory`:
```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

It should show the default value of 0:

```shell
1) "maxmemory"
2) "0"
```

Next, check `maxmemory-policy`:

```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

It should show the default value of `noeviction`:

```shell
1) "maxmemory-policy"
2) "noeviction"
```

###Step 4: Add congiguration values

Now let's add some configuration values to the `example-redis-config` ConfigMap:

{{% code_sample file="pods/config/example-redis-config.yaml" %}}

Apply the updated ConfigMap:

```shell
kubectl apply -f example-redis-config.yaml
```

Confirm that the ConfigMap was updated:

```shell
kubectl describe configmap/example-redis-config
```

You should see the configuration values we just added:

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

The configuration values have not changed because the Pod needs to be restarted to grab updated
values from associated ConfigMaps. Let's delete and recreate the Pod:

```shell
kubectl delete pod redis
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```

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

Clean up your work by deleting the created resources:

```shell
kubectl delete pod/redis configmap/example-redis-config
```

## {{% heading "whatsnext" %}}


* Learn more about [ConfigMaps](/docs/tasks/configure-pod-container/configure-pod-configmap/).
* Follow an example of [Updating configuration via a ConfigMap](/docs/tutorials/configuration/updating-configuration-via-a-configmap/).
