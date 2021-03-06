# Jam on Argocd


```
$ tree argocd/
argocd/
├── applications
│   ├── Chart.yaml
│   ├── templates
│   │   ├── argocd.yaml
│   │   ├── jam.yaml
│   │   ├── logging.yaml
│   │   └── sealded_secrets.yaml
│   └── values.yaml
└── projects
    ├── Chart.yaml
    ├── templates
    │   └── projects.yaml
    └── values.yaml

4 directories, 9 files
```

### Project

ArgoCD could both deploy apps whin the same k8s cluster it been deployed at or an external cluster. So we can define multiple projects. Since we are only dealing with in-cluster apps, we use projects to distinguish deferent function modules. We have three projects defined: 

* **`$JAM_INSTANCE`**: the project contains all of our business services related to jam; 
* **logging**: the project contains EFK stack, and 
* **arogocd**: contains both ArgoCD project and ArgoCD application definitions. 
* **workzone-sealed-secrets**: All secrets contained in cluster 

So the definitions for applications are self-deployed.

### Application:

An application equals a helm release. We have all the jam services application definitions in our repository. We can define the helm chart path, value path, sync policy, and target GitHub branch for a certain application.

### Sync:

Sync means deployment. ArgoCD will sync with the target repository every couple minutes and use helm to render templates. 

* If the rendered manifest is different from the online manifest, ArgoCD will mark the application as **"out of sync"**. 
* If the application is configured as **"auto sync"**, ArgoCD will automatically deploy the outdated applications, or we need to deploy it manually.
*  ArgoCD will also convert helm hook to Argo hook, but not 100% precise. **The pre-install and post-install hooks will become pre-sync and post-sync hooks**. **So they will be executed every sync**.

```
$ argocd repo add https://github.... --username github-serviceuser --password github-token
```



## Jam Projects

### `values.yaml`

```
jam:
  namespace: local700
```

### `Chart.yaml`

```
apiVersion: v1
appVersion: "1.0"
description: A Helm chart for jam-argocd-projects
name: jam-argocd-projects
version: 0.1.0
```

### `templates/projects.yaml`

```
{{range $project := tuple "logging" "argocd" .Values.jam.namespace}}
---
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: {{ $project }}
  namespace: argocd
spec:
  description: Project for {{ $project }}
  sourceRepos:
  - 'https://github....'
  destinations:
  - namespace: {{ $project }}
    server: https://kubernetes.default.svc
  clusterResourceWhitelist:
  - group: 'rbac.authorization.k8s.io'
    kind: ClusterRole
  - group: 'rbac.authorization.k8s.io'
    kind: ClusterRoleBinding
  - group: ''
    kind: Namespace
{{end}}
---
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: workzone-sealed-secrets
  namespace: argocd
spec:
  description: Project for Sync SealedSecrets
  sourceRepos:
  - 'https://github.tools.sap/sap-zone/jam-on-k8s'
  destinations:
  - namespace: '*'
    server: https://kubernetes.default.svc
  clusterResourceWhitelist:
  - group: 'bitnami.com/v1alpha1'
    kind: SealedSecret
```

**projects**:

* **logging**
* **argocd**
* **AppProject**
* **`.Values.jam.namespace`**: `dev902`

```
helm install argocd-projects helm/argocd/projects -f instances/$NSTANCE-k8s.yaml --namespace argocd
```

## applications


### `values.yaml`

```
jam:
  namespace: local700
```

### `Chart.yaml`

```
apiVersion: v1
appVersion: "1.0"
description: A Helm chart for jam-argocd-projects
name: jam-argocd-projects
version: 
```



### argocd application: `templates/argocd.yaml`

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd-projects
  namespace: argocd
spec:
  project: argocd
  source:
    repoURL: https://github...
    targetRevision: {{ $.Values.argocd.targetRevision }}
    path: helm/argocd/projects
    helm:
      valueFiles:
      - ../../../instances/{{ $.Values.jam.namespace }}-k8s.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: false
      selfHeal: false
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd-applications
  namespace: argocd
spec:
  project: argocd
  source:
    repoURL: https://...
    targetRevision: {{ $.Values.argocd.targetRevision }}
    path: helm/argocd/applications
    helm:
      valueFiles:
      - ../../../instances/{{ $.Values.jam.namespace }}-k8s.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: false
      selfHeal: false
