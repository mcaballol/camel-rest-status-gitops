# Guía de Despliegue de ArgoCD en OpenShift

## Tabla de Contenidos

- [Introducción](#introducción)
- [Prerrequisitos](#prerrequisitos)
- [Paso 1: Instalación del Operador de ArgoCD](#paso-1-instalación-del-operador-de-argocd)
- [Paso 2: Configuración del Namespace de GitOps](#paso-2-configuración-del-namespace-de-gitops)
- [Paso 3: Despliegue de la Instancia ArgoCD](#paso-3-despliegue-de-la-instancia-argocd)
- [Paso 4: Configuración de Acceso](#paso-4-configuración-de-acceso)
- [Paso 5: Configuración de Repositorios Git](#paso-5-configuración-de-repositorios-git)
- [Paso 6: Crear Primera Aplicación](#paso-6-crear-primera-aplicación)
- [Paso 7: Verificación y Monitoreo](#paso-7-verificación-y-monitoreo)
- [Paso 8: Configuraciones Avanzadas](#paso-8-configuraciones-avanzadas)
- [Paso 9: Configuración de Proyectos y ApplicationSets](#paso-9-configuración-de-proyectos-y-applicationsets)
- [Solución de Problemas](#solución-de-problemas)
- [Conclusión](#conclusión)
- [Referencias](#referencias)

## Introducción

Esta guía proporciona los pasos detallados para desplegar ArgoCD en un cluster de OpenShift, configurando GitOps para la gestión continua de aplicaciones.

## Prerrequisitos

### Herramientas Requeridas

- `oc` CLI configurado y conectado al cluster OpenShift
- `kubectl` CLI (opcional)
- Acceso administrativo al cluster OpenShift
- Git CLI configurado

### Verificaciones Previas

1. Verificar conectividad al cluster:

```bash
oc whoami
oc cluster-info
```

2. Verificar permisos de administrador:

```bash
oc auth can-i create projects
oc auth can-i create clusterroles
```

## Paso 1: Instalación del Operador de ArgoCD

### 1.1 Crear el Namespace

```bash
oc new-project openshift-gitops-operator
```

### 1.2 Instalar el Operador

1. Crear la suscripción al operador:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: latest
  installPlanApproval: Automatic
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

2. Aplicar la configuración:

```bash
oc apply -f openshift-gitops-subscription.yaml
```

### 1.3 Verificar la Instalación

```bash
oc get csv -n openshift-operators | grep gitops
oc get pods -n openshift-gitops
```

## Paso 2: Configuración del Namespace de GitOps

### 2.1 Crear Namespace para ArgoCD

```bash
oc new-project openshift-gitops
```

### 2.2 Configurar Permisos

1. Otorgar permisos de cluster-admin a ArgoCD:

```bash
oc adm policy add-cluster-role-to-user cluster-admin \
  system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller
```

## Paso 3: Despliegue de la Instancia ArgoCD

### 3.1 Crear la Instancia ArgoCD

1. Crear el archivo de configuración `argocd-instance.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: openshift-gitops
  namespace: openshift-gitops
spec:
  server:
    route:
      enabled: true
      tls:
        termination: reencrypt
  dex:
    openShiftOAuth: true
  rbac:
    defaultPolicy: 'role:readonly'
    policy: |
      g, system:cluster-admins, role:admin
    scopes: '[groups]'
```

2. Aplicar la configuración:

```bash
oc apply -f argocd-instance.yaml
```

### 3.2 Verificar el Despliegue

```bash
oc get argocd -n openshift-gitops
oc get pods -n openshift-gitops
oc get routes -n openshift-gitops
```

## Paso 4: Configuración de Acceso

### 4.1 Obtener la URL de ArgoCD

```bash
ARGOCD_URL=$(oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}')
echo "ArgoCD URL: https://$ARGOCD_URL"
```

### 4.2 Configurar Autenticación

#### Opción 1: OAuth de OpenShift (Recomendado)

La autenticación OAuth ya está configurada en la instancia. Los usuarios pueden acceder con sus credenciales de OpenShift.

#### Opción 2: Usuario Admin Local

1. Obtener la contraseña del admin:

```bash
oc get secret openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d
```

## Paso 5: Configuración de Repositorios Git

### 5.1 Agregar Repositorio Git

1. Acceder a la interfaz web de ArgoCD
2. Navegar a **Settings** → **Repositories**
3. Hacer clic en **Connect Repo**
4. Configurar:
   - **Type**: Git
   - **Repository URL**: URL del repositorio Git
   - **Username/Password** o **SSH Key** según corresponda

### 5.2 Configuración via CLI

```bash
argocd repo add https://github.com/tu-org/tu-repo.git \
  --username tu-usuario \
  --password tu-token
```

## Paso 6: Crear Primera Aplicación

### 6.1 Aplicación de Ejemplo

1. Crear `ejemplo-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ejemplo-app
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: https://github.com/tu-org/tu-repo.git
    targetRevision: main
    path: k8s/
  destination:
    server: https://kubernetes.default.svc
    namespace: ejemplo-namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

2. Aplicar la aplicación:

```bash
oc apply -f ejemplo-app.yaml
```

## Paso 7: Verificación y Monitoreo

### 7.1 Verificar Estado de la Aplicación

```bash
oc get applications -n openshift-gitops
oc describe application ejemplo-app -n openshift-gitops
```

### 7.2 Monitoreo via Web UI

1. Acceder a la URL de ArgoCD
2. Verificar el estado de sincronización
3. Revisar logs y eventos

## Paso 8: Configuraciones Avanzadas

### 8.1 Configurar Notificaciones

1. Crear ConfigMap para notificaciones:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: openshift-gitops
data:
  service.slack: |
    token: xoxb-your-slack-token
  template.app-deployed: |
    message: Application {{.app.metadata.name}} deployed successfully
```

### 8.2 Configurar RBAC Personalizado

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: openshift-gitops
data:
  policy.default: role:readonly
  policy.csv: |
    p, role:developer, applications, get, */*, allow
    p, role:developer, applications, sync, */*, allow
    g, developer-group, role:developer
```

#### Explicación del Archivo policy.csv

El archivo `policy.csv` define las políticas de control de acceso basado en roles (RBAC) para ArgoCD. Utiliza el formato de políticas de Casbin con la siguiente sintaxis:

**Tipos de Políticas:**

- **Políticas de Permisos (p)**: Definen qué acciones puede realizar un rol
- **Políticas de Agrupación (g)**: Asignan usuarios o grupos a roles

**Formato de Políticas de Permisos:**
```
p, <rol>, <recurso>, <acción>, <objeto>, <efecto>
```

**Parámetros explicados:**

- `<rol>`: El nombre del rol (ej: `role:developer`, `role:admin`)
- `<recurso>`: El tipo de recurso de ArgoCD (ej: `applications`, `repositories`, `clusters`)
- `<acción>`: La acción permitida (ej: `get`, `create`, `update`, `delete`, `sync`, `action`)
- `<objeto>`: El objeto específico usando wildcards (`*/*` = todos los proyectos/aplicaciones)
- `<efecto>`: `allow` o `deny`

**Ejemplos de Políticas Comunes:**

```
# Permitir a developers ver todas las aplicaciones
p, role:developer, applications, get, */*, allow

# Permitir sincronizar aplicaciones específicas del proyecto "dev"
p, role:developer, applications, sync, dev/*, allow

# Permitir crear aplicaciones solo en el proyecto "sandbox"
p, role:developer, applications, create, sandbox/*, allow

# Permitir ejecutar acciones en aplicaciones (restart, rollback, etc.)
p, role:developer, applications, action, */*, allow

# Denegar eliminación de aplicaciones
p, role:developer, applications, delete, */*, deny

# Permitir ver repositorios
p, role:developer, repositories, get, *, allow

# Permitir ver clusters
p, role:developer, clusters, get, *, allow
```

**Formato de Políticas de Agrupación:**
```
g, <usuario/grupo>, <rol>
```

**Ejemplos de Agrupación:**

```
# Asignar usuario específico a rol developer
g, juan.perez@empresa.com, role:developer

# Asignar grupo de OpenShift a rol admin
g, system:cluster-admins, role:admin

# Asignar grupo LDAP a rol developer
g, cn=developers,ou=groups,dc=empresa,dc=com, role:developer
```

**Recursos Disponibles en ArgoCD:**

- `applications` - Aplicaciones de ArgoCD
- `repositories` - Repositorios Git configurados
- `clusters` - Clusters de destino
- `projects` - Proyectos de ArgoCD
- `accounts` - Cuentas de usuario locales
- `gpgkeys` - Claves GPG para verificación
- `certificates` - Certificados TLS

**Acciones Disponibles:**

- `get` - Ver/listar recursos
- `create` - Crear nuevos recursos
- `update` - Modificar recursos existentes
- `delete` - Eliminar recursos
- `sync` - Sincronizar aplicaciones
- `action` - Ejecutar acciones específicas (restart, rollback, etc.)
- `override` - Sobrescribir configuraciones

## Paso 9: Configuración de Proyectos y ApplicationSets

### 9.1 Crear un Proyecto ArgoCD

Los proyectos en ArgoCD proporcionan una forma de agrupar aplicaciones y definir políticas específicas.

1. Crear el archivo `example-project.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: example-project
  namespace: openshift-gitops
spec:
  description: "Proyecto de ejemplo para aplicaciones GitOps"
  
  # Repositorios permitidos
  sourceRepos:
  - 'https://github.com/example/examplerepo-gitops.git'
  - 'https://github.com/example/*'
  
  # Clusters de destino permitidos
  destinations:
  - namespace: 'example-*'
    server: 'https://kubernetes.default.svc'
  - namespace: 'dev'
    server: 'https://kubernetes.default.svc'
  - namespace: 'staging'
    server: 'https://kubernetes.default.svc'
  
  # Recursos permitidos
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace
  - group: 'rbac.authorization.k8s.io'
    kind: ClusterRole
  - group: 'rbac.authorization.k8s.io'
    kind: ClusterRoleBinding
  
  # Recursos de namespace permitidos
  namespaceResourceWhitelist:
  - group: ''
    kind: ConfigMap
  - group: ''
    kind: Secret
  - group: ''
    kind: Service
  - group: 'apps'
    kind: Deployment
  - group: 'apps'
    kind: StatefulSet
  - group: 'route.openshift.io'
    kind: Route
  
  # Roles del proyecto
  roles:
  - name: developer
    description: "Rol para desarrolladores"
    policies:
    - p, proj:example-project:developer, applications, get, example-project/*, allow
    - p, proj:example-project:developer, applications, sync, example-project/*, allow
    - p, proj:example-project:developer, applications, action, example-project/*, allow
    groups:
    - example-developers
  
  - name: admin
    description: "Rol de administrador del proyecto"
    policies:
    - p, proj:example-project:admin, applications, *, example-project/*, allow
    - p, proj:example-project:admin, repositories, *, *, allow
    groups:
    - example-admins
```

2. Aplicar el proyecto:

```bash
oc apply -f example-project.yaml
```

3. Verificar la creación del proyecto:

```bash
oc get appproject -n openshift-gitops
oc describe appproject example-project -n openshift-gitops
```

### 9.2 Configurar ApplicationSet

Los ApplicationSets permiten crear y gestionar múltiples aplicaciones ArgoCD de forma automatizada.

#### 9.2.1 ApplicationSet Básico

1. Crear el archivo `example-applicationset.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: example-apps
  namespace: openshift-gitops
spec:
  generators:
  - git:
      repoURL: https://github.com/example/examplerepo-gitops.git
      revision: HEAD
      directories:
      - path: environments/*
  
  template:
    metadata:
      name: '{{path.basename}}-app'
      labels:
        environment: '{{path.basename}}'
    spec:
      project: example-project
      source:
        repoURL: https://github.com/example/examplerepo-gitops.git
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
        - PrunePropagationPolicy=foreground
        - PruneLast=true
```

#### 9.2.2 ApplicationSet con Múltiples Entornos

1. Crear un ApplicationSet más avanzado `multi-env-applicationset.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: multi-env-example
  namespace: openshift-gitops
spec:
  generators:
  - list:
      elements:
      - cluster: dev
        url: https://kubernetes.default.svc
        namespace: example-dev
        branch: develop
      - cluster: staging
        url: https://kubernetes.default.svc
        namespace: example-staging
        branch: main
      - cluster: prod
        url: https://kubernetes.default.svc
        namespace: example-prod
        branch: main
  
  template:
    metadata:
      name: 'example-{{cluster}}'
      labels:
        environment: '{{cluster}}'
    spec:
      project: example-project
      source:
        repoURL: https://github.com/example/examplerepo-gitops.git
        targetRevision: '{{branch}}'
        path: 'overlays/{{cluster}}'
      destination:
        server: '{{url}}'
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
        retry:
          limit: 5
          backoff:
            duration: 5s
            factor: 2
            maxDuration: 3m
```

2. Aplicar el ApplicationSet:

```bash
oc apply -f example-applicationset.yaml
# o para el multi-entorno
oc apply -f multi-env-applicationset.yaml
```

### 9.3 Estructura del Repositorio GitOps de Ejemplo

Para que funcione correctamente, el repositorio `https://github.com/example/examplerepo-gitops.git` debe tener una estructura como la siguiente:

```
examplerepo-gitops/
├── README.md
├── environments/
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   └── app-config.yaml
│   ├── staging/
│   │   ├── kustomization.yaml
│   │   └── app-config.yaml
│   └── prod/
│       ├── kustomization.yaml
│       └── app-config.yaml
├── overlays/
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   ├── deployment-patch.yaml
│   │   └── configmap.yaml
│   ├── staging/
│   │   ├── kustomization.yaml
│   │   ├── deployment-patch.yaml
│   │   └── configmap.yaml
│   └── prod/
│       ├── kustomization.yaml
│       ├── deployment-patch.yaml
│       └── configmap.yaml
└── base/
    ├── kustomization.yaml
    ├── deployment.yaml
    ├── service.yaml
    └── configmap.yaml
```

### 9.4 Verificación y Monitoreo

#### 9.4.1 Verificar ApplicationSets

```bash
# Listar ApplicationSets
oc get applicationset -n openshift-gitops

# Ver detalles del ApplicationSet
oc describe applicationset example-apps -n openshift-gitops

# Ver aplicaciones generadas
oc get applications -n openshift-gitops -l argocd.argoproj.io/application-set-name=example-apps
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

### 9.5 Configuración de Webhook (Opcional)

Para sincronización automática cuando hay cambios en el repositorio:

1. Configurar webhook en GitHub:
   - URL: `https://<argocd-url>/api/webhook`
   - Content type: `application/json`
   - Events: `push`, `pull_request`

2. Crear secret para webhook:

```bash
oc create secret generic webhook-secret \
  --from-literal=webhook.github.secret=your-webhook-secret \
  -n openshift-gitops
```

### 9.6 Comandos Útiles para Gestión

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

## Solución de Problemas

### Problemas Comunes

#### ArgoCD no puede sincronizar aplicaciones

1. Verificar permisos del service account
2. Revisar conectividad al repositorio Git
3. Validar la configuración RBAC

#### Aplicaciones en estado "OutOfSync"

1. Verificar diferencias en la interfaz web
2. Ejecutar sincronización manual
3. Revisar logs de ArgoCD

### Comandos de Diagnóstico

```bash
# Verificar estado de pods
oc get pods -n openshift-gitops

# Revisar logs
oc logs -f deployment/openshift-gitops-server -n openshift-gitops

# Verificar configuración
oc get configmap argocd-cm -n openshift-gitops -o yaml
```

## Conclusión

Con estos pasos, ArgoCD estará completamente desplegado y configurado en tu cluster OpenShift, listo para gestionar aplicaciones mediante GitOps.

### Próximos Pasos

- Configurar pipelines CI/CD
- Implementar estrategias de despliegue avanzadas
- Configurar monitoreo y alertas
- Establecer políticas de governance

## Referencias

- [Documentación oficial de ArgoCD](https://argo-cd.readthedocs.io/)
- [OpenShift GitOps Documentation](https://docs.openshift.com/container-platform/latest/cicd/gitops/understanding-openshift-gitops.html)
- [ArgoCD Operator Manual](https://argoproj.github.io/argo-cd/operator-manual/)
- [ApplicationSet Documentation](https://argocd-applicationset.readthedocs.io/)