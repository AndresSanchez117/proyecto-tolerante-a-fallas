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

Vamos ahora a services/webapp para construir la imagen del siguiente servicio:

`docker build -t andresss117/webapp:latest .`

Para empezar a desplegar la aplicación en Kubernetes el primer paso es crear un namespace en el que desplegaremos nuestros servicios:

`kubectl create namespace istioinaction`

`kubectl config set-context $(kubectl config current-context) \
--namespace=istioinaction`

Una vez estamos en en namespace de istioinaction, encontraremos el archivo de recursos de Kubernetes para el servicio del catálogo dentro del repositorio en services/catalog/kubernetes/catalog.yaml.

Antes de desplegar esto vamos a tomar ventaja de una caracterísitca de Istio que nos permite inyectar automáticamente el sidecar proxy a los contenedores desplegados.

Para habilitar esta característica, etiquetamos el namespace de istioinaction de la siguiente forma:

`kubectl label namespace istioinaction istio-injection=enabled`

Ahora vamos a desplegar el servicio de catálogo.

`kubectl apply -f services/catalog/kubernetes/catalog.yaml`

A continuación desplegamos el servicio de webapp de la siguiente forma:

`kubectl apply -f services/webapp/kubernetes/webapp.yaml`

Con Istio podemos utilizar el Istio ingress gateway para permitir que tráfico acceda al clúster. Ejecutamos el siguiente comando para exponer nuestro servicio de webapp:

`kubectl apply -f ingress-gateway.yaml`

Otra forma de acceder a la aplicación web es realizando una redirección de puertos con kubectl (en este caso se utilizará el puerto 8080):

`kubectl port-forward deploy/webapp 8080:8080`

Veamos si podemos alcanzar nuestro servicio y el servicio de webapp puede ver al de catalog. En Docker Desktop el endpoint se coloca por defecto como http://localhost:80.

![Webapp](/images/webapp.png)

Istio puede recolectar gran cantidad de telemetría sobre que está ocurriendo en nuestras aplicaciones. Para obtener métricas podemos utilizar Prometheus y Grafana.

Vamos a utilizar istioctl para poder visualizar el dashboard de Grafana de forma local:

`istioctl dashboard grafana`

Esto debería abrir nuestro navegador automáticamente en http://localhost:3000 donde veremos la página principal del dashboard.

![Grafana home](/images/grafana_home.png)

Da click en el botón de la parte izquierda superior para acceder a una lista de dashboards a las que podemos cambiar. Accede a la Istio Service Dashboard y nos aseguramos de que la parte de servicio muestre "webapp.istioinaction.svc.cluser.local".

Inicialmente los valores estarán en cero o mostrando N/A. Para generar algo de tráfico artificial podemos ejecutar el siguiente comando:

`while true; do curl http://localhost/api/catalog; sleep .5; done`

Si ahora vemos el dashboard de grafana deberiamos de ver algo de tráfico. Notamos que por ahora tenemos una proporción de éxito de peticiones del 100%.

![Grafana 100](/images/grafana_100.png)

Podemos utilizar Jaeger con Istio para obtener un dashboard de seguimiento de peticiones.

`istioctl dashboard jaeger`

Naveguemos a http://localhost:16686, que nos llevará a la consola web de Jaeger. Aquí podemos empezar a filtrar peticiones. El servicio en el campo superior izquierdo debe ser istio-ingressgateway.istio-system, después da click en el botón de búsqueda de la parte inferior izquierda, deberiamos ver entonces la lista de peticiones más recientes.

![Jaeger traces](/images/jaeger_traces.png)

Al dar click en alguna de estas peticiones podremos ver más de sus detalles.

![Jaeger trace](/images/jaeger_trace.png)

Ahora veremos los aspectos de resiliencia que nos provee Istio.
Uno de estos aspectos es el de reintentar peticiones entre errores intermitentes de la red. Por ejemplo, si la red experimenta fallos, nuestras aplicaciones podrian ver estos errores, y continuar reintentando la petición.

Veremos primero que con el siguiente comando todas las peticiones al servicio de catálogo regresan exitósamente:

`while true; do curl http://localhost/api/catalog; sleep .5; done`

Para hacer que las peticiones fallen podemos utilizar un script que inyecta un mal comportamiento en la aplicación. Ejecutar el siguiente comando desde la raíz del repositorio causará que las peticiones fallen con un error 500 el %50 de las veces.

`./bin/chaos.sh 500 50`

Ahora podemos probar de nuevo realizar peticiones al servicio:

`while true; do curl http://localhost/api/catalog; sleep .5; done`

La salida de este comando deberían ser errores intermintentes del servicio.
Podemos ver ahora datos sobre estos errores en la consola de Grafana.

![Grafana 50](/images/grafana_50.png)

Así como también visualizar la diferencia entre cada una de las peticiones exitósas y las fallidas en Jaeger.

![Jaeger 50](/images/jaeger_50.png)

Veremos entonces como Istio puede hacer la red más tolerante a fallos entre los servicios de webapp y catálogo.

Utilizando un VirtualService de Istio, podemos especificar reglas sobre la interacción entre servicios en la malla. El siguiente es un ejemplo de tal definición de un VirtualService.

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog
spec:
  hosts:
  - catalog
  http:
  - route:
    - destination:
        host: catalog
    retries:
      attempts: 3
      retryOn: 5xx
      perTryTimeout: 2s
```

Con esta definición se especifica que las peticiones al servicio de catálogo son elegibles para reintentarse hasta tres veces, con cada una teniendo un tiempo límite de dos segundos. Si se aplica esta regla podemos utilizar Istio para automáticamente reintentar la petición cuando se experimentan fallos.

`kubectl apply -f catalog-virtualservice.yaml`

Ahora prueba realizar peticiones de nuevo con:

`while true; do curl http://localhost/api/catalog; sleep .5; done`

Deberiamos ver ahora menos excepciones. Esto demuestra como podemos utilizar Istio para añadir características de resiliencia sobre la red sin tener que tocar el código de nuestra aplicación.

Para detener los fallos en el servicio puedes ejecutar:

`./bin/chaos.sh 500 delete`

## Referencias:

- Posta, C. E., & Maloku, R. (2022). Istio in action. Manning Publications Co.