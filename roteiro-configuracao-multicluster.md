# Roteiro — Dev Spaces multicluster

## Ambiente

OpenShift 4.20.15 · Dev Spaces 3.31.3 · OpenShift GitOps 1.22.3 · IdP Entra ID (`AAD`, claim `groups`)

Executar os comandos em ambos os clusters.

## Pré-requisitos

| Item | Valor |
| --- | --- |
| Repositório GitOps | `https://github.com/marcusviniciusaso/devspaces-multicluster-openshift` |
| Repositório do redirector | `https://github.com/marcusviniciusaso/devspaces-multicluster-redirector` |
| Repositório do sample | `https://github.com/marcusviniciusaso/custom-udi-java` |
| Token GitHub com leitura nos repositórios privados | `<GITHUB_PAT>` |
| GitHub OAuth App do aro1 (Settings → Developer settings → OAuth Apps) | Homepage `https://devspaces.apps.kmw9dlzrxf7af3b80c.eastus.aroapp.io/` · Callback `https://devspaces.apps.kmw9dlzrxf7af3b80c.eastus.aroapp.io/api/oauth/callback` → `<CLIENT_ID>` / `<CLIENT_SECRET>` |
| GitHub OAuth App do aro2 | Homepage `https://devspaces.apps.pmw9dsj2w2debff947.eastus.aroapp.io/` · Callback `https://devspaces.apps.pmw9dsj2w2debff947.eastus.aroapp.io/api/oauth/callback` → `<CLIENT_ID>` / `<CLIENT_SECRET>` |

```bash
git clone https://github.com/marcusviniciusaso/devspaces-multicluster-openshift.git
cd devspaces-multicluster-openshift
```

## 1. Operators

### 1.1 Dev Spaces

```bash
oc apply -f - <<'YAML'
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: devspaces
  namespace: openshift-operators
spec:
  channel: stable
  name: devspaces
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Manual
YAML
oc get installplan -n openshift-operators
oc patch installplan <NOME> -n openshift-operators --type merge -p '{"spec":{"approved":true}}'
oc get csv -n openshift-operators
```

```
NAME                           DISPLAY                        VERSION   PHASE
devspacesoperator.v3.31.3      Red Hat OpenShift Dev Spaces   3.31.3    Succeeded
devworkspace-operator.v1.0.0   DevWorkspace Operator          1.0.0     Succeeded
```

### 1.2 OpenShift GitOps

```bash
oc apply -f - <<'YAML'
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-gitops-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-gitops-operator
  namespace: openshift-gitops-operator
spec:
  upgradeStrategy: Default
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-gitops-operator
spec:
  channel: latest
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Manual
YAML
oc get installplan -n openshift-gitops-operator
oc patch installplan <NOME> -n openshift-gitops-operator --type merge -p '{"spec":{"approved":true}}'
oc get csv -n openshift-gitops-operator
```

```
NAME                                DISPLAY                    VERSION   PHASE
openshift-gitops-operator.v1.22.3   Red Hat OpenShift GitOps   1.22.3    Succeeded
```

## 2. Secrets de acesso ao GitHub

### 2.1 Dev Spaces — GitHub OAuth App (`gitServices.github`)

```bash
oc apply -f openshift/base/namespace.yaml
oc apply -f - <<'YAML'
apiVersion: v1
kind: Secret
metadata:
  name: github-oauth-config
  namespace: openshift-devspaces
  labels:
    app.kubernetes.io/part-of: che.eclipse.org
    app.kubernetes.io/component: oauth-scm-configuration
  annotations:
    che.eclipse.org/oauth-scm-server: github
    che.eclipse.org/scm-server-endpoint: https://github.com
type: Opaque
stringData:
  id: <CLIENT_ID>
  secret: <CLIENT_SECRET>
YAML
```

### 2.2 GitOps — credencial do repositório privado

```bash
oc apply -f argocd/base/namespace.yaml
oc apply -f - <<'YAML'
apiVersion: v1
kind: Secret
metadata:
  name: github-repo-creds
  namespace: devspaces-gitops
  labels:
    argocd.argoproj.io/secret-type: repo-creds
type: Opaque
stringData:
  type: git
  url: https://github.com/marcusviniciusaso
  username: marcusviniciusaso
  password: <GITHUB_PAT>
YAML
```

## 3. Instância do Dev Spaces

```bash
oc apply -k openshift/aro1
oc get checluster devspaces -n openshift-devspaces \
  -o jsonpath='{.status.chePhase} {.status.cheVersion} {.status.cheURL}{"\n"}'
```

```
Active 3.31.3 https://devspaces.apps.kmw9dlzrxf7af3b80c.eastus.aroapp.io
```

