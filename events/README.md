# Laboratorio Kubernetes – Eventos, Estados y RBAC

Guía de **capacitación práctica** orientada al examen **KCNA (Kubernetes and Cloud‑Native Associate)**.  Incluye ejemplos de fallos habituales, comandos para depuración y verificación de permisos.

---

## Tabla de contenido

1. [Requisitos previos](#requisitos-previos)
2. [Conceptos básicos de Eventos](#conceptos-básicos-de-eventos)
3. [Comandos rápidos](#comandos-rápidos)
4. [Laboratorios paso a paso](#laboratorios-paso-a-paso)

   * 4.1 [Evento Normal — *Scheduled*](#lab-1--evento-normal-scheduled)
   * 4.2 [Fallo *ImagePullBackOff*](#lab-2--imagepullbackoff)
   * 4.3 [Fallo *CrashLoopBackOff*](#lab-3--crashloopbackoff)
   * 4.4 [Fallo *OOMKilled*](#lab-4--oomkilled)
   * 4.5 [Filtrado avanzado de eventos](#lab-5--filtrado-de-eventos)

---


## Conceptos básicos de Eventos

| Campo                    | Descripción                                                               |
| ------------------------ | ------------------------------------------------------------------------- |
| **Type**                 | Severidad del evento: `Normal` (flujo esperado) o `Warning` (anomalía)    |
| **Reason**               | Etiqueta corta que resume la causa (`Scheduled`, `BackOff`, `OOMKilled`…) |
| **Message**              | Texto legible con detalles adicionales                                    |
| **InvolvedObject**       | Recurso afectado (Pod, Node, PVC, etc.)                                   |
| **Source**               | Componente que generó el evento (kubelet, scheduler, controller)          |
| **FirstSeen / LastSeen** | Primer y último instante en que ocurrió                                   |

Los Eventos son objetos `kind: Event` dentro del *API group core* (`apiVersion: v1`).  Se listan con `kubectl get events`.

---

## Comandos rápidos

| Acción                                             | Comando                                                                                                          |                          |           |
| -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- | ------------------------ | --------- |
| Listar eventos de un namespace (orden cronológico) | `kubectl get events -n <NS> --sort-by=.metadata.creationTimestamp`                                               |                          |           |
| Ver *todos* los eventos del clúster                | `kubectl get events -A`                                                                                          |                          |           |
| Filtrar por severidad `Warning`                    | `kubectl get events -A --field-selector type=Warning`                                                            |                          |           |
| Ver eventos de un Pod                              | `kubectl describe pod <POD> -n <NS>`                                                                             |                          |           |
| Motivos (Reason) únicos                            | \`kubectl get events -A -o json                                                                                  | jq -r '.items\[].reason' | sort -u\` |
| Último evento de un recurso                        | \`kubectl get events --field-selector involvedObject.name=<OBJETO> -n <NS> --sort-by=.metadata.creationTimestamp | tail -1\`                |           |

---

## Laboratorios paso a paso

Cada laboratorio despliega un Pod que produce un estado\*; observa los **eventos** y lee la explicación del fallo.

> ℹ️ *Estado* en Kubernetes se ve en la columna **STATUS** de `kubectl get pods` y suele reflejar el `Reason` del último evento o de `containerStatuses`.

### Lab 1 · Evento Normal — *Scheduled*

**Manifiesto** (`busybox-ok.yaml`):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-ok
  namespace: dev
spec:
  containers:
  - name: bb
    image: busybox
    command: ["sh", "-c", "echo Hello && sleep 30"]
  restartPolicy: Never
```

```bash
kubectl apply -f busybox-ok.yaml
kubectl get events -n events --sort-by=.metadata.creationTimestamp | tail -5
```

**Explicación del estado**

| Evento (Reason)              | Tipo        | Significado                                               |
| ---------------------------- | ----------- | --------------------------------------------------------- |
| `Scheduled`                  | Normal      | El scheduler asignó el Pod a un nodo.                     |
| `Pulled / Created / Started` | Normal      | kubelet descargó la imagen, creó y arrancó el contenedor. |
| **Estado final**             | `Completed` | El comando finalizó con código `0`; no hay error.         |

---

### Lab 2 · *ImagePullBackOff*

**Manifiesto** (`bad-image.yaml`):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bad-image
  namespace: events
spec:
  containers:
  - name: bad
    image: busybox:notreal          # etiqueta inexistente
    command: ["sh", "-c", "sleep 3600"]
```

```bash
kubectl apply -f bad-image.yaml
kubectl get pod bad-image -n dev -w &
```

**Explicación del estado**

| Evento (Reason)  | Tipo               | Significado                                                                   |
| ---------------- | ------------------ | ----------------------------------------------------------------------------- |
| `Failed`         | Warning            | kubelet intenta `Pull` y Docker/CRI devuelve error 404.                       |
| `BackOff`        | Warning            | kubelet espera incrementalmente antes de reintentar la descarga.              |
| **Estado final** | `ImagePullBackOff` | El Pod no puede iniciar porque la imagen no existe o las credenciales fallan. |

---

### Lab 3 · *CrashLoopBackOff*

**Manifiesto** (`crash-loop.yaml`):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: crash-loop
  namespace: events
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "exit 1"]
```

```bash
kubectl get pod crash-loop -n events -w &
```

**Explicación del estado**

| Detector                | Valor              | Descripción                                                                |
| ----------------------- | ------------------ | -------------------------------------------------------------------------- |
| Evento `Killing`        | Warning            | kubelet mata el contenedor que salió con código distinto de `0`.           |
| Contador `RestartCount` | incrementa         | Cada reinicio aumenta el contador.                                         |
| **Estado final**        | `CrashLoopBackOff` | Exceso de fallos rápidos ⇒ kubelet aplica back‑off antes de nuevo intento. |

---

### Lab 4 · *OOMKilled*

**Manifiesto** (`oom.yaml`):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: oom-pod
  namespace: dev
spec:
  containers:
  - name: memhog
    image: polinux/stress
    args: ["--vm", "1", "--vm-bytes", "256M", "--vm-hang", "1"]
    resources:
      limits:
        memory: "64Mi"   # límite intencionalmente bajo
```

```bash

kubectl get pod oom-pod -n events -w
```

**Explicación del estado**

| Campo                                  | Valor                            |                                                                            |
| -------------------------------------- | -------------------------------- | -------------------------------------------------------------------------- |
| Evento `Killing` + message `OOMKilled` | Warning                          | El contenedor excede límite → kernel OOM da kill, kubelet registra evento. |
| ContainerState `terminated.reason`     | `OOMKilled`                      | Lo verás en `kubectl get pod -o json`.                                     |
| **Estado final**                       | `OOMKilled` → `CrashLoopBackOff` | Tras matar el contenedor, kubelet lo reinicia y entra en bucle.            |

---

### Lab 5 · Filtrado de eventos

```bash
# Solo eventos Warning del namespace dev
kubectl get events -n events --field-selector type=Warning

# Eventos de Deployments en todo el clúster
kubectl get events -A --field-selector involvedObject.kind=Deployment

# Reasons únicos
kubectl get events -A -o json | jq -r '.items[].reason' | sort -u
```

**¿Qué estamos aprendiendo?**  A usar *field selectors* para buscar eventos por severidad, recurso, nodo, etc., habilidad clave para el examen.

---



## Recursos y enlaces útiles

* 📖 [Kubernetes Events – Tasks](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_events/)
* 🔗 [KCNA Curriculum](https://training.linuxfoundation.org/certification/kubernetes-cloud-native-associate/)


---

¡Con estos ejemplos y explicaciones podrás identificar rápidamente cada tipo de fallo y **relacionar** los estados del Pod con los eventos generados, tal como se evalúa en el examen KCNA!
