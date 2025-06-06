# Migrate to ambient mode

## Upgrade Istio to ambient

```shell
istioctl install --set profile=ambient --set values.global.platform=k3d --skip-confirmation
```

Verify that, in `istio-system` namespace we now have added Istio components necessary for ambient mode:  the Istio CNI node agent, and the ztunnel Daemonset.

```shell
kubectl get pod -n istio-system
```

```console
NAME                      READY   STATUS    RESTARTS   AGE
istio-cni-node-64xj2      1/1     Running   0          74s
istiod-77f86b5796-5ksqz   1/1     Running   0          106s
ztunnel-f84rm             1/1     Running   0          50s
```

## Install the Kubernetes Gateway API

Istio ambient mode requires the [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/).

Install the standard channel CRDs:

```shell
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
```

## Checkpoint

At this point, the EnvoyFilter continues to function because the workloads still have sidecars.

Verify this:

```shell
for i in {1..5}; do kubectl exec -n httpbin deploy/curl -- curl -s --head httpbin:8000/json; done
```

You should still see the fifth call return an HTTP 429 (Rate-limited).

## Migrate the data plane

To migrate the workloads to ambient:

1.  Remove the `istio-injection` labels:

    ```shell
    kubectl label namespace httpbin istio-injection-
    ```

2. Add the `dataplane-mode` label

    ```shell
    kubectl label namespace httpbin istio.io/dataplane-mode=ambient
    ```

    This label will ensure that ztunnel intercepts traffic in and out of our workloads.

3. Remove the sidecars by restarting the workloads:

    ```shell
    kubectl rollout restart deploy -n httpbin
    ```

Verify that the workloads no longer bear sidecars:

```shell
kubectl get pod -n httpbin
```

```console
NAME                       READY   STATUS    RESTARTS   AGE
curl-5d7946555f-ts24c      1/1     Running   0          10s
httpbin-7dbddd7b86-7mx6b   1/1     Running   0          10s
```

Inspect the workloads to ascertain that they use the HBONE protocol:

```shell
istioctl ztunnel-config workload
```

```console
NAMESPACE    POD NAME                  ADDRESS      NODE                    WAYPOINT PROTOCOL
httpbin      curl-5d7946555f-ts24c     10.42.0.14   k3d-my-cluster-server-0 None     HBONE
httpbin      httpbin-7dbddd7b86-7mx6b  10.42.0.15   k3d-my-cluster-server-0 None     HBONE
...
```

## Test rate-limiting

This time you will notice that the EnvoyFilter has stopped functioning:

```shell
for i in {1..5}; do kubectl exec -n httpbin deploy/curl -- curl -s --head httpbin:8000/json; done
```

This makes perfect sense:  ztunnel is in the path of requests to `httpbin`, but an EnvoyFilter requires a waypoint.

## Add the waypoint

Per the [documentation on ambientmesh.io](https://ambientmesh.io/docs/setup/configure-waypoints/), we can deploy a waypoint to the `httpbin` namespace with the command:

```shell
istioctl waypoint apply -n httpbin --enroll-namespace
```

Note the `--enroll-namespace` option which automatically binds workloads in that namespace to the waypoint.

The output confirms the operation:

```console
✅ waypoint httpbin/waypoint applied
✅ namespace httpbin labeled with "istio.io/use-waypoint: waypoint"
```

You can also see the waypoint running in the `httpbin` namespace:

```shell
kubectl get pod -n httpbin
```

```console
NAME                        READY   STATUS    RESTARTS   AGE
curl-5d7946555f-ts24c       1/1     Running   0          10m
httpbin-7dbddd7b86-7mx6b    1/1     Running   0          10m
waypoint-7d4fdb5b7c-rlq57   1/1     Running   0          63s
```

## Another checkpoint

Now that we have a waypoint fronting `httpbin`, let's test rate-limiting again:

```shell
for i in {1..5}; do kubectl exec -n httpbin deploy/curl -- curl -s --head httpbin:8000/json; done
```

We discover that rate limiting is still not functioning.

The reason is that EnvoyFilter is not supported in open-source Istio in ambient mode (i.e. applied to waypoints).

Thankfully, solo.io provides builds of Istio that do.

## Upgrade to Solo's build

The ambientmesh.io documentation provides the instructions for [using Solo's builds](https://ambientmesh.io/docs/operations/solo-builds/).

Upgrade Istio by overriding the `hub` and `tag` configuration options.  `hub` tells the Istio CLI to fetch images from a different location, while the `tag` option specifies the version to use:

```shell
istioctl install --set profile=ambient --set values.global.platform=k3d --skip-confirmation \
  --set hub=us-docker.pkg.dev/soloio-img/istio \
  --set tag=1.26.1-solo
```

Verify that Istio's version is now the Solo version:

```shell
istioctl version
```

```console
client version: 1.26.1
control plane version: 1.26.1-solo
data plane version: 1.26.1-solo (2 proxies)
```

## Verify rate-limiting once more

Re-run the test command:

```shell
for i in {1..5}; do kubectl exec -n httpbin deploy/curl -- curl -s --head httpbin:8000/json; done
```

## Retrofit the EnvoyFilter spec

In sidecar mode the EnvoyFilter targeted the service via the `workloadSelector` field.
In ambient mode, the EnvoyFilter must reference the waypoint via the `targetRefs` field.

Here is a revised envoy filter for ambient:

```yaml title="envoyfilter-ambient.yaml" linenums="1" hl_lines="7-9"
--8<-- "envoyfilter-ambient.yaml"
```

Apply the updated EnvoyFilter:

```shell
kubectl apply -n httpbin -f artifacts/envoyfilter-ambient.yaml
```