```bash
oc get checluster devspaces -n openshift-devspaces \
  -o jsonpath='{.spec.gitServices.github[0].secretName} {.spec.networking.auth.advancedAuthorization.allowGroups}{"\n"}'
```

```
github-oauth-config ["73f74865-025d-404c-8214-22a3941a563b","6a758b5d-bbfb-498c-ae13-0cea9803de29"]
```

```bash
oc get deploy devspaces -n openshift-devspaces -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="CHE_OAUTH2_GITHUB_CLIENTID__FILEPATH")].value}{"\n"}'
```

```
/che-conf/oauth/github/id
```

## 4. Instância dedicada do GitOps

`argocd/base` cria o namespace `devspaces-gitops`, a instância `devspaces-gitops` e o ClusterRole `devspaces-gitops-scc` (a instância cluster-scoped não recebe permissão em SCCs por padrão).

```bash
oc patch subscription openshift-gitops-operator -n openshift-gitops-operator --type merge \
  -p '{"spec":{"config":{"env":[{"name":"ARGOCD_CLUSTER_CONFIG_NAMESPACES","value":"openshift-gitops,devspaces-gitops"}]}}}'
oc apply -k argocd/base
oc get argocd devspaces-gitops -n devspaces-gitops -o jsonpath='{.status.phase}{"\n"}'
oc get pods -n devspaces-gitops
oc get route devspaces-gitops-server -n devspaces-gitops -o jsonpath='https://{.spec.host}{"\n"}'
```

```
Available
NAME                                                         READY   STATUS    RESTARTS   AGE
devspaces-gitops-application-controller-0                    1/1     Running   0          2m
devspaces-gitops-dex-server-...                              1/1     Running   0          2m
devspaces-gitops-redis-...                                   1/1     Running   0          2m
devspaces-gitops-repo-server-...                             1/1     Running   0          2m
devspaces-gitops-server-...                                  1/1     Running   0          2m
https://devspaces-gitops-server-devspaces-gitops.apps.kmw9dlzrxf7af3b80c.eastus.aroapp.io
```

Login na console do ArgoCD: **LOG IN VIA OPENSHIFT** (grupo admin e `cluster-admins` recebem `role:admin`).

Namespaces gerenciados precisam do label `argocd.argoproj.io/managed-by` (já presente nos `namespace.yaml` dos repositórios; o operator cria os RoleBindings):

```bash
oc label ns openshift-devspaces argocd.argoproj.io/managed-by=devspaces-gitops --overwrite
oc create ns devspaces-redirector
oc label ns devspaces-redirector argocd.argoproj.io/managed-by=devspaces-gitops --overwrite
oc get rolebinding -n openshift-devspaces -n devspaces-redirector | grep devspaces-gitops
```

```
devspaces-gitops-argocd-application-controller   Role/devspaces-gitops-argocd-application-controller
devspaces-gitops-argocd-server                   Role/devspaces-gitops-argocd-server
```

## 5. Applications

| Application | Repositório | Path | prune |
| --- | --- | --- | --- |
| `openshift-aro1` | devspaces-multicluster-openshift | `openshift/aro1` | `false` |
| `devspaces-aro1` | devspaces-multicluster-openshift | `devspaces/aro1` | `true` |
| `redirector-aro1` | devspaces-multicluster-redirector | `openshift/aro1` | `true` |

```bash
oc apply -k argocd/aro1
oc get application -n devspaces-gitops
```

```
NAME              SYNC STATUS   HEALTH STATUS
devspaces-aro1    Synced        Healthy
openshift-aro1    Synced        Healthy
redirector-aro1   Synced        Progressing
```

`redirector-aro1` fica `Progressing` até o secret de sessão do passo 6. Para forçar um novo sync:

```bash
oc patch application redirector-aro1 -n devspaces-gitops --type merge \
  -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"HEAD"}}}'
```

## 6. Redirector

Mapeamento (`openshift/base/configmap.yaml` do repositório do redirector):

```json
{
  "9fc9bd05-1de2-4fad-920d-b8427e6cf276": "https://devspaces.apps.pmw9dsj2w2debff947.eastus.aroapp.io",
  "73f74865-025d-404c-8214-22a3941a563b": "https://devspaces.apps.kmw9dlzrxf7af3b80c.eastus.aroapp.io"
}
```

Overlay `openshift/aro1`: annotation `serviceaccounts.openshift.io/oauth-redirecturi.primary: https://devspaces-redirector.apps.kmw9dlzrxf7af3b80c.eastus.aroapp.io/oauth/callback` e `spec.host` da Route.

Imagem (`openshift/base/deployment.yaml`): `quay.io/marolive/devspaces-multicluster-redirector:1.0.1`, build do fork. A imagem oficial `registry.redhat.io/devspaces/multicluster-redirector-rhel9:3.31.3` lê o usuário de `X-Forwarded-User`, que o oauth-proxy corta no `@`; com nomes de usuário em formato de e-mail (Entra ID) nenhum grupo casa. O fork usa `X-Forwarded-Email` (`UserInfoResource.resolveUserName`).

