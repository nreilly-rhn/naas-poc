# Namespace as a Service

Git paths inside the Argo CD Applications are `platform` and `tenants/<name>`.

The manifests install the OpenShift GitOps operator on ROSA and reconcile one namespace per customer. Each tenant directory creates a namespace, a resource quota, a limit range, an ingress network policy, and three `RoleBindings`. A second Application creates an `AppProject` on the shared `openshift-gitops` instance so that tenant can deploy workloads into its own namespace. GitOps self-heal restores the quota, limits, network policy, and these bindings if a namespace admin changes them. Each namespace is labeled `argocd.argoproj.io/managed-by: openshift-gitops`. `platform/controller-rbac.yaml` lets the GitOps application controller create the quota, limit range, and network policy.

Argo CD tracks `https://github.com/nreilly-rhn/naas-poc.git` on branch `main`. That URL is set in:

- `bootstrap/root-application.yaml`
- `platform/root-application.yaml`
- `platform/appproject.yaml`
- `platform/appproject-control.yaml`
- `platform/applicationset.yaml`
- `platform/applicationset-projects.yaml`

`bootstrap/root-application.yaml` and `platform/root-application.yaml` are the same manifest. The bootstrap copy is the first apply. The platform copy is what GitOps reconciles afterward. Keep them identical.

Apply the manifests with a cluster-admin kubeconfig. The ROSA cluster must be able to reach that GitHub repository over HTTPS.

## Layout

- `bootstrap/` installs the OpenShift GitOps operator. Apply it once. `bootstrap/oauth.yaml` and `bootstrap/argocd-rbac.yaml` are samples and are not applied with the operator.
- `platform/` holds the `naas` and `naas-control` AppProjects, the tenant ApplicationSets, and the root Application.
- `components/tenant/` holds the shared namespace guardrails.
- `tenants/<name>/` is one customer. `tenants/naas-example/` is the copy-paste reference. `argocd/` is that tenant's Argo CD project. `tenants/acme/` and `tenants/globex/` are dummy organizations for local login.
- `testing/` holds the local users and OpenShift groups for those two organizations. It is not synced by Argo CD.

## Bootstrap

From this directory:

```bash
oc apply -k bootstrap
oc rollout status deployment/openshift-gitops-server -n openshift-gitops
oc apply -f bootstrap/root-application.yaml
```

Skip `bootstrap/` when the OpenShift GitOps operator is already installed, and apply only `bootstrap/root-application.yaml`. The operator Subscription uses the cluster `redhat-operators` catalog, channel `latest`.

The root Application syncs `platform`. `naas-tenants` creates one Application per `tenants/*/tenant.yaml` for the namespace guardrails. `naas-tenant-projects` creates `<name>-argocd`, which syncs that tenant's `AppProject` into `openshift-gitops`. A public repository needs no repository credential.

## Entra ID

Create the identity provider with the ROSA CLI. The provider name `entra` must match the redirect URI registered on the Entra application: `https://oauth-openshift.apps.<cluster-domain>/oauth2callback/entra`. `--groups-claims roles` maps Entra app roles to OpenShift groups. Those role values are `<namespace>-admins`, `<namespace>-devs`, and `<namespace>-ro`.

Pass the client secret on the command line and keep it out of git. For one Entra directory, set `--issuer-url` to `https://login.microsoftonline.com/<entra-tenant-id>/v2.0`.

```bash
rosa create idp \
  --cluster <cluster-name> \
  --type openid \
  --name entra \
  --mapping-method claim \
  --client-id <application-client-id> \
  --client-secret <client-secret> \
  --issuer-url https://login.microsoftonline.com/organizations/v2.0 \
  --username-claims preferred_username \
  --name-claims name \
  --email-claims email \
  --extra-scopes profile,email \
  --groups-claims roles
```

`bootstrap/oauth.yaml` is the same provider as a cluster `OAuth` object. The CLI stores the client secret itself, so leave that sample unapplied when the identity provider is created with `rosa`.

## Local users

Acme and Globex are dummy organizations for a test cluster that has no external identity provider. Their tenant directories are `tenants/acme` and `tenants/globex`. Each has the usual three groups.

`testing/users.htpasswd` contains APR1 hashes for the local accounts. Every account uses the password `NaasLocal-Test1`. Create the identity provider from that file, then apply the groups. Group membership is what puts a user into `<namespace>-admins`, `<namespace>-devs`, or `<namespace>-ro`.

```bash
rosa create idp \
  --type=htpasswd \
  --cluster=<cluster-name> \
  --name=htpasswd \
  --from-file=testing/users.htpasswd
oc apply -k testing
```

