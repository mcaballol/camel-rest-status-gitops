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
oc create ns openshift-gitops-operator
```

### 1.2 Instalar el Operador
1. Crear el OperatorGroup "gitops-operator-group.yaml":


```yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-gitops-operator
  namespace: openshift-gitops-operator
spec:
  upgradeStrategy: Default
```
Aplicar la configuración:

```bash
oc apply -f gitops-operator-group.yaml
```

2. Crear la suscripción al operador "openshift-gitops-subscription.yaml":

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
Aplicar la configuración:

```bash
oc apply -f openshift-gitops-subscription.yaml
```


## Paso 2: Configuración del Namespace de GitOps

### 2.1 Configurar Permisos

1. Otorgar permisos de cluster-admin a ArgoCD:

```bash
oc adm policy add-cluster-role-to-user cluster-admin \
  system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller
```

## Paso 3: Despliegue de la Instancia ArgoCD

### 3.1 Crear la Instancia ArgoCD

1. Crear/Modifica el archivo de configuración `argocd-instance.yaml`:

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
  applicationSet:
    webhookServer:
      ingress:
        enabled: false
      route:
        enabled: false
  rbac:
    defaultPolicy: 'role:readonly'
    policy: |
      g, system:cluster-admins, role:admin
      g, cluster-admins, role:admin
    scopes: '[groups]'
```
Aplicar la configuración:

```bash
oc apply -f argocd-instance.yaml
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


## Paso 8: Configuraciones Avanzadas

### 8.1 Configurar RBAC Personalizado

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
[detalle policy.csv](detalle_policy_csv.md)