```

* Application: `argocd-projects`
* Application: `argocd-applications`


### logging application: `templates/logging.yaml`

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: logging
  namespace: argocd
spec:
  project: logging
  source:
    repoURL: https://github...
    targetRevision: {{ $.Values.argocd.targetRevision }}
    path: helm/charts/logging
    helm:
      valueFiles:
      - ../../../instances/{{ $.Values.jam.namespace }}-k8s.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: logging
```

* Application: `logging `


### jam application: `templates/jam.yaml`


```
{{- $allApps := list "elasticsearch" "elasticsearch6" "mail" "memcached" "rabbitmq" "agent-server" "antivirus" "ct" "doc" "jod" "load-balancer" "mail-inbound"  "opensocial" "ps" }}
{{- $applyCdApps := list "agent-server" "antivirus" "ct" "doc" "jod" "load-balancer" "mail-inbound"  "opensocial" "ps" }}
{{range $application := $allApps}}
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: {{ $application }}
  namespace: argocd
{{- if has $application $applyCdApps  }}
  labels:
    apply-cd: "true"
{{- end }}
spec:
{{- if eq $application "ct"  }}
  ignoreDifferences:
{{- if or $.Values.jam.ctWebapp.minReplicas $.Values.jam.ctWebapp.maxReplicas }}
  - group: apps
    kind: Deployment
    name: ct-webapp
    jsonPointers:
    - /spec/replicas
{{- end }}
{{- if or $.Values.jam.ctWorker.minReplicas $.Values.jam.ctWorker.maxReplicas }}
  - group: apps
    kind: Deployment
    name: ct-worker
    jsonPointers:
    - /spec/replicas
{{- end }}
{{- end }}
  project: {{ $.Values.jam.namespace }}
  source:
    repoURL: https://github.tools.sap/sap-zone/jam-on-k8s
    targetRevision: {{ $.Values.argocd.targetRevision }}
    path: helm/jam/{{ $application }}
    helm:
      values: |
        argocd:
          runningInArgo: true
      valueFiles:
      - ../../../instances/{{ $.Values.jam.namespace }}-k8s.yaml
{{- if eq $.Values.argocd.autoSync true  }}
  syncPolicy:
    automated:
      prune: true
      selfHeal: false
{{- end }}
  destination:
    server: https://kubernetes.default.svc
    name
```

* **allApps:**
	* elasticsearch, elasticsearch6, mail, memcached, rabbitmq
	* agent-server, antivirus, ct, doc, jod, load-balancer
	* mail-inbound, opensocial, ps
* **applyCdApps:**
	* agent-server, antivirus, ct, doc, jod, load-balancer
	* mail-inbound, opensocial, ps
* **Excluede**
	*  mail, memcached, rabbitmq


### argocd application: `templates/sealded_secrets.yaml`

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sealed-secrets
  namespace: argocd
  labels:
    apply-cd: "true"
spec:
  project: workzone-sealed-secrets
  source:
    repoURL: https://github.tools.sap/sap-zone/jam-on-k8s
    targetRevision: {{ $.Values.argocd.targetRevision }}
    path: {{ $.Values.kustomize_path }}secrets
  syncPolicy:
    automated:
      prune: false
      selfHeal: false
  destination:
    server: https://kubernetes.default.svc
    namespace: '*'
{{- if eq $.Values.argocd.autoSync true  }}
  syncPolicy:
    automated:
      prune: false
      selfHeal: false
{{- end }}
```


	
### argocd jam application related value

```	
jam:
  namespace: local700
argocd:
  autoSync: false
  targetRevision: master

# Path for kustomize manifest of current instance, refer to exists ones.
kustomize_path:  kustomize/dev/{alicloud,aws,azure,gcp}/{6-9}{00-99}/ 
```


```
{{range $application := $allApps}}
	

{{- if has $application $applyCdApps  }}
labels:
	apply-cd: "true"
{{- end }}
	

{{- if eq $.Values.argocd.autoSync true  }}
  syncPolicy:
    automated:
      prune: false
      selfHeal: false
{{- end }}

{{end}}
```

### `argo_nonce.yaml`

```
argocd:
  nonce: '20200229000022'
```

### `current_release.yaml`

```
jam:
  release: RNumber
```

```
helm install argocd-applications helm/argocd/applications -f instances/$..-k8s.yaml --namespace argocd
```