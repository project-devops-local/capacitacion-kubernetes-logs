## Control de Acceso Basado en Roles (RBAC) en Kubernetes

Kubernetes RBAC (Role‑Based Access Control) permite regular las operaciones que los usuarios, grupos o ServiceAccounts pueden realizar sobre los recursos del clúster mediante objetos declarativos en la API `rbac.authorization.k8s.io`.

---

### Objetivos de esta guía

1. **Comprender los componentes clave** (`Role`, `ClusterRole`, `RoleBinding`, `ClusterRoleBinding`).
2. **Conocer el listado completo de *verbs*** que se pueden asignar en reglas RBAC.
3. **Aplicar buenas prácticas** de diseño de permisos siguiendo el principio de **menor privilegio**.
4. **Realizar ejercicios prácticos** para reforzar la teoría y validar configuraciones con `kubectl`.

---

### Componentes principales

| Objeto                 | Alcance   | Propósito                                                                                                   |
| ---------------------- | --------- | ----------------------------------------------------------------------------------------------------------- |
| **Role**               | Namespace | Agrupa reglas de acceso para recursos en un único *namespace*.                                              |
| **ClusterRole**        | Cluster   | Igual que `Role`, pero sus reglas se aplican a todos los *namespaces* o a recursos *cluster‑scoped*.        |
| **RoleBinding**        | Namespace | Asocia un `Role` o `ClusterRole` con sujetos (usuarios, grupos o ServiceAccounts) dentro de un *namespace*. |
| **ClusterRoleBinding** | Cluster   | Asocia un `ClusterRole` con sujetos a nivel global (todos los *namespaces*).                                |

> **Tip:** Prefiere `Role` + `RoleBinding` frente a privilegios de clúster completos, a menos que un servicio realmente requiera acceso inter‑namespace.

---

### Lista completa de *verbs* disponibles

Los *verbs* definen la **acción** autorizada sobre el recurso. A partir de Kubernetes v1.33 (abril 2025) los *verbs* reconocidos son:

| Categoría                                                        | Verbs                             | Descripción breve                                                                                      |
| ---------------------------------------------------------------- | --------------------------------- | ------------------------------------------------------------------------------------------------------ |
| **Wildcard**                                                     | `*`                               | Autoriza **todas** las operaciones en los recursos especificados. ¡Úsalo con cautela!                  |
| **Lectura**                                                      | `get`, `list`, `watch`            | Obtener un objeto, listar colecciones y recibir eventos en tiempo real.                                |
| **Creación / Modificación**                                      | `create`, `update`, `patch`       | Crear objetos; reemplazar o modificar parcialmente los existentes.                                     |
| **Eliminación**                                                  | `delete`, `deletecollection`      | Borrar un objeto o una colección completa.                                                             |
| **Conectividad**                                                 | `connect`                         | Abrir un canal (exec/attach/port‑forward/proxy) hacia un subrecurso.                                   |
| **Gestión de certificados**                                      | `approve`, `deny`                 | Aprobar o denegar `CertificateSigningRequests`.                                                        |
| **Privilegios avanzados**                                        | `bind`, `escalate`, `impersonate` |                                                                                                        |
|   • `bind`: crear bindings a roles que el usuario no posee.      |                                   |                                                                                                        |
|   • `escalate`: crear/actualizar roles con permisos superiores.  |                                   |                                                                                                        |
|   • `impersonate`: suplantar usuarios, grupos o ServiceAccounts. |                                   |                                                                                                        |
| **Políticas / Uso**                                              | `use`                             | Consumir ciertos recursos de política (por ejemplo `PodSecurityPolicy`, `SecurityContextConstraints`). |

> **Nota:** Si especificas `*` en `verbs`, se otorgan automáticamente todos los verbos anteriores para los recursos indicados.

---

### Buenas prácticas de diseño de roles

1. **Principio de menor privilegio**: concede solo los *verbs* y recursos indispensables.
2. **Evita comodines a nivel de clúster** (`resources: ["*"]`, `verbs: ["*"]`).
3. **Segmenta por namespaces**: usa `Role` + `RoleBinding` cuando sea posible.
4. **Revisa los *verbs* peligrosos** (`bind`, `escalate`, `impersonate`) y limítalos a operadores de plataforma.
5. **Audita periódicamente** con:

