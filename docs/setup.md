# Setup

## Install Istio in sidecar mode

```shell
istioctl install --set values.global.platform=k3d
```

## Deploy the `httpbin` sample application

Create the namespace, configure it for sidecar injection, and deploy the workload:

```shell
kubectl create ns httpbin
kubectl label ns httpbin istio-injection=enabled
kubectl apply -n httpbin -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/httpbin/httpbin.yaml
```

Verify that the workload is running and has two containers (that sidecar injection took place):

```shell
kubectl get pod -n httpbin
```

Deploy a `curl` client to test `httpbin`:

```shell
kubectl apply -n httpbin -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/curl/curl.yaml
```

Send a test request to `httpbin`:

```shell
kubectl exec -n httpbin deploy/curl -- curl -s httpbin:8000/get
```

## Apply an EnvoyFilter

For this example, we wish to apply local rate limiting to `httpbin`.

The Istio documentation [provides an example](https://istio.io/latest/docs/tasks/policy-enforcement/rate-limit/#local-rate-limit) which we can adapt for our scenario:

```yaml title="envoyfilter.yaml" linenums="1"
--8<-- "envoyfilter.yaml"
```

Per the example documentation, the EnvoyFilter enables local rate limiting for traffic to the `httpbin` service.
The `HTTP_FILTER` patch inserts the `envoy.filters.http.local_ratelimit` local envoy filter into the HTTP connection manager filter chain.
The local rate limit filter’s token bucket is configured to allow 4 requests per minute.
The filter is also configured to add an `x-local-rate-limit` response header to requests that are blocked.

Apply the EnvoyFilter:

```shell
kubectl apply -n httpbin -f artifacts/envoyfilter.yaml
```

### Validate

Verify that the filter functions.

```shell
for i in {1..5}; do kubectl exec -n httpbin deploy/curl -- curl -s --head httpbin:8000/json; done
```

The output will show four consecutive HTTP 200 (Success) responses, followed by a rate-limited response:

```console
HTTP/1.1 200 OK
access-control-allow-credentials: true
access-control-allow-origin: *
content-type: application/json; charset=utf-8
date: Fri, 06 Jun 2025 04:34:32 GMT
x-envoy-upstream-service-time: 0
server: envoy
transfer-encoding: chunked

...

HTTP/1.1 429 Too Many Requests
x-local-rate-limit: true
content-length: 18
content-type: text/plain
date: Fri, 06 Jun 2025 04:34:32 GMT
server: envoy
x-envoy-upstream-service-time: 1
```

Note the added `x-local-rate-limit` header in the response which came from the configuration of our EnvoyFilter.

## Next..

We wish migrate to ambient mode without losing this capability.

Next, we turn our attention to this migration task.
