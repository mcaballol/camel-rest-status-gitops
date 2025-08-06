# Explicación del Archivo policy.csv

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