| User | Group | Namespace |
| --- | --- | --- |
| `acme-admin` | `acme-admins` | `acme` |
| `acme-dev` | `acme-devs` | `acme` |
| `acme-reader` | `acme-ro` | `acme` |
| `globex-admin` | `globex-admins` | `globex` |
| `globex-dev` | `globex-devs` | `globex` |
| `globex-reader` | `globex-ro` | `globex` |

The identity provider takes a minute or two to become available. Then:

```bash
oc login --username=acme-admin --password='NaasLocal-Test1'
oc whoami --show-groups
oc auth can-i create deployment -n acme
oc auth can-i create deployment -n globex
```

`acme-admin` can create workloads in `acme`. The same check in `globex` is denied. `globex-reader` can get objects in `globex` and cannot create them.

## Tenant Argo CD

Tenants use the cluster `openshift-gitops` instance. Each tenant's project is `<name>`. It accepts any source repository, deploys only into namespace `<name>`, and cannot create cluster-scoped resources or the platform guardrails (`ResourceQuota`, `LimitRange`, `NetworkPolicy`, `Role`, `RoleBinding`). Workload `Application` objects are created in `openshift-gitops` through Argo CD. They are not created in the tenant namespace.

Log in with the same Entra identity. The project roles match the three groups:

| Group | AppProject role | Argo CD access |
| --- | --- | --- |
| `<name>-admins` | `admin` | create, sync, delete, and read logs for every Application in the project |
| `<name>-devs` | `dev` | create, update, sync, and read logs |
| `<name>-ro` | `readonly` | get Applications and read logs |

The default instance often grants `role:readonly` to every logged-in user, which lists every tenant's Applications. `bootstrap/argocd-rbac.yaml` is a merge patch that sets an empty `defaultPolicy` and leaves `role:admin` with `cluster-admins`. Apply it once after the instance exists. It is not part of `oc apply -k bootstrap`.

```bash
oc patch argocd openshift-gitops \
  -n openshift-gitops \
  --type=merge \
  --patch-file bootstrap/argocd-rbac.yaml
```

A private workload repository needs a repository credential on the instance. Those secrets are visible to the instance, so a cluster admin creates them. Keep the token out of git.

```bash
oc create secret generic <name>-repo \
  --namespace openshift-gitops \
  --from-literal=type=git \
  --from-literal=url=https://github.com/<org>/<workload>.git \
  --from-literal=username=git \
  --from-literal=password='<token>'
oc label secret <name>-repo -n openshift-gitops argocd.argoproj.io/secret-type=repository
```

## Add a tenant

Copy `tenants/naas-example` to `tenants/<name>`. Use a lowercase DNS-1123 name. These values must all be `<name>`:

- the directory name
- `name` in `tenant.yaml`
- `namespace` in `kustomization.yaml`
- the `naas.naramajac.xyz/tenant` label
- `metadata.name` and `spec.destinations[0].namespace` in `argocd/appproject.yaml`
- the `proj:<name>:<role>` policies and `<name>/*` objects in that same file

Each namespace has three groups. The Entra app role value and the `RoleBinding` subject use the namespace name as a prefix, so membership in one tenant does not grant another:

| Group | App role and `RoleBinding` subject | ClusterRole |
| --- | --- | --- |
| namespace-admins | `<name>-admins` | `admin` |
| namespace-devs | `<name>-devs` | `edit` |
| namespace-ro | `<name>-ro` | `view` |

For `naas-example` those subjects are `naas-example-admins`, `naas-example-devs`, and `naas-example-ro`. Set the same three strings in `tenant.yaml`, `rolebinding-patch.yaml`, and `argocd/appproject.yaml`. Create a matching Entra app role for each, and assign that customer's Entra group to the role. The OpenShift groups claim reads the token `roles` claim, so a user receives these names at login. Change `quota-patch.yaml` for that customer's quota. Commit the directory to `main`.

The reference quota is 1 CPU request, 2Gi memory request, 2 CPU limit, 4Gi memory limit, 10 pods, 2 persistent volume claims, and 10Gi of storage. PVC requests use the ROSA default storage class. The limit range gives every container a request and caps one container at the requests quota.

## Remove a tenant

Delete `tenants/<name>` and commit to `main`. Both ApplicationSets remove their Applications. The guardrail Application deletes the namespace and the workloads in it. The `<name>-argocd` Application deletes that tenant's `AppProject`.

## Private repository

Create this Secret only when the GitHub repository is private. Keep the token out of git. The `url` value must match `repoURL` in the Application, AppProject, and ApplicationSet.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: naas-repo
  namespace: openshift-gitops
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/nreilly-rhn/naas-poc.git
  username: git
  password: <token>
```
