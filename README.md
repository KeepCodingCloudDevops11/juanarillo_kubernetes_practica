# PRÁCTICA CONTENEDORES: MÁS QUE VMS KUBERNETES - JUAN ARILLO

Práctica de Juan Arillo para el módulo de **Contenedores: Más que VMs Kubernetes** a la nube.

## TABLA DE CONTENIDOS

[Descripción](#descripción)  
[Requisitos](#requisitos)  
[Pasos previos](#pasos-previos)  
[Funcionamiento](#funcionamiento)  
[Ajustes](#ajustes)  
[Prueba del escalado](#prueba-del-escalado)  
[Finalización despliegue](#finalización-despliegue)

## DESCRIPCIÓN

Este proyecto despliega en Kubernetes una aplicación *Flask* que trabaja sobre una base de datos *Redis*.  

La aplicación *Flask* muestra un texto que proviene del fichero `values.yaml` de la configuración del chart de Helm, más un texto que muestra las veces que se ha cargado la página principal de la aplicación. La imagen de la aplicación *Flask* está en un repositorio personal de docker hub ([*Docker hub*](https://hub.docker.com/repository/docker/juanarillo/docker_practica/general)).

El servicio de la base de datos *Redis*, sirve como persistencia del número de veces que se visita la página principal
de la aplicación.

Su arquitectura sería de este tipo  

<p align="center">
    <img src="images/arquitectura.png" alt="architecture" width="500"/>
</p>

## REQUISITOS

Para poder desplegar este proyecto, serán necesarias las siguientes herramientas:

- *Minikube* -> Cluster local de Kubernetes para desarrollo y pruebas ([*Minikube*](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fmacos%2Farm64%2Fstable%2Fbinary+download)).
- *Kubectl* -> Interacción con los clústeres Kubernetes ([*Kubectl*](https://kubernetes.io/docs/tasks/tools/)).
- *Helm* -> Gestor de paquetes para Kubernetes ([*Helm*](https://helm.sh/docs/intro/install/)).

## PASOS PREVIOS

1- Descarga del proyecto desde el repositorio de *Github*.

```bash
git clone https://github.com/KeepCodingCloudDevops11/juanarillo_kubernetes_practica.git
```

2- Arranque de minikube

```bash
minikube start
```

3- Instalación del addon de ingress para minikube

```bash
minikube addons enable ingress
```

## DESPLIEGUE

- Nos situaremos dentro de la carpeta del proyecto.

```bash
cd juanarillo_kubernetes_practica
```

- Una vez en la carpeta, desplegaremos la aplicación usando *Helm*

```bash
helm install flask-app .

# Donde flask-app es el nombre que queramos darle a la release.
```

- Para poder ver la aplicación funcionando en el navegador, tendremos que hacer dos cosas:   

  - Al no tener dns, habrá que reflejar el host establecido en el fichero `values.yaml` para ingress > host, en el fichero hosts del ordenador local:

    ```bash
    sudo nano /etc/hosts
    ```

    ```txt
    # Añadimos en el fichero hosts el nombre que hayamos establecido en
    # el fichero values.yaml, en la sección ingress > host
    127.0.0.1    flask-app.local
    ```

  - Arrancamos el tunel de minikube para poder usar el ingress.  

    ```bash
    minikube tunnel
    ```

## FUNCIONAMIENTO

Una vez desplegado el cluster, y configurado el ordenador local, se puede abrir al navegador y acceder usando la url establecido en el host (http://flask-app.local si no se modifica el `values.yaml`).  

La aplicación debe mostrar un texto variable al principio (establecido en el fichero `values.yaml` en flask > message), y un texto con el numero de veces que se ha recargado la página. Este contador aumentará cada vez que se recargue la misma.

<p align="center">
    <img src="images/app.png" alt="architecture" width="500"/>
</p>

[Ver el video](/images/demo.mov)

<p align="center">
    <video width="600" controls align="center">
    <source src="images/demo.mov" type="video/quicktime">
    Your browser does not support the video tag.
    </video>
</p>

## AJUSTES

El fichero `values.yaml` presenta diferentes variables que nos permite personalizar nuestro despliegue:

**Sección flask:**

Esta parte define la configuración principal de la aplicación Flask.

- image: Especifica la imagen de Docker que se utilizará para desplegar la aplicación Flask (juanarillo/kubernetes_practica:v5).
- replicas: Define el número de réplicas del despliegue (en este caso, 2).
- message: Un mensaje personalizado que puede ser utilizado por la aplicación Flask (por ejemplo, en el contenido mostrado).
- cpuRequest: Cantidad mínima de CPU asignado a cada réplica (250 millicores, o 0.25 CPU).
- memoryRequest: Mínimo de memoria RAM asignado a cada réplica (256 MB).
- cpuLimit: El límite máximo de CPU asignado a cada réplica (500 millicores, o 0.5 CPU).
- memoryLimit: El límite máximo de memoria RAM asignado a cada réplica (512 MB).

**Sección service:**

Define la configuración del servicio de Kubernetes que expone la aplicación.

- type: El tipo de servicio, aquí configurado como NodePort, que permite acceder a la aplicación desde fuera del clúster.
- port: El puerto en el que la aplicación Flask escucha internamente (5000).

**Sección redis:**

Configura la instancia de Redis utilizada por la aplicación.

- image.repository: Especifica el repositorio de la imagen de Redis (en este caso, redis).
- image.tag: Define la versión de Redis (7.0).
- port: El puerto interno donde se ejecuta Redis (6379).
- password: La contraseña de acceso a la instancia de Redis (mipassword).
- persistentVolume.enabled: Indica si se habilita el almacenamiento persistente (valor true).
- persistentVolume.size: Define el tamaño del volumen persistente (1 GiB).
- persistentVolume.storageClass: La clase de almacenamiento a utilizar (standard).

**Sección hpa (Horizontal Pod Autoscaler):**

Configura el escalado horizontal automático de las réplicas.

- enabled: Si está habilitado o no (true).
- minReplicas: Número mínimo de réplicas cuando el HPA está habilitado (2).
- maxReplicas: Número máximo de réplicas permitidas (5).
- cpuThreshold: El umbral de utilización de CPU en porcentaje (70%) a partir del cual el HPA ajustará las réplicas.

**Sección ingress:**

Configura el acceso externo a la aplicación a través de un Ingress Controller.

- enabled: Indica si el recurso Ingress está habilitado (true).
- host: El nombre de host asignado para acceder a la aplicación (flask-app.local).

## PRUEBA DEL ESCALADO

- Activar el addon de Metrics para minikube. El addon tardará un rato en arrancar el pod.

```bash
minikube addons enable metrics-server

kubectl get pods -n kube-system # para ver cuando está arrancado el metrics-server
```

- Una vez arrancado, lanzamos el pod de tests que existe en este repositorio, y le ampliamos el número de réplicas para que aumente el porcentaje de cpu.

```bash
kubectl apply -f test-pod.yaml

kubectl scale deployment test-pod --replicas 5
```

- Observamos el porcentaje de uso del hpa. Se irá actualizando cada poco tiempo.

```bash
kubectl get hpa -w
```

- Cuando se alcanza el límite alcanzado empezarán a crearse nuevos pod. Podemos observarlo mediante el siguiente comando.

```bash
kubectl get pods -w
```

## FINALIZACIÓN DESPLIEGUE

Una vez se desea finalizar el despliegue se puede usar el comando *uninstall* de *Helm*.

```bash
helm uninstall flask-app

# Siendo flask-app el nombre que le hayamos dado a la release.
```

También se puede borrar.

```bash
helm delete flask-app
```

Posteriormente podremos apagar minikube o borrarlo todo

```bash
minikube delete
```
