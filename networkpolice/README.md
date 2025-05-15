
# Network Policies in Kubernetes

## 1. ¿Qué es una **NetworkPolicy**?
Una **NetworkPolicy** es un objeto (`apiVersion: networking.k8s.io/v1`, `kind: NetworkPolicy`) que define reglas de **capa 3/4** (IP, puerto y protocolo) para decidir **qué Pods pueden comunicarse entre sí y en qué dirección** (ingress / egress), tanto dentro como fuera del clúster.

---

## 2. Requisitos previos
- Las NetworkPolicies **solo surten efecto** si el clúster utiliza un **CNI** que implemente esta funcionalidad. Si tu proveedor de red no la soporta, el YAML "no hace nada".
- Ejemplos de CNIs compatibles (según la documentación oficial): **Calico, Antrea, Cilium, Kube‑router, Weave Net**.  
  - En **AKS**, la bandera `--network-policy calico` activa Calico al crear el clúster.

---

## 3. Comportamiento por defecto (pregunta frecuente en exámenes)
- **Sin policies**: todos los Pods están *no‑aislados* → todo el tráfico está permitido.
- **Con al menos una policy** que seleccione un Pod y un `policyType` dado: ese Pod queda **aislado** en esa dirección; **solo** el tráfico que coincida con alguna regla se permite.
- Las políticas son **aditivas**: las reglas permitidas se **unen**; nunca se niega explícitamente el tráfico.

---

## 4. Anatomía clave de `spec`
```yaml
spec:
  podSelector: {}            # ¿A qué Pods se aplica? '{}' = todos en el namespace
  policyTypes: [Ingress, Egress]
  ingress:
    - from:                  # 'from' / 'to' comparten la misma sintaxis
        - podSelector:
        - namespaceSelector:
        - ipBlock: { cidr: 10.0.0.0/8, except: [10.1.0.0/16] }
      ports:
        - protocol: TCP
          port: 80
  egress:
    - to:
        - ipBlock: { cidr: 0.0.0.0/0 }
      ports:
        - protocol: TCP
          port: 443
```
- `podSelector: {}` → selecciona **todos los Pods** del namespace.
- Si `policyTypes` se omite, Kubernetes asume **Ingress**; añade **Egress** automáticamente si hay reglas egress.
- `ipBlock` utiliza rangos **CIDR** y puede incluir una lista `except`.

---

## 5. Reglas de evaluación
1. **Egress**: la policy del **Pod origen** debe permitir la conexión.
2. **Ingress**: la policy del **Pod destino** debe permitir la conexión.

Si cualquiera de las dos la bloquea, **el tráfico se corta**.

---

## 6. Limitaciones actuales de la API
- No existen reglas *deny* explícitas (modelo "deny‑all + allows aditivos").
- No se puede filtrar por nombre de *Service* ni aplicar TLS/L7 (usa una malla de servicio si lo necesitas).
- No hay políticas globales "por defecto" para todos los namespaces (salvo extensiones específicas del CNI).
- El comportamiento con Pods que usan `hostNetwork` varía según el CNI.

---

## 7. Flujo típico para el examen
### 7.1 Verificar soporte de NetworkPolicy
```bash
kubectl get pods -n kube-system | egrep 'calico|cilium|antrea|weave|kube-router'
```

### 7.2 Crear una política "deny‑all" y excepciones
```bash
# Aísla todos los Pods del namespace
kubectl label ns app-ns purpose=restricted

cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: app-ns
spec:
  podSelector: {}
  policyTypes:
    - Ingress
EOF

# Permitir solo tráfico desde el namespace 'frontend'
kubectl apply -f allow-from-frontend.yaml
```

---

## 8. Preguntas de repaso rápido
| Pregunta | Respuesta corta |
|----------|-----------------|
| **¿Qué ocurre si aplicas una policy sin CNI compatible?** | Ningún efecto. |
| **¿Qué significa `podSelector: {}`?** | Selecciona todos los Pods del namespace. |
| **¿Se pueden mezclar Ingress y Egress en una sola policy?** | Sí, usando `policyTypes`. |
| **¿Las policies se ordenan?** | No: son aditivas, el orden no importa. |
| **¿Cómo aislar solo Egress?** | Incluye `Egress` en `policyTypes` y deja `Ingress` fuera. |

---

## 9. Regla nemotécnica
> **Selector → Tipo → Dirección → Permiso**  
> 1) Selecciona el **Pod**.  
> 2) Define el **tipo** (Ingress/Egress).  
> 3) Indica la **dirección** (from/to).  
> 4) Especifica **quién/qué/puerto** se permite.

Con este esquema y los conceptos anteriores, estarás listo para responder sin fallar las preguntas de NetworkPolicy en el examen KCNC/KCNA.
