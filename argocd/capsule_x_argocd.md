- Beispielhaft für Tenants t1/t2
- Vorbereitung ArgoCD/Ingress/Capsule
- Bei Nutzung von Certmanager
# k edit secret capsule-tls -n capsule-system
...ca.crt -> ca

- Ausführung als mit User + cluster-admin (kubernetes-admin)
- enable-ssl-passthrough=true falls nicht gesetzt
# k edit daemonset.apps/ingress-nginx-controller -n ingress-nginx
...
    spec:
      containers:
      - args:
...
        - --enable-ssl-passthrough=true
...

# k rollout restart  daemonset.apps/ingress-nginx-controller -n ingress-nginx
# k get all -n ingress-nginx


- Erweiterung ArgoCD kontrollierter NS
# k edit cm argocd-cmd-params-cm -n argocd
...
data:
  application.namespaces: "*-argocd, argocd"
...

# kubectl rollout restart -n argocd deployment argocd-server
# kubectl rollout restart -n argocd statefulset argocd-application-controller

- User und Impersonation 
- Beispiel mit ArgoCD User t1-admin/t2-admin 
# k edit cm argocd-cm -n argocd
...
data:
  application.sync.impersonation.enabled: "true"
  timeout.reconciliation: 60s
  users.session.duration: 24h
  accounts.t1-admin: login
  accounts.t2-admin: login

# Ingress Rule + DNS Auflösung zum ArgoCD-Server muss vorhanden sein
# GGF. Too many redirects ERROR
# kubectl edit configmap argocd-cmd-params-cm -n argocd
...
data:
  server.insecure: "true"
...
# kubectl rollout restart deployment argocd-server -n argocd


# argocd login argocd.hs.local --insecure 
# argocd account update-password --account t1-admin --current-password ... --new-password ...
# argocd account update-password --account t2-admin --current-password ... --new-password ...

# k edit cm argocd-rbac-cm -n argocd
...
apiVersion: v1
data:
  policy.csv: |
    p, role:customer_admin-t1, applications, *, default/*, deny
    p, role:customer_admin-t1, applications, *, gisa*/*, deny
    p, role:customer_admin-t1, applications, *, t1-*/*, allow
    p, role:customer_admin-t1, projects, *, t1-*, allow
    p, role:customer_admin-t1, projects, *, default, deny
    p, role:customer_admin-t1, projects, *, gisa*, deny
    p, role:customer_admin-t1, repositories, *, *, allow
    p, role:customer_admin-t1, clusters, *, *, allow
    g, t1-admin, role:customer_admin-t1
...

# k edit appproject default -n argocd
...
spec:
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
  destinationServiceAccounts:
  - defaultServiceAccount: argocd:argocd-application-controller
    namespace: '*'
    server: https://kubernetes.default.svc
  destinations:
  - namespace: '*'
    server: '*'
  sourceRepos:
  - '*'

# k edit CapsuleConfiguration default -n capsule-system
...
spec:
  ...
  forceTenantPrefix: true
...
  userGroups:
  - projectcapsule.dev
  - system:serviceaccounts:*-argocd
  - t1-grp
  - t2-grp


- Tenant t1 anlegen
# vi t1-tenant.yaml
apiVersion: capsule.clastix.io/v1beta2
kind: Tenant
metadata:
  name: t1
spec:
  podOptions:
    additionalMetadata:
      labels:
        tenant: t1
  additionalRoleBindings:
  - clusterRoleName: cluster-admin
    subjects:
    - name: argocd-server
      kind: ServiceAccount
      namespace: argocd
  owners:
  - kind: ServiceAccount
    name: system:serviceaccount:argocd:argocd-application-controller
  - name: system:serviceaccount:t1-argocd:t1-argocd-sa
    kind: ServiceAccount
    clusterRoles:
      - cluster-admin
      - admin
      - argoproj-admin
      - capsule-namespace-deleter
  - name: t1-admin
    kind: User
    clusterRoles:
      - admin
      - capsule-namespace-deleter
      - argoproj-admin
    proxySettings:
    - kind: IngressClasses
      operations:
      - List

# k apply -f t1-tenant.yaml

- Tenant User
- K8s-User admin.t1 + CA
- Capsule-Proxy (muss bereits vorhanden sein) als Beispiel via https://capsule.hs.local
# ./create_user.sh t1-admin t1 t1-grp
# vi admin-t1.kubeconfig
...
    server: https://capsule.hs.local
...

- Erstellung Argocd-Namespace für Apps in Tenant t1
# k create ns t1-argocd --kubeconfig t1-admin-t1.kubeconfig --insecure-skip-tls-verify # capsule-proxy self-signed
# k get ns --kubeconfig t1-admin-t1.kubeconfig --insecure-skip-tls-verify
# k get ns t1-argocd -o json | jq '.metadata.labels."capsule.clastix.io/tenant"'

- Erstellung SA für ArgoCD App-Kontrolle in t1
- Erstellung ArgoCD Project t1-default für t1
- Ausführung als mit User + cluster-admin (kubernetes-admin)
# vi t1-argocd-sa.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: t1-argocd-sa
  namespace: t1-argocd
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: argoproj-admin
rules:
- apiGroups: ["argoproj.io"]
  resources: ["*"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: t1-argocd-sa-argoproj-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: argoproj-admin
subjects:
- kind: ServiceAccount
  name: t1-argocd-sa
  namespace: t1-argocd
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: t1-argocd-sa-cluster-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: t1-argocd-sa
  namespace: t1-argocd

# k apply -f t1-argocd-sa.yaml 

# vi t1-project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: t1-default
  namespace: argocd
spec:
  sourceNamespaces:
    - t1-argocd
  sourceRepos:
    - '*'
  destinations:
    - namespace: 't1-*'
      server: https://kubernetes.default.svc
  destinationServiceAccounts:
    - namespace: 't1-*'
      server: https://kubernetes.default.svc
      defaultServiceAccount: "t1-argocd:t1-argocd-sa"
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'

# k apply -f t1-project.yaml

- Deploy Beispielapp
- https://github.com/git67/k8s-argocd-bootstrap.git/generic/tenants/t1-web1
- https://github.com/git67/k8s-argocd-bootstrap.git/generic/tenants/t1-web2
# ll app
total 8
drwxrwxr-x 2 hs hs  46 Apr  8 16:57 ./
drwxrwxr-x 4 hs hs 248 Apr  8 16:53 ../
-rw-rw-r-- 1 hs hs 519 Apr  8 16:57 t1-web1.yaml
-rw-rw-r-- 1 hs hs 518 Apr  8 16:56 t1-web2.yaml
# k apply -f app/ --kubeconfig t1-admin-t1.kubeconfig --insecure-skip-tls-verify

# k get ns --kubeconfig t1-admin-t1.kubeconfig --insecure-skip-tls-verify
NAME        STATUS   AGE
t1-argocd   Active   4h46m
t1-web1     Active   45m
t1-web2     Active   18m

# k get app -n t1-argocd --kubeconfig t1-admin-t1.kubeconfig --insecure-skip-tls-verify
NAME      SYNC STATUS   HEALTH STATUS
t1-web1   Synced        Progressing
t1-web2   Synced        Progressing














