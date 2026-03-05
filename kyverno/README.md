# Kyverno

A collection of useful Kyverno Policies.

## Block Label Namespace

Reserve a special label namespace for internal usage like:
\*.<companyname>.com/\*.

## Block Operations for Label Namspace

Block any operations (CREATE, REMOVE etc.) on Kubernetes Objects with your
choosen internal namespace e.g. \*.<companyname>.com/\*.

## Restrict Image Registries

Define which image registries are available for pods to pull images from.
Useable image registries can be defined on namespace level using a
custom-defined label and globally using a configmap.

## Default Resource Quota

A default resource quota to be inserted into every newly created namespace.

## Deny All Ingress

Deny all ingress to any pod in a namespace.

## Deny All Egress

Deny all egress from any pod. The only exception is `kube-dns` in the
`kube-system` namespace to allow pods to resolve internal DNS records.
