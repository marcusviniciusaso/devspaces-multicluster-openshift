# devspaces-multicluster-openshift

Artefatos do Dev Spaces aplicados por OpenShift GitOps nos clusters aro1 e aro2.

| Diretório | Conteúdo | Application | prune |
| --- | --- | --- | --- |
| `openshift/` | Essenciais: namespace, SCCs, CheCluster | `openshift-<cluster>` | `false` (permanecem no cluster ao sair do Git) |
| `devspaces/` | Configuráveis: samples e demais escolhas | `devspaces-<cluster>` | `true` (removidos do cluster ao sair do Git) |
| `argocd/` | Instância ArgoCD dedicada e Applications | aplicado com `oc apply -k` | — |

Cada diretório tem `base/` (comum) e overlays `aro1/` e `aro2/` com os patches específicos de cada cluster.

```bash
oc kustomize openshift/aro1
oc kustomize devspaces/aro1
oc kustomize argocd/aro1
```

O redirector é aplicado a partir do repositório `devspaces-multicluster-redirector` (`openshift/<cluster>`) pela Application `redirector-<cluster>`.
