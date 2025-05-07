# capacitacion kubernetes logs
capacitacion-kubernetes-logs


## Introducción

Esta capacitación virtual se mostrará de manera sencilla los conceptos básicos de observabilidad, monitoreo y seguridad en Kubernetes. Usaremos Prometheus para recopilar métricas y Grafana para visualizarlas, y repasaremos prácticas básicas de seguridad en tu clúster.

## Seccion 1

### Introducción breve a Kubernetes
¿Qué es Kubernetes? Kubernetes (k8s) es una plataforma open source para la orquestación de contenedores, que automatiza la implementación, gestión y escalado de aplicaciones en contenedores.

### Componentes

**control plane:** Es el cerebro de Kubernetes. Aquí se toman todas las decisiones: qué ejecutar, dónde y cómo reaccionar si algo falla.

#### Componentes dentro del Control Plane:
**API Server (kube-api-server)**

* Es la puerta de entrada a Kubernetes. Todo pasa por aquí, como los pedidos que haces en un restaurante.

* Si tú o alguna herramienta quiere crear una app, escalarla o ver su estado, lo hace a través del API Server.

**etcd**

* Es como una libreta de apuntes, donde Kubernetes guarda todo: configuraciones, estado del sistema, qué apps están corriendo, etc.

* Es una base de datos muy rápida y confiable.

**Scheduler (kube-scheduler)**

* Decide en qué servidor (nodo) se va a ejecutar una aplicación.

* Por ejemplo: "Tengo que lanzar esta app, ¿en qué máquina hay más espacio?"


**Controller Manager (kube-controller-manager)**

* Se asegura de que lo que pediste realmente esté pasando.

* Si pediste 3 copias de una app (3 pods) y solo hay 2, este componente creará la tercera.

**Cloud Controller Manager**

* Se conecta con proveedores de nube (como AWS, GCP, Azure).

* Si estás en la nube, este se encarga de integrar con servicios como balanceadores de carga o almacenamiento externo.



### Nodos de Trabajo (Worker Nodes):

**kubelet**

* Es un pequeño agente que recibe órdenes del Control Plane y se asegura de que los contenedores estén funcionando correctamente.

* Si el Control Plane le dice: "Corre esta app", el kubelet lo hace.

**kube-proxy**

* Es el que maneja la red.

* Se asegura de que las aplicaciones puedan comunicarse entre sí y con el mundo exterior.

**Pods**

* Es la unidad más pequeña de ejecución en Kubernetes.

* Un Pod puede contener uno o varios contenedores (aplicaciones).

* CRI (Container Runtime Interface): No se ve claramente explicado, pero es lo que ejecuta los contenedores, como Docker o containerd.

## Arquitectura Kubernetes. 

![Arquitectura Kubernetes](images/kubernetes-cluster-architecture.svg)