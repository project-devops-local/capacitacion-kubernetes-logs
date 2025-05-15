# Guía práctica: Clúster local con **KIND** y NetworkPolicy &mdash; + notas para AKS

> Este README te guía para levantar un laboratorio Kubernetes local con KIND, habilitar **NetworkPolicy** usando **Calico**, y explica **cómo se habilitan las NetworkPolicy en AKS** y qué CNI se emplea allí.

---

## 0. Prerrequisitos
| Herramienta | Versión mínima | Notas |
|-------------|----------------|-------|
| **Docker** / Containerd | 20.10 + | KIND crea los “nodos” como contenedores Docker. |
| **kubectl** | similar a la versión K8s | CLI para interactuar con el clúster. |
| **curl / wget** | &mdash; | Descarga de binarios. |

---

## 1. Instalación del binario **kind**

### Linux AMD64 / x86‑64
```bash
[ "$(uname -m)" = x86_64 ] && \
  curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.27.0/kind-linux-amd64 && \
  chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind
```
> Comprueba la instalación con `kind --version`.

## 2. Crear un clúster KIND

> Necesitas Docker corriendo.

```bash
kind create cluster --config kind-config.yaml --name demo
```
Ejemplo mínimo de **kind-config.yaml** (control‑plane + 2 workers):
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: demo
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

Valida que el clúster esté activo:
```bash
kubectl cluster-info --context kind-demo
kubectl get nodes -o wide
```

---

## 3. Habilitar **NetworkPolicy** en KIND (instalar Calico)
KIND usa **kindnet**, que *no* implementa NetworkPolicy. Para practicar políticas:
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```
Esto despliega:
* `calico-kube-controllers`
* `calico-node-*` (uno por nodo)

### Verificación
```bash
kubectl get pods -n kube-system -l k8s-app=calico-node
```
Todos deben estar en estado **Running/Ready**:
```
NAME                  READY   STATUS    RESTARTS   AGE
calico-node-9h77d     1/1     Running   0          3d
calico-node-f2c9s     1/1     Running   0          3d
```
A partir de aquí las `NetworkPolicy` se aplican normalmente.

---

## 4. NetworkPolicy en **Azure Kubernetes Service (AKS)**

### 4.1 Cómo habilitarla
Al **crear** el clúster debes añadir **`--network-policy calico`**; no se puede habilitar después.
```bash
az aks create \
  --resource-group myRG \
  --name myCluster \
  --node-count 3 \
  --network-plugin azure \   # o kubenet
  --network-policy calico
```
> Si ya tienes un clúster existente sin NetworkPolicy, es necesario recrearlo.

### 4.2 ¿Qué CNI usa AKS cuando activas NetworkPolicy?
| Concepto | Detalle |
|----------|---------|
| **Proveedor de NetworkPolicy** | **Calico** ➜ se despliega automáticamente en `kube-system` como DaemonSet. |
| **Plugin de red primario** | **Depende de la bandera `--network-plugin`:**  
- `azure` → **Azure CNI** (asigna IP VNet a cada Pod).  
- `kubenet` → **kubenet** (bridge Linux + rutas UDR). |
| **Resultado práctico** | El CNI principal *sigue* siendo `azure` o `kubenet`; Calico solo intercepta el tráfico para aplicar las reglas **Ingress/Egress**. |

### 4.3 Comprobación rápida en AKS
```bash
az aks show -g myRG -n myCluster --query networkProfile
# → { "networkPlugin": "azure", "networkPolicy": "calico", ... }

kubectl get pods -n kube-system -l k8s-app=kube-policy-controller
kubectl get daemonset -n kube-system calico-node
```

---

## 5. Preguntas de repaso rápido
| Pregunta | Respuesta |
|----------|-----------|
| ¿Qué ocurre si instalas una NetworkPolicy sin Calico en KIND? | No tiene efecto; kindnet no la implementa. |
| ¿Qué proveedor aplica NetworkPolicy en AKS? | Calico (se despliega automáticamente). |
| ¿Puedo habilitar NetworkPolicy en AKS después? | No; exige recrear el clúster. |
| ¿`podSelector: {}` qué significa? | Selecciona **todos** los Pods del namespace. |

---

## 6. Limpieza
```bash
kind delete cluster --name demo
```

---

### Créditos y referencias
- KIND docs: <https://kind.sigs.k8s.io>
- Calico docs: <https://docs.tigera.io>
- AKS Network Policy: <https://learn.microsoft.com/azure/aks/use-network-policies>
