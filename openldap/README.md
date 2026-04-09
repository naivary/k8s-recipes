# Install and Run OpenLDAP on Kubernetes

To run OpenLDAP on Kubernetes we will use the `osixia/openldap` container image.
If you have access to `bitnami/openldap` it is recommended to use that for
regular updates.

## Install OpenLDAP

```bash
kubectl create cm ldifs --from-file=uid-admin.ldif=ldifs/uid-admin.ldif --from-file=acl.ldif=ldifs/acl.ldif --from-file=ou-groups-and-users.ldif=ldifs/ou-groups-and-users.ldif
```

```bash
kubectl create -f server.yaml
```

Using the `server.yaml` manifest you will create a deployment which is running
OpenLDAP, a service of type `NodePort` exposing the LDAP port 369 and a
ConfigMap which is containing the OpenLDAP environment variables.

A list of all configuration options can be found
[here](https://github.com/leenooks/phpLDAPadmin/wiki/Configuration-Variables).

## Install phpLDAPAdmin

```bash
kubectl create -f dashboard.yaml
```

This manifest includes the actual dashboard deployment, a configmap for the
configuration using environment variables and a service of type `NodePort`
exposing the dashboard for external traffic.

A list of all configuration options can be found
[https://github.com/leenooks/phpLDAPadmin/wiki/Configuration-Variables].