```bash
mvn package -DskipTests -Dquarkus.container-image.build=true -Dquarkus.container-image.push=true \
  -Dquarkus.container-image.registry=quay.io -Dquarkus.container-image.group=<org> \
  -Dquarkus.container-image.name=devspaces-multicluster-redirector -Dquarkus.container-image.tag=1.0.1 \
  -Dquarkus.jib.platforms=linux/amd64
```

```bash
oc create secret generic devspaces-multicluster-redirector-session-secret \
  -n devspaces-redirector --from-literal=session_secret=$(openssl rand -base64 32)
oc get pods,route -n devspaces-redirector
oc get application redirector-aro1 -n devspaces-gitops
curl -sk -o /dev/null -w '%{http_code}\n' https://devspaces-redirector.apps.kmw9dlzrxf7af3b80c.eastus.aroapp.io/
```

```
NAME                                                     READY   STATUS    RESTARTS   AGE
pod/devspaces-multicluster-redirector-...                2/2     Running   0          1m

NAME                                                         HOST/PORT
route.route.openshift.io/devspaces-multicluster-redirector   devspaces-redirector.apps.kmw9dlzrxf7af3b80c.eastus.aroapp.io

NAME              SYNC STATUS   HEALTH STATUS
redirector-aro1   Synced        Healthy
403
```

`403` = página de login do oauth-proxy para usuário não autenticado.

## 7. Sample a partir de repositório privado

`devspaces/base/samples/sample-custom-udi-java.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sample-custom-udi-java
  namespace: openshift-devspaces
  labels:
    app.kubernetes.io/component: getting-started-samples
    app.kubernetes.io/part-of: che.eclipse.org
data:
  sample-custom-udi-java.json: |-
    [
      {
        "displayName": "Java multi versão (custom UDI)",
        "description": "...",
        "tags": ["Java", "Spring", "Maven", "Gradle"],
        "url": "https://github.com/marcusviniciusaso/custom-udi-java",
        "icon": { "base64data": "<SVG_BASE64>", "mediatype": "image/svg+xml" }
      }
    ]
```

```bash
oc get cm -n openshift-devspaces -l app.kubernetes.io/component=getting-started-samples
```

```
NAME                     DATA   AGE
sample-custom-udi-java   1      5m
```

Dashboard `https://devspaces.apps.kmw9dlzrxf7af3b80c.eastus.aroapp.io` → card **Java multi versão (custom UDI)** → **Authorize** GitHub (OAuth App do passo 2.1) → workspace inicia.

```bash
oc get devworkspace -A
```

```
NAMESPACE                       NAME                   DEVWORKSPACE ID             PHASE     INFO
<usuario>-devspaces             custom-udi-java-xxxx   workspacexxxxxxxxxxxxxxxx   Running   Workspace is running
```

## 8. Repetir no aro2

| Item | aro2 |
| --- | --- |
| Secret `github-oauth-config` | `<CLIENT_ID>` / `<CLIENT_SECRET>` do OAuth App do aro2 |
| Instância Dev Spaces | `oc apply -k openshift/aro2` → `allowGroups` com team-a `9fc9bd05-...` |
| Applications | `oc apply -k argocd/aro2` → `openshift-aro2`, `devspaces-aro2`, `redirector-aro2` |
| Redirector | host `devspaces-redirector.apps.pmw9dsj2w2debff947.eastus.aroapp.io` |

## Estrutura dos repositórios

```
devspaces-multicluster-openshift/
├── openshift/            # essenciais (prune: false)
│   ├── base/             # namespace, SCCs, CheCluster (gitServices.github)
│   ├── aro1/             # patch: advancedAuthorization team-b + admins
│   └── aro2/             # patch: advancedAuthorization team-a + admins
├── devspaces/            # configuráveis (prune: true)
│   ├── base/samples/     # ConfigMap do sample custom-udi-java
│   ├── aro1/
│   └── aro2/
└── argocd/
    ├── base/             # namespace devspaces-gitops + ArgoCD + ClusterRole SCC
    ├── aro1/             # Applications openshift-aro1, devspaces-aro1, redirector-aro1
    └── aro2/

devspaces-multicluster-redirector/
└── openshift/
    ├── base/             # namespace, SA, RBAC, ConfigMap (mapping), Deployment (imagem do fork 1.0.1), Service, Route
    ├── aro1/             # patch: oauth-redirecturi + host da Route
    └── aro2/
```

Secrets fora do Git: `github-oauth-config` (openshift-devspaces), `github-repo-creds` (devspaces-gitops), `devspaces-multicluster-redirector-session-secret` (devspaces-redirector).