```bash
kubectl auth can-i delete pods --namespace=dev --as=vaian
```

6. **Agrega anotaciones** y labels descriptivos (`rbac.authorization.k8s.io/aggregate-to-view: "true"`) para facilitar futuras agregaciones de roles.

---



### Auditoría y *troubleshooting*

* **Simular peticiones** con `kubectl auth can-i` para verificar permisos antes de aplicarlos.
* **Logs del API Server** con nivel `--v=5` muestran eventos `RBAC DENIED`.
* **Herramientas externas**: Políticas OPA Gatekeeper, Kyverno, rbac‑manager.

---

### Ejercicios prácticos sugeridos

1. **Crear un espacio de nombres** `dev` y conceder a un ServiceAccount acceso solo para crear ConfigMaps.
2. **Configurar un rol de solo lectura** para un analista que necesite consultar logs (`pods/log`) en el *namespace* `prod`.
3. **Reproducir un escenario de escalamiento de privilegios** intentando crear un `RoleBinding` sin `bind` y observar la denegación.
4. **Aplicar un rol agregado** que extienda el rol `view` para incluir lectura de un CRD personalizado.

---



# Laboratorio Kubernetes – Guía Rápida

> Sustituye los valores entre **<…>** por los nombres reales que uses.
> Ejemplo: `NAMESPACE=dev`, `SERVICE_ACCOUNT=demo-permissions-dev`, etc.

---

## 1. Verificar permisos con `kubectl auth can‑i`

### ◼︎ Como **ServiceAccount**

```bash
kubectl auth can-i get <RESOURCE> \
  --namespace=<NAMESPACE> \
  --as=system:serviceaccount:<NAMESPACE>:<SERVICE_ACCOUNT>
```

### ◼︎ Como **usuario normal**

```bash
kubectl auth can-i get <RESOURCE> \
  --namespace=<NAMESPACE> \
  --as=<USER>
```

`<RESOURCE>` suele ser `pods`, `deployments`, `services`, etc.

---

## 2. Desplegar manifiestos

```bash
kubectl apply -f manifest.yaml
```

*Coloca tus recursos Kubernetes en `manifest.yaml` o divide el despliegue en múltiples archivos según sea necesario.*

---

## 3. Consultas rápidas de clúster

| Acción                              | Comando                                             |
| ----------------------------------- | --------------------------------------------------- |
| Listar todos los **namespaces**     | `kubectl get ns`                                    |
| Ver **pods** en un namespace        | `kubectl get pods -n <NAMESPACE>`                   |
| Ver **deployments** en un namespace | `kubectl get deployments -n <NAMESPACE>`            |
| Ver **services** en un namespace    | `kubectl get svc -n <NAMESPACE>`                    |
| Describir un recurso                | `kubectl describe <RESOURCE>/<NAME> -n <NAMESPACE>` |

---

## 4. Variables de entorno útiles

Para evitar repetir parámetros puedes exportar variables con los valores que más usas:

```bash
export NAMESPACE=dev
export SERVICE_ACCOUNT=demo-permissions-dev
export RESOURCE=pods

# Ejemplo de uso
kubectl auth can-i get "$RESOURCE" \
  --namespace="$NAMESPACE" \
  --as="system:serviceaccount:$NAMESPACE:$SERVICE_ACCOUNT"
```

Guarda estas líneas en tu `~/.bashrc` (o equivalente) si las necesitas en cada sesión.

---

## 5. Ejemplo completo: lectura de pods y roles

```bash
# Comprobar si el SA puede listar pods
kubectl auth can-i list pods \
  --namespace=dev \
  --as=system:serviceaccount:dev:demo-permissions-dev

# Comprobar si el SA puede listar roles (mismo namespace)
kubectl auth can-i list roles \
  --namespace=dev \
  --as=system:serviceaccount:dev:demo-permissions-dev
```

---

## 6. Referencias

* [Documentación oficial de RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

