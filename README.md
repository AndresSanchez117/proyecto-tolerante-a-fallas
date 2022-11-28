# Proyecto final de Computación Tolerante a Fallas

Para el despliegue de este proyecto es necesario tener acceso a una distribución de Kubernetes.
Aquí se utilizará como ejemplo Docker Desktop, que es capaz de habilitar un clúster de Kubernetes local.

![Docker Desktop](/images/docker_desktop.png)

> NOTA: es importante tener asignado por lo menos 4 CPU y 4 GB de RAM en Docker Desktop.

Después de esto podremos ver todos los contenedores asociados con Kubernetes desde Docker Desktop.

![Containers](/images/containers.png)

Se habilitará también en nuestro sistema el comando kubectl. Por ejemplo, para ver los nodos de nuestro clúster de Kubernetes podemos ejecutar.

`kubectl get nodes`

A continuación instalaremos Istio en nuestra distribución de Kubernetes. Primero descargamos istio con el siguiente comando. (Aquí se utiliza la versión 1.15.3).

`curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.15.3 sh -`

Tendremos una carpeta de Istio en la ruta en la que ejecutamos el comando anterior.

`cd istio-1.15.3`

En este punto podemos ejecutar istioctl para asegurarnos de que todo funciona correctamente.

`./bin/istioctl version`

Debemos de ver la versión de Istio desplegada en el terminal. Podemos agregar el binario de istioctl al path de nuestro sistema, o continuar ejecutándolo como se demustra en el comando anterior.

Para verificar que nuestra instalación de Istio sea apta para nuestro clúster de Kubernetes podemos ejecutar.

`istioctl x precheck`

Ahora instalamos istio con el perfil de demostración ejecutando:

`istioctl install --set profile=demo -y`

Verifica la instalación con:

`istioctl verify-install`

Finalmente necesitamos instalar los componentes adicionales de soporte. Desde la raíz de nuestra descarga de Istio ejecutamos:

`kubectl apply -f ./samples/addons`

Podemos ver con el siguiente comando todos los pods del plano de control de Istio, que incluye componentes tales como Kiali, Grafana, o Jaeger.

`kubectl get pod -n istio-system`

![Control plane pods](/images/pods.png)

Examinemos estos componentes con la siguiente figura:

![Istio system](/images/istio_system.png)

Ahora construiremos las imágenes de Docker necesarias para desplegar la aplicación. Dentro del directorio del repositorio nos dirigimos a services/catalog, ahí encontraremos el código de Node.js para el servicio, y un Dockerfile para construir la imágen correspondiente. Ejecutamos aquí el siguiente comando:

`docker build -t andresss117/catalog:latest .`

> NOTA: Reemplaza "andresss117" por tu propio namespace de Docker Hub.


## Referencias:
