# Introduction

When migrating to ambient mesh, one of the concerns that engineering teams face is the scenario where their existing mesh configurations include the use of [EnvoyFilters](https://istio.io/latest/docs/reference/config/networking/envoy-filter/).

An example is rate limiting, a feature that Istio does not natively provide a custom resource definition for.
To apply Envoy rate limiting in Istio, users must employ an EnvoyFilter.

In ambient mode, there are better ways to solve this problem.
Waypoints play the role of microgateways that proxy mesh services.
In ambient mode, we can plug in an alternative waypoint implementation that does provide native support for rate limiting (or whatever specific API engineering teams are looking to implement).

For example, solo.io's kgateway project provides first-class support for rate limiting.
Plugging in kgateway as a waypoint allows us to eschew ourselves of the use of EnvoyFilters.

Nevertheless, when performing a migration to ambient mode, as a first step, it is desirable to be able to migrate existing resources "as is."

Open-source Istio does not support EnvoyFilters for waypoints.
Solo.io however provides builds of Istio that do.

In this exercise, you will start with an Istio installation in sidecar mode, and configure an Envoy Filter for local rate limiting, and test it.

Next, we will walk through upgrading Istio to use Solo.io's build, and migrate to ambient mode.
In the process, we will demonstrate the continued functioning of the EnvoyFilter post-migration.