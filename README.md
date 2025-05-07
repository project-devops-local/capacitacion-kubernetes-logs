# capacitacion kubernetes logs
capacitacion-kubernetes-logs


## Introducción

Esta capacitación virtual se mostrará de manera sencilla los conceptos básicos de observabilidad, monitoreo y seguridad en Kubernetes. Usaremos Prometheus para recopilar métricas y Grafana para visualizarlas, y repasaremos prácticas básicas de seguridad en tu clúster.

## Seccion 1

### Introducción breve a Kubernetes
¿Qué es Kubernetes? Kubernetes (k8s) es una plataforma open source para la orquestación de contenedores, que automatiza la implementación, gestión y escalado de aplicaciones en contenedores.


## Arquitectura Kubernetes. 

![Arquitectura Kubernetes](images/kubernetes-cluster-architecture.svg)


## 🧠 Componentes de Kubernetes

| Componente                  | Tipo            | Descripción breve                                                                 |
|----------------------------|-----------------|------------------------------------------------------------------------------------|
| **Control Plane**          | Cerebro         | Decide qué ejecutar, dónde y cómo reaccionar si algo falla.                       |
| kube-api-server            | Control Plane   | Puerta de entrada a Kubernetes; recibe peticiones y las comunica al sistema.      |
| etcd                       | Control Plane   | Base de datos distribuida que guarda el estado y configuración del clúster.       |
| kube-scheduler             | Control Plane   | Asigna Pods a los nodos según disponibilidad de recursos.                         |
| kube-controller-manager    | Control Plane   | Garantiza que el estado deseado se cumpla (ej. mantener 3 Pods activos).          |
| cloud-controller-manager   | Control Plane   | Integra Kubernetes con servicios del proveedor cloud (balanceadores, discos).     |
| **Worker Nodes**           | Ejecutores      | Máquinas donde realmente corren los contenedores (aplicaciones).                  |
| kubelet                   | Nodo de trabajo | Agente que ejecuta instrucciones del control plane en el nodo correspondiente.    |
| kube-proxy                | Nodo de trabajo | Maneja la red y permite la comunicación entre Pods y con el exterior.             |
| Pod                       | Nodo de trabajo | Unidad mínima de ejecución; puede contener uno o más contenedores.               |
| CRI (containerd/Docker)   | Nodo de trabajo | Ejecuta los contenedores; interfaz de ejecución compatible con Kubernetes.         |



## 🔍 Observabilidad en Sistemas Distribuidos

| Concepto        | Definición                                                                 |
|-----------------|----------------------------------------------------------------------------|
| **Observabilidad** | Capacidad de entender el estado interno de un sistema complejo a partir de su salida de datos. |
| **¿Por qué es importante?** | Permite diagnosticar fallas, detectar cuellos de botella y mejorar la confiabilidad. Fundamental en entornos dinámicos como Kubernetes y microservicios. |
| **Logs**         | Registros detallados de eventos que ocurren dentro del sistema. Útiles para auditoría y diagnóstico. |
| **Métricas**     | Valores numéricos que representan el estado del sistema en el tiempo (CPU, memoria, latencia, etc.). |
| **Trazas**       | Seguimiento de una petición a través de múltiples servicios. Permite ver tiempos y dependencias. |
| **Ventaja frente al monitoreo** | Mientras el monitoreo alerta sobre fallos, la observabilidad permite investigar la causa raíz con datos más ricos. |
