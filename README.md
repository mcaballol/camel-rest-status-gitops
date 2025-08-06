# Guía de Despliegue de ArgoCD en OpenShift

- [Habilitacion ArgoCD](doc/habilitar_argocd.md)
- [Configuración de Proyectos y ApplicationSets](#configuración-de-proyectos-y-applicationsets)
- [Solución de Problemas](#solución-de-problemas)
- [Conclusión](#conclusión)
- [Referencias](#referencias)



## Configuración de Proyectos y ApplicationSets

### Crear un Proyecto ArgoCD

Los proyectos en ArgoCD proporcionan una forma de agrupar aplicaciones y definir políticas específicas.

1. Crear el archivo `appproject.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: demo-appproject
  namespace: openshift-gitops
spec:
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
  destinations:
    - namespace: '*'
      server: '*'
  sourceRepos:
    - '*'

```

2. Aplicar el proyecto:

```bash
oc apply -f demo-appproject.yaml
```

3. Verificar la creación del proyecto:

```bash
oc get appproject -n openshift-gitops
oc describe appproject example-project -n openshift-gitops
```

### 9.2 Configurar ApplicationSet

Los ApplicationSets permiten crear y gestionar múltiples aplicaciones ArgoCD de forma automatizada.

#### 9.2.1 ApplicationSet

1. Crear el archivo `applicationset.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: demo-appplicationset
  namespace: openshift-gitops
spec:
  generators:
    - list:
        elements:
          - name: dev
            namespace: camel-dev
            path: kustomize/overlays/dev
          - name: prod
            namespace: camel-prod
            path: kustomize/overlays/prod
  template:
    metadata:
      name: 'camel-rest-status-{{name}}'
    spec:
      project: demo-appproject
      source:
        repoURL: https://github.com/mcaballol/camel-rest-status-gitops.git  # Cambia esto a tu repo real
        targetRevision: main
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

2. Aplicar el ApplicationSet:

```bash
oc apply -f applicationset.yaml
```

### 9.3 Estructura del Repositorio GitOps

Para que funcione correctamente, el repositorio `https://github.com/example/camel-rest-status-gitops.git` debe tener una estructura como la siguiente:

```
camel-rest-status-gitops/
camel-rest-status-gitops/
├── README.md                           # Documentación principal (717 líneas)
├── .git/                               # Control de versiones Git
└── kustomize/                          # Configuración de Kustomize
    ├── applicationset.yaml             # Definición del ApplicationSet de ArgoCD
    ├── appproject.yaml                 # Configuración del proyecto de ArgoCD
    ├── base/                           # Configuración base de Kubernetes
    │   ├── kustomization.yaml          # Archivo de configuración de Kustomize base
    │   ├── deployment.yaml             # Definición del Deployment
    │   ├── service.yaml                # Definición del Service
    │   └── ingress.yaml                # Definición del Ingress
    └── overlays/                       # Configuraciones específicas por entorno
        ├── dev/                        # Configuración para desarrollo
        │   ├── kustomization.yaml      # Kustomize para dev
        │   └── patch-deployment.yaml   # Parches específicos para dev
        └── prod/                       # Configuración para producción
            ├── kustomization.yaml      # Kustomize para prod
            └── patch-deployment.yaml   # Parches específicos para prod
```

### 9.4 Verificación y Monitoreo

#### 9.4.1 Verificar ApplicationSets

```bash
# Listar ApplicationSets
oc get applicationset -n openshift-gitops

# Ver detalles del ApplicationSet
oc describe applicationset demo-appplicationset -n openshift-gitops

# Ver aplicaciones generadas
oc get applications -n openshift-gitops -l argocd.argoproj.io/application-set-name=demo-appplicationset
```

#### 9.4.2 Monitorear Aplicaciones Generadas

```bash
# Ver estado de todas las aplicaciones
oc get applications -n openshift-gitops

# Ver aplicaciones por etiqueta de entorno
oc get applications -n openshift-gitops -l environment=dev

# Verificar sincronización
argocd app list --project example-project
```

### 9.5 Comandos Útiles para Gestión

```bash
# Sincronizar todas las aplicaciones de un proyecto
argocd app sync -l argocd.argoproj.io/project=example-project

# Refrescar ApplicationSet
argocd appset get example-apps --refresh

# Ver logs del ApplicationSet controller
oc logs -f deployment/openshift-gitops-applicationset-controller -n openshift-gitops

# Eliminar aplicaciones generadas por ApplicationSet
oc delete applications -l argocd.argoproj.io/application-set-name=example-apps -n openshift-gitops
```


## Referencias

- [Documentación oficial de ArgoCD](https://argo-cd.readthedocs.io/)
- [OpenShift GitOps Documentation](https://docs.openshift.com/container-platform/latest/cicd/gitops/understanding-openshift-gitops.html)
- [ArgoCD Operator Manual](https://argoproj.github.io/argo-cd/operator-manual/)
- [ApplicationSet Documentation](https://argocd-applicationset.readthedocs.io/)
- [Red Hat Openshift Gitops](https://docs.redhat.com/en/documentation/red_hat_openshift_gitops/1.16/html/installing_gitops/installing-openshift-gitops#installing-gitops-operator-using-cli_installing-openshift-gitops)