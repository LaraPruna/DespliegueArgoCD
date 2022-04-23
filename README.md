# Despliegue continuo en Kubernetes con GitOps y ArgoCD

## Tabla de contenidos

1. [Introducción](#introduccion)
2. [Objetivos del proyecto](#objetivos-del-proyecto)
3. [Fundamentos teóricos y conceptos previos](#fundamentos-teoricos-y-conceptos-previos)
	1. [GitOps](#gitops)
		1. [¿Qué es GitOps?](#¿que-es-gitops?)
		2. [Principios de GitOps](#principio-de-gitops)
		3. [Ventajas e inconvenientes de GitOps](#ventajas-e-inconvenientes-de-gitops)
	2. [ArgoCD](#argocd)
	3. [Ngrok](#ngrok)
4. [Escenario necesario para la realización del proyecto](#escenario-necesario-para-la-realización-del-proyecto)
	1. [Instalación de ArgoCD con manifiestos](#instalación-de-argocd-con-manifiestos)
	2. [Instalación de ArgoCD con Autopilot](#instalación-de-argocd-con-autopilot)
	3. [Instalación de ArgoCD con Helm](#instalación-de-argocd-con-helm)
	4. [Configuración de ArgoCD](#configuración-de-argocd)
5. [Desarrollo del proyecto](#desarrollo-del-proyecto)
	1. [Crear una aplicación en ArgoCD](#crear-una-aplicación-en-argocd)
		1. [Por interfaz gráfica](#por-interfaz-gráfica)
		2. [Por línea de comandos](#por-línea-de-comandos)
		3. [Por manifiesto de Kubernetes](#por-manifiesto-de-kubernetes)
	2. [Sincronizar una aplicación manualmente](#sincronizar-una-aplicación-manualmente)
	3. [Sincronizar una aplicación automáticamente](#sincronizar-una-aplicación-automáticamente)
		1. [Crear un webhook en GitHub](#crear-un-webhook-en-github)
		2. [Configurar el secret del webhook en ArgoCD](#configurar-el-secret-del-webhook-en-argocd)
		3. [Instalación y uso de ngrok](#instalación-y-uso-de-ngrok)
	4. [La salud de la aplicación](#la-salud-de-la-aplicación)
	5. [Estrategias de sincronización](#estrategias-de-sincronización)
	6. [Gestión de secrets](#gestión-de-secrets)
		1. [Cómo funcionan los secrets en Kubernetes](#cómo-funcionan-los-secrets-en-kubernetes)
		2. [Encriptar los secrets](#encriptar-los-secrets)
6. [Conclusiones y propuestas adicionales para el proyecto](#conclusiones-y-propuestas-adicionales-para-el-proyecto)
7. [Bibliografía](#bibliografia)

-------------------------------------------------------------------------

## Introducción

<br>

## Objetivos del proyecto

<br>

## Fundamentos teóricos y conceptos previos

### GitOps

#### ¿Qué es GitOps?

Para explicar ArgoCD, antes debemos conocer el concepto de **GitOps**. El término deriva de la unión de las palabras Git (software de control de versiones) y Ops (operaciones), y surge en 2017, junto con la creciente popularidad de los contenedores y la computación en la nube, como un conjunto de buenas prácticas para la gestión de infraestructuras en Kubernetes y la entrega de aplicaciones. Se trata de una metodología en la que se emplea Git para gestionar todas las fases en el desarrollo de un proyecto.

<p align="center">
<img src="images/Short-GitOps-Timeline-Light.png" alt="Proceso de despliegue sin GitOps" width="750"/>
</p>

GitOps se considera una forma mejorada de infraestructura como código (**Infrastructure as Code, IaC**), que no es más que un mecanismo para crear y mantener infraestructuras usando código para definirlas y automatizar su configuración, en lugar de hacerlo todo manualmente. Normalmente, el código se encuentra en ficheros legibles por la máquina, y almacenados de manera que se pueda llevar a cabo un control de versiones.

Las IaC, como Ansible o Terraform, nos permiten ahorrar tiempo a la hora de desplegar nuestra infraestructura en un entorno y reproducirla en otro, pero con la llegada de las **plataformas en la nube** y, sobre todo, de los **contenedores**, la forma de aprovisionar dichas infraestructuras ha ido cambiando considerablemente y a un paso vertiginoso. Como consecuencia de esto, y siendo Git cada vez más el sistema de control de versiones predominante, GitOps ha ido ganando altura, pues va más allá del resto de IaC cuando se trata de definir todos los componentes del software (la infraestructura, la red, los datos y la aplicación) en forma de código (Everything as a Code, EaaC).

Con GitOps, podremos usar determinados agentes para que nos informen de cualquier **divergencia** que exista entre el código almacenado en Git y el que se está ejecutando en los clústeres. Si se encuentra alguna diferencia, los reconciliadores de Kubernetes se encargan de actualizar o hacer un *rollback* en el cluster.

<br>

#### Principios de GitOps

La metodología GitOps sigue estos cuatro principios:

1. **Todo el sistema se define usando un lenguaje declarativo**: esto significa que la configuración se realiza indicando lo que se debe hacer en lugar de cómo debería hacerse (hechos en lugar de instrucciones). De esta manera, se puede leer el código de la infraestructura desde un repositorio fácilmente.

2. **Git es la única fuente de verdad**: al almacenar la declaración del sistema en un sistema de control de versiones, podremos ir siguiendo la evolución de todo el código desde un único lugar. Además, esto facilita el hecho de volver a una versión anterior si algo sale mal. Git también nos permite hacer uso de claves SSH para firmar los cambios, lo que nos garantiza una mayor seguridad en la autoría y procedencia del código.

3. **Los cambios aprobados se aplican automáticamente al sistema**: una vez se ha declarado el código en Git, el siguiente paso es permitir que cualquier cambio que hagamos ahí se aplique de forma automática en el sistema. La clave de esto es que no necesitamos usar las credenciales del cluster para cambiar el sistema, bastará con realizar un "commit" o un "pull request".

4. **Uso de agentes para asegurar la corrección y las alertas de divergencia**: ya declarada la infraestructura y almacenada en un sistema de control de versiones, los agentes nos informarán de cualquier estado que se aleje de nuestras expectativas. El uso de agentes también nos asegura la autocorrección del sistema en el caso de que se produzca un error humano, en cuyo caso el agente tratará de aplicar sus propias soluciones (como volver a desplegar la aplicación) para recuperar el estado deseado.

<br>

#### Ventajas e inconvenientes de GitOps

Este es el proceso que se realiza en un despliegue tradicional **sin GitOps**:

1. El desarrollador guarda los cambios en el código fuente de la aplicación.
2. Un sistema de integración continua construye la aplicación, y puede llegar a realizar otras tareas, como pruebas unitarias, escaneos de seguridad, verificaciones de estado, etc.
3. La imagen del contenedor se almacena de un registro de contenedores.
4. La plataforma de IC (u otro sistema externo) con acceso directo al cluster de Kubernetes crea un despliegue usando una variación del comando "kubectl apply".
5. La aplicación se despliega en el cluster.

<p align="center">
<img src="images/ProcesoDespliegueSinGitOps.png" alt="Proceso de despliegue sin GitOps" width="750"/>
</p>

Aspectos a destacar de este proceso:

* El estado del cluster se decide de forma **manual** mediante comandos de kubectl o accesos API.
* La plataforma que despliega el cluster de Kubernetes tiene **total acceso** a este desde un punto externo.

GitOps nos permite modificar el proceso para que sea como sigue:

1. Los primeros pasos son los mismos: el desarrollador aplica los cambios del código fuente, y el sistema de IC crea un despliegue que se guarda en un registro.
2. Nadie tiene acceso directo al cluster de Kubernetes, y hay un segundo repositorio Git con todos los manifiestos que definen la aplicación.
3. Otra persona o un sistema automatizado cambia los manifiestos en el segundo repositorio Git.
4. Un controlador de GitOps que se ejecuta dentro del cluster monitoriza el repositorio Git y modifica el estado del cluster tan pronto se aplique un cambio, para hacer coincidir dicho estado con el que se describe en Git.

<p align="center">
<img src="images/ProcesoDespliegueConGitOps.png" alt="Proceso de despliegue con GitOps" width="750"/>
</p>

Aspectos a destacar de este proceso:

* El estado del cluster siempre se describe en **Git**, donde se almacena todo lo que forma parte de la aplicación.
* No hay ningún sistema de despliegue externo con total acceso al cluster. Es el **propio cluster** el que obtiene los cambios y la información del despliegue.
* El controlador de GitOps se ejecuta en un **bucle constante** que sincroniza el estado del repositorio Git con el del cluster.

De esto podemos deducir que GitOps ofrece las siguientes **ventajas**:

* Una mayor productividad, al reducir el trabajo humano.
* Una experiencia de desarrollo mejorada, al facilitarle las tareas al desarrollador.
* Una mayor estabilidad, ya que Git ofrece un registro más completo de todos los cambios realizados.
* Una mayor fiabilidad, puesto que siempre se puede dar marcha atras con un rollback en el caso de que algo salga mal.
* Consistencia y estandarización, porque GitOps ofrece un modelo para desarrollar infraestructuras, aplicaciones y cambios adicionales en Kubernetes de principio a fin.
* Una mayor garantía de seguridad, tanto en la integración de los datos como a la hora de probar la autoría y origen de los mismos.

Hay que mencionar que no todo es oro en GitOps: el almacenamiento de **objetos de tipo *Secret*** en Git supone un problema, puesto que este valor suele codificarse (que no encriptarse) en base64, lo cual resulta muy fácil de descodificar. No obstante, algunas empresas le han dado solución a este problema, como los *Sealed Secrets* de Bitnami, que encriptan los *Secrets* para que sea seguro almacenarlos en Git.

<br>

### ArgoCD

ArgoCD, perteneciente a la Cloud Native Computing Foundation (CNCF), es una de las herramientas más populares actualmente (y de las primeras que apareció en el mercado) para desplegar de forma continua aplicaciones en Kubernetes y con base de GitOps. No solo se la conoce por su excelente **administración y despliegue de aplicaciones de Kubernetes**, sino también por sus funciones de autocorrección de errores, gestión de acceso de usuarios, verificación de estado, etc. ArgoCD implementa todos los principios de GitOps que hemos descrito antes de la siguiente manera:

1. Instalamos Argo CD como un controlador en el cluster de Kubernetes. Normalmente, lo instalaremos en el mismo cluster que va a gestionar, aunque también puede administrar otros clústeres externos.
2. Almacenamos las definiciones en Git, sean del tipo que sean. ArgoCD es bastante abierto a este respecto, soporta tanto manifiestos simples de Kubernetes como *charts* de Helm, definiciones de Kustomize y otros sistemas de plantillas.
3. Creamos una aplicación en ArgoCD indicando qué repositorio Git va a monitorizar, y en qué cluster o espacio de nombre deberá instalarse dicha aplicación.
4. Desde este momento, ArgoCD monitorizará el repositorio de Git, y cuando haya algún cambio, modificará el cluster de forma automática para que tenga el mismo estado.
5. De manera opcional, ArgoCD desplegará aplicaciones en otros clústeres (no solo en el que se encuentra instalado).

<p align="center">
<img src="images/ProcesoArgoCD.png" alt="Proceso de ArgoCD" width="750"/>
</p>

Este es el proceso básico de despliegue de ArgoCD, pero también cuenta con otras características (que describiremos más adelante), como:

* Sincronización manual o automática
* Definición de oleadas y marcos de sincronización (*Sync Waves* y *Sync Windows*, respectivamente)
* Configuración declarativa de aplicaciones

Argo CD cuenta con una **interfaz gráfica integrada** clara y fácil de administrar, donde se muestra tanto la estructura de la aplicación como estado de la sincronización. Tras desplegarla, ArgoCD irá realizando *pulls* a los repositorios añadidos para comparar el estado de estos con el de los clústeres.

<p align="center">
<img src="images/InterfazGrafica.png" alt="Interfaz gráfica de ArgoCD" width="750"/>
</p>

Con ArgoCD no solo podremos desplegar aplicaciones usando **ficheros yaml y json**, sino también herramientas como **Helm, Kustomize, Ksonnet**, etc. Permite la **autenticación** mediante OICD, OAuth2, LDAP, SAML 2.0, Github, GitLab, Microsoft, etc., y emplea una política basada en roles. En cuanto a la gestión de ***Secrets***, ArgoCD es compatible con Bitnami Sealed Secrets, Godaddy K8S External Secrets, Vault, Banzai Cloud Bank Vaults, Helm Secrets, Kustomize Secrets y AWS Secret Operator.

<br>

### Ngrok

En este proyecto también aprenderemos a utilizar ngrok, un **proxy inverso** que nos ofrece una forma segura de acceder a nuestros servicios locales desde cualquier parte del mundo. En este caso, usaremos dicha herramienta para exponer a Internet las aplicaciones de ArgoCD alojadas en Kubernetes, para que los *webhooks* que creemos en nuestros repositorios Git puedan acceder a ellas y activar la sincronización con el cluster automáticamente.

Ngrok funciona de una manera muy rápida y sencilla: una vez instalada la herramienta, basta con ejecutar un único comando en nuestra terminal indicando la URL/IP y el puerto donde tengamos la aplicación, con lo que obtendremos una URL generada dinámicamente y accesible desde Internet. Esta es la que utilizaremos a la hora de generar los webhooks.

<br>

## Escenario necesario para la realización del proyecto

En este apartado describiremos el proceso de instalación de ArgoCD para poder desplegar posteriormente nuestra aplicación. Existen varias formas de instalarlo en un cluster de Kubernetes:

### Instalación de ArgoCD con manifiestos

Si solo queremos ArgoCD para hacer pruebas y experimentar con la herramienta, podemos desplegar ArgoCD directamente con el fichero install.yaml que nos ofrece ArgoCD en su [repositorio de GitHub](https://github.com/argoproj/argo-cd/tree/master/manifests). Creamos un espacio de nombre y aplicamos dicho archivo en el mismo.
```
lpruna@debian:~$ kubectl create namespace argocd
lpruna@debian:~$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

lpruna@debian:~$ kubectl get all -n argocd
NAME                                                    READY   STATUS    RESTARTS   AGE
pod/argocd-application-controller-0                     1/1     Running   0          101s
pod/argocd-applicationset-controller-79f97597cb-p2g4l   1/1     Running   0          102s
pod/argocd-dex-server-6fd8b59f5b-4p2jf                  1/1     Running   0          102s
pod/argocd-notifications-controller-5549f47758-xsm4m    1/1     Running   0          102s
pod/argocd-redis-79bdbdf78f-x4868                       1/1     Running   0          102s
pod/argocd-repo-server-5569c7b657-zn9ql                 1/1     Running   0          102s
pod/argocd-server-664b7c6878-l7vlm                      1/1     Running   0          101s

NAME                                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/argocd-applicationset-controller          ClusterIP   10.103.132.210   <none>        7000/TCP                     102s
service/argocd-dex-server                         ClusterIP   10.98.252.112    <none>        5556/TCP,5557/TCP,5558/TCP   102s
service/argocd-metrics                            ClusterIP   10.104.107.166   <none>        8082/TCP                     102s
service/argocd-notifications-controller-metrics   ClusterIP   10.103.223.62    <none>        9001/TCP                     102s
service/argocd-redis                              ClusterIP   10.110.177.243   <none>        6379/TCP                     102s
service/argocd-repo-server                        ClusterIP   10.107.77.98     <none>        8081/TCP,8084/TCP            102s
service/argocd-server                             ClusterIP   10.110.54.94     <none>        80/TCP,443/TCP               102s
service/argocd-server-metrics                     ClusterIP   10.103.70.41     <none>        8083/TCP                     102s

NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/argocd-applicationset-controller   1/1     1            1           102s
deployment.apps/argocd-dex-server                  1/1     1            1           102s
deployment.apps/argocd-notifications-controller    1/1     1            1           102s
deployment.apps/argocd-redis                       1/1     1            1           102s
deployment.apps/argocd-repo-server                 1/1     1            1           102s
deployment.apps/argocd-server                      1/1     1            1           102s

NAME                                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/argocd-applicationset-controller-79f97597cb   1         1         1       102s
replicaset.apps/argocd-dex-server-6fd8b59f5b                  1         1         1       102s
replicaset.apps/argocd-notifications-controller-5549f47758    1         1         1       102s
replicaset.apps/argocd-redis-79bdbdf78f                       1         1         1       102s
replicaset.apps/argocd-repo-server-5569c7b657                 1         1         1       102s
replicaset.apps/argocd-server-664b7c6878                      1         1         1       101s

NAME                                             READY   AGE
statefulset.apps/argocd-application-controller   1/1     101s
```

En el mismo repositorio tenemos disponible otro archivo yaml llamado "namespace-install.yaml", con el que instalaremos ArgoCD como en el caso anterior. La única diferencia es que este está pensado para entornos con muchos usuarios repartidos entre diferentes equipos y proyectos, entre los cuales se han dividido los recursos del cluster mediante el uso de **espacios de nombre**. En este sentido, namespace-install.yaml permite instalar ArgoCD en un espacio de nombre concreto, y no requiere que tengamos privilegios a nivel de cluster. Esta es la mejor opción si queremos instalar ArgoCD en un solo cluster pero que también pueda gestionar otros externos.

Por último, también tenemos la opción de instalar ArgoCD en **alta disponibilidad**. El proceso es similar a los casos anteriores, con la única diferencia de que aquí se crearán múltiples réplicas para los distintos componentes.
```
lpruna@debian:~$ kubectl create namespace argocd
lpruna@debian:~$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/ha/install.yaml

lpruna@debian:~$ kubectl get all -n argocd
NAME                                                    READY   STATUS              RESTARTS   AGE
pod/argocd-application-controller-0                     0/1     ContainerCreating   0          5s
pod/argocd-applicationset-controller-79f97597cb-bx2hk   1/1     Running             0          6s
pod/argocd-dex-server-6fd8b59f5b-m7xvs                  0/1     Init:0/1            0          6s
pod/argocd-notifications-controller-5549f47758-b69bl    0/1     ContainerCreating   0          5s
pod/argocd-redis-ha-haproxy-5b75bb98dc-jtk8x            0/1     Pending             0          5s
pod/argocd-redis-ha-haproxy-5b75bb98dc-x6lk9            0/1     Pending             0          5s
pod/argocd-redis-ha-haproxy-5b75bb98dc-xw7lw            0/1     Init:0/1            0          5s
pod/argocd-redis-ha-server-0                            0/2     Init:0/1            0          5s
pod/argocd-repo-server-67c5d5bf75-5cwjl                 0/1     PodInitializing     0          5s
pod/argocd-repo-server-67c5d5bf75-njdcc                 0/1     Pending             0          5s
pod/argocd-server-757f6d6795-gw4bs                      0/1     ContainerCreating   0          5s
pod/argocd-server-757f6d6795-pfs5z                      0/1     Pending             0          5s

NAME                                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/argocd-applicationset-controller          ClusterIP   10.102.66.212    <none>        7000/TCP                     6s
service/argocd-dex-server                         ClusterIP   10.97.97.253     <none>        5556/TCP,5557/TCP,5558/TCP   6s
service/argocd-metrics                            ClusterIP   10.107.125.159   <none>        8082/TCP                     6s
service/argocd-notifications-controller-metrics   ClusterIP   10.104.154.138   <none>        9001/TCP                     6s
service/argocd-redis-ha                           ClusterIP   None             <none>        6379/TCP,26379/TCP           6s
service/argocd-redis-ha-announce-0                ClusterIP   10.100.78.54     <none>        6379/TCP,26379/TCP           6s
service/argocd-redis-ha-announce-1                ClusterIP   10.104.171.78    <none>        6379/TCP,26379/TCP           6s
service/argocd-redis-ha-announce-2                ClusterIP   10.97.51.120     <none>        6379/TCP,26379/TCP           6s
service/argocd-redis-ha-haproxy                   ClusterIP   10.103.93.213    <none>        6379/TCP                     6s
service/argocd-repo-server                        ClusterIP   10.102.153.79    <none>        8081/TCP,8084/TCP            6s
service/argocd-server                             ClusterIP   10.105.140.145   <none>        80/TCP,443/TCP               6s
service/argocd-server-metrics                     ClusterIP   10.105.227.247   <none>        8083/TCP                     6s

NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/argocd-applicationset-controller   1/1     1            1           6s
deployment.apps/argocd-dex-server                  0/1     1            0           6s
deployment.apps/argocd-notifications-controller    0/1     1            0           6s
deployment.apps/argocd-redis-ha-haproxy            0/3     3            0           6s
deployment.apps/argocd-repo-server                 0/2     2            0           5s
deployment.apps/argocd-server                      0/2     2            0           5s

NAME                                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/argocd-applicationset-controller-79f97597cb   1         1         1       6s
replicaset.apps/argocd-dex-server-6fd8b59f5b                  1         1         0       6s
replicaset.apps/argocd-notifications-controller-5549f47758    1         1         0       6s
replicaset.apps/argocd-redis-ha-haproxy-5b75bb98dc            3         3         0       5s
replicaset.apps/argocd-repo-server-67c5d5bf75                 2         2         0       5s
replicaset.apps/argocd-server-757f6d6795                      2         2         0       5s

NAME                                             READY   AGE
statefulset.apps/argocd-application-controller   0/1     5s
statefulset.apps/argocd-redis-ha-server          0/3     5s
```

<br>

### Instalación de ArgoCD con Autopilot

<p align="center">
<img src="images/argo_autopilot.png" alt="ArgoCD en piloto automático" width="450"/>
</p>

Si queremos trabajar directamente en un entorno de producción, tenemos a nuestra disposición la herramienta [Autopilot](https://github.com/argoproj-labs/argocd-autopilot), que además de instalar la propia aplicación de ArgoCD y desplegarla en un cluster de Kubernetes, guarda toda la configuración en un repositorio de GitOps para que se gestione a sí misma. Una vez instalado ArgoCD, ya podremos crear nuestros proyectos y aplicaciones con esta misma herramienta, que guardará todas las definiciones en el repositorio creado. Por su parte, ArgoCD notará los cambios en dicho repositorio y los aplicará al cluster, dando lugar a un proceso de despliegue continuo.

Para instalar Autopilot en Linux, podemos usar varias herramientas:

* **brew**, con el que ejecutaremos un único comando:

```
brew install argocd-autopilot
```

* **curl**, con el que podemos bajarnos la última versión de Autopilot directamente desde el repositorio oficial:

```
# Guardamos en una variable la última versión de Autopilot
VERSION=$(curl --silent "https://api.github.com/repos/argoproj-labs/argocd-autopilot/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')

# Nos descargamos el paquete .tar.gz de dicha versión y extraemos el binario de instalación
curl -L --output - https://github.com/argoproj-labs/argocd-autopilot/releases/download/$VERSION/argocd-autopilot-linux-amd64.tar.gz | tar zx

# Movemos el comando a nuestro directorio de binarios
mv ./argocd-autopilot-* /usr/local/bin/argocd-autopilot

# Ya tendremos instalado Autopilot, con el siguiente comando podremos ver la ayuda para empezar a usarlo
argocd-autopilot help
```

* También podemos utilizar una **imagen de Docker**, especificando las rutas en las que vamos a montar los directorios .kube y .gitconfig:

```
docker run \
  -v ~/.kube:/home/autopilot/.kube \
  -v ~/.gitconfig:/home/autopilot/.gitconfig \
  -it quay.io/argoprojlabs/argocd-autopilot <cmd> <flags>
```

En las dos últimas etiquetas, indicaremos los comandos que queremos ejecutar con la herramienta Autopilot, que en principio serán los necesarios para instalar ArgoCD y el controlador de aplicaciones, así como para crear nuestro primer proyecto.

Una vez tengamos instalado Autopilot, el siguiente paso será crear un **token de Git** para Autopilot. Para ello, nos vamos a los ajustes de nuestra cuenta de GitHub, y entramos en la pestaña "Ajustes de desarrollador" > "Tokens de acceso personal", y generamos un nuevo token. Le damos un nombre e indicamos el periodo de validez y los permisos que queremos darle (en este caso, bastaría con marcar la casilla "repo", puesto que lo que va a hacer Autopilot principalmente es crear y modificar repositorios). 

<p align="center">
<img src="images/NewPersonalAccessToken.png" alt="ArgoCD en piloto automático" width="750"/>
</p>

Nota: es importante que copiemos el token antes de salir de la página, ya que no podremos volver a verlo.

Hecho esto, crearemos un repositorio en GitHub para Autopilot (también podemos dejar que sea Autopilot quien lo cree), y generaremos dos variables de entorno en la máquina donde lo tengamos instalado:
```
# Nuestro token de Git
export GIT_TOKEN=ghp_d3nrpZHPTCGAOYBZop1VDCBDHQVgTj0OYxPu

# La URL de nuestro repositorio
export GIT_REPO=https://github.com/LaraPruna/autopilot.git
```

A continuación, ejecutamos el arranque de instalación en nuestro entorno de Kubernetes, con lo que se instalará ArgoCD y el controlador de aplicaciones.
```
argocd-autopilot repo bootstrap
```

Con esto, se nos habrá creado el repositorio (si no existía ya) con tres directorios: apps, bootstrap y projects. También se nos indicará al final de la instalación el usuario y la contraseña para acceder a ArgoCD y el comando que necesitamos para realizar un *port-forward*:
```
INFO pushing bootstrap manifests to repo          
Resolving deltas: 100% (1/1), done.
INFO applying argo-cd bootstrap application       
application.argoproj.io/autopilot-bootstrap created
INFO running argocd login to initialize argocd config 
'admin:login' logged in successfully
Context 'autopilot' updated

INFO argocd initialized. password: **************** 
INFO run:

    kubectl port-forward -n argocd svc/argocd-server 8080:80
```

<p align="center">
<img src="images/PortalArgoCD.png" alt="Portal de ArgoCD" width="750"/>
<img src="images/PortalArgoCD2.png" alt="Proyectos de Autopilot en ArgoCD" width="750"/>
</p>

Si olvidamos la contraseña del administrador de ArgoCD, siempre podemos recuperarla con el siguiente comando:
```
kubectl get secret argocd-initial-admin-secret -n argocd -ogo-template='{{printf "%s\n" (index (index . "data") "password" | base64decode)}}'
```

Lo siguiente sería crear nuestro primer proyecto. Para ello, ejecutamos el siguiente comando:
```
argocd-autopilot project create mi-primer-proyecto
```

Por último, instalamos nuestra primera aplicación en el proyecto, habiendo creado previamente un repositorio para la misma. En la documentación de Autopilot nos ofrecen una aplicación de prueba:
```
argocd-autopilot app create demoapp --app github.com/argoproj-labs/argocd-autopilot/examples/demo-app/ -p mi-primer-proyecto
```

Si nos vamos a la interfaz gráfica de ArgoCD, veremos que nos aparece la aplicación instalada:

<p align="center">
<img src="images/demoapp-argocd.png" alt="Nueva app en ArgoCD" width="750"/>
</p>

En el repositorio de Autopilot también se nos habrá añadido un nuevo directorio para la app:

<p align="center">
<img src="images/demoapp-git.png" alt="Nueva app en GitOps" width="750"/>
</p>

Para desinstalar Autopilot, usamos el argumento `uninstall`, con el que se borrarán el repositorio donde hemos guardado la herramienta y los recursos del mismo en el cluster de Kubernetes.
```
argocd-autopilot repo uninstall
```

<br>

### Instalación de ArgoCD con Helm

Si tenemos Helm instalado en nuestra máquina, podemos emplear un [chart de ArgoCD](https://github.com/argoproj/argo-helm/tree/master/charts/argo-cd) para instalarlo. Para ello, primero añadimos el repositorio de argo:
```
helm repo add argo https://argoproj.github.io/argo-helm
```

Después, instalamos el chart desde el mismo:
```
helm install my-release argo/argo-cd
```

Si queremos tener el chart de ArgoCD en alta disponibilidad, tendríamos que editar el fichero `values.yaml` y modificar los valores que se muestran a continuación, dependiendo de si queremos habilitar o no el autoescalado:

* Alta disponibilidad con autoescalado:
```
redis-ha:
  enabled: true

controller:
  enableStatefulSet: true

server:
  autoscaling:
    enabled: true
    minReplicas: 2

repoServer:
  autoscaling:
    enabled: true
    minReplicas: 2
```

* Alta disponibilidad sin autoescalado:
```
redis-ha:
  enabled: true

controller:
  enableStatefulSet: true

server:
  replicas: 2
  env:
    - name: ARGOCD_API_SERVER_REPLICAS
      value: '2'

repoServer:
  replicas: 2
```

<br>

### Configuración de ArgoCD

Una vez tengamos instalado ArgoCD mediante cualquiera de las opciones descritas antes, podemos personalizarlo según nuestras necesidades. Hay que tener en cuenta que, si lo hemos **instalado con Autopilot**, no podremos realizar ningún tipo de modificación en los ficheros de configuración, puesto que dicha aplicación se encargará de restaurarlos al menor cambio. Esto se debe a que, como he mencionado anteriormente, Autopilot está pensado para que la aplicación de ArgoCD se gestione por sí sola. Por lo tanto, si ese es el caso, podéis ignorar este apartado y continuar en el siguiente.

Dicho esto, para empezar a configurar ArgoCD, podemos generar un recurso ***Ingress*** para poder acceder a la interfaz gráfica a través de un nombre de dominio en lugar de una dirección IP. Para ahorrarnos tener que generar certificados SSL, editamos el recurso `deployments.apps/argocd-server`:
```
kubectl edit -n argocd deployments.apps argocd-server
```

Nos dirigimos a la línea donde se define el comando `argocd-server` y añadimos debajo la opción `--insecure`:
```
containers:
- command:
  - argocd-server
  - --insecure
```

Ya podríamos crear el *Ingress* de la manera tradicional:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
spec:
  rules:
  - host: www.argocd.org
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 80
```

Finalmente, aplicamos el fichero yaml:
```
kubectl apply -n argocd -f ingress.yaml
```

Hecho esto, solo quedaría añadir el dominio a nuestro servidor DNS o realizar una resolución estática en el fichero `/etc/hosts` de nuestra máquina.

<p align="center">
<img src="images/PortalArgoCD_dominio.png" alt="Accediendo al portal de ArgoCD con un dominio" width="750"/>
</p>

Una vez comprobemos que podemos acceder a ArgoCD a través con el Ingress, es recomandable que, por razones de seguridad, creemos **usuarios locales** en lugar de utilizar el usuario `admin` que se nos proporciona al final de la instalación. El uso de usuarios locales nos permite no solo gestionar el acceso de estos de manera independiente, sino también configurar una cuenta API con permisos limitados y generar un token de autenticación con el que el usuario podrá crear proyectos y aplicaciones automáticamente. Para ello, crearemos un recurso *ConfigMap* para cada usuario con el siguiente contenido:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  # En este caso, añadimos un usuario local con los privilegios apikey y login
  #   apiKey: permite generar claves API
  #   login: permite iniciar sesión en la interfaz de usuario
  # El nombre de usuario no debe pasarse de los 32 caracteres
  accounts.lara: apiKey, login
  # La siguiente línea deshabilita el usuario, que por defecto está habilitado
  #accounts.lara.enabled: "false"
```

Al aplicar el fichero yaml, habremos creado un nuevo usuario. Para comprobarlo, emplearemos la interfaz de línea de comandos de ArgoCD, para lo cual tenemos dos opciones:

* Entrar directamente en el **pod argocd-server** y gestionar los usuarios ahí dentro:
```
kubectl exec -it pod/argocd-server-6456fd8bd6-n8x9g  -- bash
```

* O bien, instalar **ArgoCD CLI** en nuestra máquina. Esta es la opción más recomendable, así evitamos tocar el pod y ahorramos tiempo:
```
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.1.5/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```

Iniciamos sesión con el usuario `admin` (el comando para obtener la contraseña del administrador inicial se indica en el apartado de instalación con Autopilot):
```
~$ argocd login www.argocd.org --insecure --grpc-web
Username: admin
Password:
```

* **--insecure**: ArgoCD dispone de un certificado autofirmado, y por defecto intenta realizar una conexión TLS. En un entorno de producción usaríamos un certificado para comunicarnos con el servidor, pero en su ausencia podemos deshabilitar TLS con este parámetro.
* **--grpc-web**: si estamos utilizando un proxy que no soporta HTTP2, en la mayoría de comandos de ArgoCD necesitaremos añadir esta opción para habilitar el protocolo gRPC-web y que no nos aparezca un *Warning*.

A continuación, ejecutamos el siguiente comando para ver la lista de usuarios en ArgoCD:
```
~$ argocd account list --grpc-web
NAME   ENABLED  CAPABILITIES
admin  true     login
lara   true     apiKey, login
```

Como vemos, el nuevo usuario puede iniciar sesión y generar claves API. Con el siguiente comando, podemos obtener más información sobre el usuario:
```
~$ argocd account get --account lara --grpc-web
Name:               lara
Enabled:            true
Capabilities:       apiKey, login

Tokens:
NONE
```

El siguiente paso es generar una contraseña para el nuevo usuario:
```
argocd account update-password \
  --account lara \
  --current-password **************** \
  --new-password ********* \
  --grpc-web
```

* **--account**: nombre del nuevo usuario.
* **--current-password**: contraseña del usuario actual (en mi caso , será la contraseña de `admin`).
* **--new-password**: contraseña del nuevo usuario.

También podemos aprovechar para generar el token de autenticación para el acceso API. Para ello, ejecutamos el siguiente comando seguido del usuario para el cual queremos generarlo (si es el usuario actual, podemos omitir el parámetro `--account`):
```
argocd account generate-token --account lara --grpc-web
```

Hecho esto, cerramos la sesión y salimos del pod.
```
argocd logout www.argocd.org
```

Ya podemos deshabilitar el usuario `admin` añadiendo una línea a nuestra definición de argocd-cm:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  accounts.lara: apiKey, login
  admin.enabled: "false"
```

<br>

## Desarrollo del proyecto

### Crear una aplicación en ArgoCD

En ArgoCD se puede crear una aplicación mediante la interfaz gráfica, por línea de comandos o definiendo los recursos en ficheros yaml que posteriormente le pasemos a Kubernetes. A continuación, veremos los diferentes procedimientos:

#### Por interfaz gráfica

En primer lugar, iniciamos sesión en ArgoCD con el nuevo usuario que nos hemos creado. Después, pulsamos en el bótón **+ NEW APP** y rellenamos los siguientes campos:

**GENERAL**:

* **Application Name**: nombre de la aplicación.
* **Project**: el proyecto en el que queremos guardarla (si no hemos creado ninguno, escribimos "default").
* **Sync Policy**: aquí decidiremos si queremos que el cluster se sincronice con el repositorio de forma manual o automática.

**SOURCE**:

* **Repository URL**: indicamos la URL del repositorio donde tengamos la aplicación.
* **Revision (Branches)**: introducimos el nombre de la rama (*master* o *main*).
* **Path**: ruta dentro del repositorio donde se encuentra la aplicación en sí.

**DESTINATION**:

* **Cluster URL**: URL del cluster donde queremos que se despliegue la aplicación (si es en la misma máquina donde tenemos ArgoCD, introducimos "https://kubernetes.default.svc").
* **Namespace**: espacio de nombre donde se desplegará la aplicación (si no tenemos ninguno en particular, escribimos "default").

<p align="center">
<img src="images/CreateApp.gif" alt="Creación de una aplicación en ArgoCD GUI" width="750"/>
</p>

<br>

#### Por línea de comandos

Para crear una aplicación desde ArgoCD CLI, basta con ejecutar un único comando, con el que introduciremos los mismos valores que en la interfaz gráfica:
```
argocd app create app1 \
--project default \
--repo https://github.com/LaraPruna/PruebaApps/ \
--path app \
--dest-namespace default \
--dest-server https://kubernetes.default.svc \
--sync-policy auto \
--revision main
```

* En primer lugar, después de `argocd app create` indicamos el nombre de la aplicación.
* **--project**: nombre del proyecto creado o, en su defecto, "default".
* **--repo**: URL del repositorio Git donde se encuentra la aplicación.
* **--path**: directorio dentro del repositorio donde está la aplicación.
* **--dest-namespace**: espacio de nombre del cluster donde se va a desplegar.
* **--dest-server**: URL del cluster donde se va a desplegar la aplicación ("https://kubernetes.default.svc" si está en la misma máquina que ArgoCD).
* **--sync-policy**: indicamos si la sincronización se hará manual (`none`) o automáticamente (`auto`,`automated` o `automatic`).
* **--revision**: rama del repositorio, etiqueta, *commit* o versión del chart de Helm con el que se sincronizará la aplicación.

Una vez creada la aplicación, podemos comprobar que se ha creado de la siguiente manera:
```
~$ argocd app list
NAME       CLUSTER                         NAMESPACE  PROJECT  STATUS  HEALTH   SYNCPOLICY  CONDITIONS  REPO                                    PATH  TARGET
guestbook  https://kubernetes.default.svc  default    default  Synced  Healthy  Auto        <none>      https://github.com/LaraPruna/guestbook  app   main
```

Para ver más detalles de la aplicación, usaremos este comando:
```
~$ argocd app get guestbook
Name:               guestbook
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://www.argocd.org/applications/guestbook
Repo:               https://github.com/LaraPruna/guestbook/
Target:             main
Path:               app
SyncWindow:         Sync Allowed
Sync Policy:        Automated
Sync Status:        Synced to main (98304b7)
Health Status:      Healthy
```

<br>

#### Por manifiesto de Kubernetes

Por último, tenemos la opción de definir las características de la aplicación en un fichero yaml y aplicarlo como un manifiesto de Kubernetes. Esta opción resulta bastante útil, y más rápida que los otros procedimientos, cuando queremos crear múltiples aplicaciones a partir de una **plantilla**.

Para mostrar un ejemplo de ello, he creado un [repositorio](https://github.com/LaraPruna/PruebaApps) en GitHub con tres aplicaciones de Nginx definidas en yaml, que he pasado a Kubernetes para que se creen las réplicas, despliegues y pods correspondientes. Para crear estas aplicaciones en ArgoCD, creamos un recurso de tipo *Application* para cada una con el siguiente contenido, adaptando las rutas y nombres:
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app1
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  destination:
    namespace: argocd
    server: https://kubernetes.default.svc
  project: default
  source:
    path: app1
    repoURL: https://github.com/LaraPruna/PruebaApps.git
    targetRevision: master
# Si queremos realizar la sincronización manualmente, la siguiente sección no la incluiríamos.
  syncPolicy:
    automated:
      prune: false
      selfHeal: false
```

Tras pasar a Kubernetes las tres aplicaciones, veremos que ahora nos aparecen en ArgoCD:

<p align="center">
<img src="images/apps1.png" alt="Tres nuesvas aplicaciones en ArgoCD" width="750"/>
</p>

<br>

### Sincronizar una aplicación manualmente
Si hemos establecido una política de sincronización manual, al crear una aplicación en ArgoCD, está nos aparecerá desincronizada (***OutOfSync***). Esto quiere decir que aún no se ha desplegado la aplicación ni se han creado los recursos correspondientes en Kubernetes.

<p align="center">
<img src="images/OutOfSync.png" alt="Estado de la aplicación: OutOfSync" width="550"/>
</p>

```
~$ argocd app list
NAME  CLUSTER                         NAMESPACE  PROJECT  STATUS     HEALTH   SYNCPOLICY  CONDITIONS  REPO                                     PATH  TARGET
app1  https://kubernetes.default.svc  default    default  OutOfSync  Missing  <none>      <none>      https://github.com/LaraPruna/PruebaApps  app1  master
```

Para sincronizar y desplegar la aplicación manualmente, podemos hacerlo de dos maneras:

* **Por interfaz gráfica**

1. Pulsamos en el botón **SYNC** en la aplicación, con lo que nos saldrá una ventana con diferentes opciones de sincronización.
2. Seleccionamos la opciones por defecto (**default**) y sincronizamos todos los manifiestos.

<p align="center">
<img src="images/sync.png" alt="Sincronización" width="500"/>
</p>

Antes de sincronizar la aplicación, mientras aún está *OutOfSync*, también tenemos la opción de ver las diferencias entre el estado del repositorio y el del cluster, mediante el botón **APP DIFF**. En el siguiente ejemplo, se ve que he cambiado la versión de la imagen de Docker:

<p align="center">
<img src="images/AppDiff.png" alt="Diferencias entre ambos estados" width="750"/>
</p>

* **Por línea de comandos**

Aquí bastará con ejecutar un único comando, en el que introduciremos como argumento el nombre de la aplicación:
```
argocd app sync app1
```

Para comprobar que se han creado los recursos, ejecutamos el siguiente comando (si se ha realizado la sincronización correctamente, los pods de la aplicación aparecerán con el estado *Runnning*):
```
~$ kubectl get all
NAME                        READY   STATUS    RESTARTS   AGE
pod/app1-54487f9586-c5psw   1/1     Running   0          4m11s
pod/app1-54487f9586-t7jg5   1/1     Running   0          4m11s
pod/app1-54487f9586-tn58h   1/1     Running   0          4m11s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   39d

NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/app1   3/3     3            3           4m11s

NAME                              DESIRED   CURRENT   READY   AGE
replicaset.apps/app1-54487f9586   3         3         3       4m11s
```

También podemos ver el historial de despliegues de la aplicación. Como acabamos de sincronizarlo por primera vez, solo nos aparecerá uno:
```
~$ argocd app history app1
ID  DATE                            REVISION
0   2022-04-15 13:12:19 +0200 CEST  master (74d2c1a)
```

### Sincronizar una aplicación automáticamente

En el apartado anterior hemos visto cómo sincronizar una aplicación manualmente a través de la interfaz grafica y de la consola. Sin embargo, también podemos configurarla de tal manera que sea ArgoCD el que se encargue sincronizarla automáticamente. Por defecto, la **reconciliación** se realizará cada 3 minutos, momento en el que ArgoCD llevará a cabo las siguientes tareas:

1. Recopilar todas las aplicaciones configuradas para que se sincronicen automáticamente.
2. Buscar el último estado de los repositorios Git correspondientes.
3. Comparar el estado de los repositorios con el del cluster.
4. Tomar una de las siguientes medidas:
	* Si ambos estados coinciden, marcar directamente la aplicación como sincronizada (*Synced*).
	* Si los estados difieren, marcar la aplicación como no sincronizada (*OutOfSync*).

Si no estamos satisfechos con el periodo de reconciliación, podemos cambiarlo editando el recurso ConfigMap que encontraremos en el propio espacio de nombre de ArgoCD.
```
kubectl edit configmaps -n argocd argocd-cm
```

Añadiendo la siguiente línea dentro del apartado "data", cambiamos el periodo de sincronización a 4 minutos:
```
timeout.reconciliation: 240s
```

Tras editar el recurso, tendremos que volver a desplegar el servicio `argocd-repo-server`. Para deshabilitar completamente la sincronización automática, pondremos el valor a 0.

Por último, podemos cambiar la forma en que ArgoCD descubre los cambios en el repositorio Git. En lugar de hacer que ArgoCD revise el repositorio cada cierto tiempo, podemos emplear ***webhooks*** Git, o bien combinarlos con el bucle de reconciliación. De esta manera, nos aseguramos de que se sincronice la aplicación en el caso de que los *webhooks* fallen.

La ventaja de los *webhooks* es que permiten desplegar la aplicación de forma eficiente, al activar la sincronización en el momento de guardar los cambios en el repositorio en lugar de tener esperar que pase el periodo de reconciliación que tengamos configurado en ArgoCD.

ArgoCD soporta *webhooks* de GitHub, GitLab, Bitbucket, BitBucket Server y Gogs. A continuación veremos cómo configurar un *webhook* de GitHub:

#### Crear un webhook en GitHub

Desde el repositorio de nuestra aplicación en GitHub, nos dirigimos a la pestaña "Settings" > "Webhooks" y añadimos uno nuevo:

* **Payload URL**: introducimos la URL de ArgoCD seguido de la ruta `/api/webhook`. Si se encuentra en nuestro host local y no es accesible desde Internet, podemos utilizar [ngrok](https://ngrok.com/download).
* **Content type**: por defecto, nos encontraremos el valor "application/x-www-form-urlencoded", que no está soportado por la librería usada por ArgoCD para gestionar los *webhooks*. Por lo tanto, cambiaremos este valor por "application/json".
* De manera opcional, podemos configurar el *webhook* con un ***secret***, en cuyo caso, introduciríamos un valor arbitrario en este campo. En el siguiente apartado hablamos de esto con más detalle.

<p align="center">
<img src="images/NewWebhook.png" alt="Creando un webhook" width="600"/>
</p>

<br>

#### Configurar el secret del webhook en ArgoCD

La configuración de un *secret* compartido a la hora de crear un *webhook* es **opcional**. ArgoCD actualizará igualmente las aplicaciones relacionadas con el repositorio Git independientemente de que tengan activado o no un control de acceso. El uso del *secret* se recomienda especialmente si Argo CD es accesible públicamente, para evitar sufrir un ataque DDoS. Por lo demás, no habría peligro en saltarnos este paso, puesto que lo único que se consigue con los *webhooks* es actualizar la aplicación (cosa que ya hace ArgoCD cada tres minutos).

En el caso de que hayamos añadido un *secret*, editaríamos el recurso "argocd-secret" en Kubernetes:
```
kubectl edit secret argocd-secret -n argocd
```

Y añadiríamos el apartado `stringData` con el siguiente contenido, dependiendo de dónde hayamos creado el *webhook*:
```
apiVersion: v1
kind: Secret
metadata:
  name: argocd-secret
  namespace: argocd
type: Opaque
data:
...

stringData:
  # github webhook secret
  webhook.github.secret: secret de github

  # gitlab webhook secret
  webhook.gitlab.secret: secret de gitlab

  # bitbucket webhook secret
  webhook.bitbucket.uuid: uuid de bitbucket

  # bitbucket server webhook secret
  webhook.bitbucketserver.secret: secret de bitbucket server

  # gogs server webhook secret
  webhook.gogs.secret: secret de gogs server
```

Al guardar los cambios, estos tendrán efecto de forma inmediata.

<br>

#### Instalación y uso de ngrok

Una vez creado el *webhook* y configurado el *secret* (si procede), instalaremos la herramienta ngrok en la máquina donde tengamos ArgoCD. Podemos hacerlo de varias formas:

* Descargando el **archivo TGZ** y descomprimiendo el binario en /usr/local/bin:
```
cd ~/Downloads
wget https://bin.equinox.io/c/bNyj1mQVY4c/ngrok-v3-stable-linux-amd64.tgz
sudo tar xvzf ngrok-v3-stable-linux-amd64.tgz -C /usr/local/bin
```

* Instalándolo con **APT**:
```
curl -s https://ngrok-agent.s3.amazonaws.com/ngrok.asc | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null
echo "deb https://ngrok-agent.s3.amazonaws.com buster main" | sudo tee /etc/apt/sources.list.d/ngrok.list
sudo apt update && sudo apt install ngrok
```

Hecho esto, entramos en el sitio oficial de ngrok e iniciamos sesión (nos registramos si no lo hemos hecho ya). Nos vamos a la pestaña "Getting started > Setup & Installation", y ejecutamos el comando del paso 2 para conectarnos desde nuestra máquina al proxy de ngrok:

<p align="center">
<img src="images/ngrokAuth.png" alt="Nos conectamos a ngrok" width="750"/>
</p>

Por último, exponemos el servidor de ArgoCD indicando la URL que creamos con el *Ingress*:
```
ngrok http www.argocd.org
```

Entonces nos aparecerá la siguiente salida:
```
ngrok                                                                           (Ctrl+C to quit)
                                                                                                
Session Status                online                                                            
Session Expires               1 hour, 59 minutes                                                
Terms of Service              https://ngrok.com/tos                                             
Version                       3.0.2                                                             
Region                        Europe (eu)                                                       
Latency                       38.516666ms                                                       
Web Interface                 http://127.0.0.1:4040                                             
Forwarding                    https://ec90-188-78-86-122.eu.ngrok.io -> http://www.argocd.org:80
                                                                                                
Connections                   ttl     opn     rt1     rt5     p50     p90                       
                              0       0       0.00    0.00    0.00    0.00
```

Hay que tener en cuenta que ngrok solo lo utilizaremos en un entorno de desarrollo, dado que se trata de una solución temporal para realizar pruebas. Como podemos ver, la URL que se nos ha generado durará un máximo de dos horas.

Ya habríamos terminado de configurar el *webhook*. En el tiempo que dure la sesión creada por ngrok, cuando hagamos un cambio en la aplicación y lo guardemos mediante un `git push`, se nos sincronizará automáticamente en ArgoCD:

<p align="center">
<img src="images/AutosyncApp.gif" alt="Sincronizando Bookmedik" width="750"/>
</p>

<br>

### La salud de la aplicación

Además del estado de la sincronización, ArgoCD monitoriza la "salud" de los recursos en las aplicaciones desplegadas. Los estados de salud se pueden ver tanto en la interfaz gráfica como a través de la consola, y son los siguientes:

* **Healthy**: el recurso funciona correctamente.
* **Progressing**: el recurso no funciona del todo bien, pero puede repararse y alcanzar el estado *healthy*.
* **Suspended**: el recurso está suspendido o pausado. Un ejemplo de esto es una tarea cron.
* **Missing**: el recurso no se encuentra en el cluster.
* **Degraded**: el estado del recurso indica un fallo, o simplemente no se ha podido reparar antes de la sincronización.
* **Unknown**: no se ha podido evaluar el estado del recurso y, por lo tanto, se desconoce.

ArgoCD considera que los recursos están "sanos" si cumplen las siguientes características:

* Los *Deployments*, *ReplicaSets*, *StatefulSets* y *DaemonSets* generados deben ser iguales a los que se desean generar.
* El número de réplicas actualizadas debe equivaler al número de réplicas deseadas.

En el caso de un servicio de tipo *LoadBalancer* o *Ingress*, se considerará que el recurso está sano si la sección `status.loadBalancer.ingress` no está vacía, es decir, debe haber al menos un valor, ya sea una IP o el nombre de un host.

<br>

### Estrategias de sincronización

En los anteriores apartados, hemos visto cómo sincronizar una aplicación en ArgoCD tanto manual como automáticamente, pero ArgoCD también ofrece otras opciones. A la hora de definir una estrategia de sincronización, existen tres parámetros que se pueden cambiar según nuestras necesidades:

1. **Sincronización manual o automática**.
2. **Eliminación automática de recursos** (solo aplicable a la sincronización automática).
3. **Autorreparación del cluster** (solo aplicable a la sincronización automática).

La sincronización manual o automática le indica a ArgoCD qué hacer cuando descubra una nueva versión de la aplicación en el repositorio Git. Si está en modo automático, se aplicarán los cambios y actualizará o creará nuevos recursos en el cluster. Si está en modo manual, ArgoCD detectará los cambios, pero no hará nada en el cluster.

La eliminación automática (*auto-pruning*) le indica a ArgoCD qué hacer cuando eliminamos ficheros del repositorio Git. Si está habilitado, también se eliminarán los recursos correspondientes en el cluster. Si no, ArgoCD no eliminará nada del cluster.

La autorreparación del cluster (*self-heal*) le indica a ArgoCD qué hacer cuando modifiquemos directamente el cluster. Hay que tener en cuenta que, si se quieren respetar los principios de GitOps, no es recomendable que hagamos los cambios manualmente. Si está habilitado, ArgoCD descartará estos cambios y devolverá el estado del cluster a como está en el repositorio.

En este sentido, podemos elegir entre varias combinaciones de estas configuraciones:

|   Estrategia   |    A   |   B  |   C  |   D  |   E  |
|:--------------:|:------:|:----:|:----:|:----:|:----:|
| Sincronización | Manual | Auto | Auto | Auto | Auto |
|  Auto-pruning  | No hay |  No  |  Sí  |  No  |  Sí  |
|    Self-heal   | No hay |  No  |  No  |  Sí  |  Sí  |

La **estrategia A**, en la que ArgoCD no hace nada por su cuenta, es la que se recomienda seguir al principio, especialmente si se desea aplicar la metodología GitOps en un proyecto existente. De esta manera, se puede ir aprendiendo a utilizar ArgoCD sin afectar a los despliegues.

En la **estrategia B** solo está habilitada la sincronización automática. Este sería el primer paso a seguir a la hora de implementar funciones de automatización en GitOps. En el momento en el que se realice un cambio en Git, el cluster se actualizará por sí solo. Por otro lado, el hecho de tener desactivados el *auto-pruning* y el *self-heal* implica que hay que eliminar los recursos a mano y que se pueden realizar cambios directamente en el cluster.

Las **estrategias C y D** son perpendiculares, y pueden actuar como pasos intermedios hacia la **estrategia E**, que es a la que se debería aspirar. En esta última, todo está automatizado en ambas direcciones. Los cambios en Git se reflejan automáticamente en el cluster (incluyendo el borrado de recursos), y los cambios manuales en el cluster son descartados al instante. 

<br>

### Gestión de secrets

Uno de los principios de GitOps es el uso de **Git como fuente de verdad** de todo el sistema. Esto implica almacenar en Git todas las partes de la aplicación: la configuración, los manifiestos de Kubernetes, los scripts de las bases de datos, las definiciones del cluster, etc. Sin embargo, aquí entra en juego un problema de seguridad: ¿cómo almacenamos los *secrets* en GitOps? Pues, en estos momentos, no existe ninguna práctica que esté aceptada de forma general en lo que concierne a la gestión de datos sensibles a través de GitOps. Se pueden emplear soluciones externas, como Hashicorp Vault, aunque técnicamente iría en contra de los principios de GitOps.

Si empezamos un nuevo proyecto desde cero, existen varias maneras de almacenar datos confidenciales en Git para que también podamos administrarlos usando los principios de GitOps, y en todas ellas se almacenan los *secrets* de forma encriptada. De este modo, se pueden gestionar los datos con GitOps y manteniendo la confidencialidad. En este proyecto, usaremos el controlador de **Bitnami Sealed Secrets** para cifrar los datos antes de almacenarlos en Git y descifrarlos antes de que pasen a la aplicación.

#### Cómo funcionan los secrets en Kubernetes

Existen dos tipos de *secrets*: los recursos que creamos en Kubernetes y los "sellados" (*sealed secrets*) introducidos por Bitnami. Los primeros no se encriptan de ninguna manera, y el hecho de codificarlos con base64 no debería contar como una medida de seguridad. Aunque hay formas de encriptar estos *secrets* dentro del cluster, lo que nos interesa es hacerlo fuera, para poder almacenarlos en Git y seguir los principios de GitOps.

El uso de *secrets* en aplicaciones de Kubernetes es muy sencillo. Al igual que con los *ConfigMaps*, podemos montarlos como ficheros en nuestra aplicación o pasarlos como variables de entorno.

<p align="center">
<img src="images/kubesecrets.png" alt="Funcionamiento de los secrets normales" width="600"/>
</p>

Los *sealed secrets* tienen como base los *secrets* nativos de Kubernetes, es decir, tras encriptarlos o desencriptarlos, se utilizan de la manera usual. El controlador de *sealed secrets* de Bitnami es una **extensión de Kubernetes** que se instala en el cluster y que lleva a cabo una única tarea: convertir los *sealed secrets* (que se pueden almacenar en Git) en simples *secrets* (que se pueden usar en la aplicación).

El controlador se instala mediante un chart de Helm:
```
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update
helm install sealed-secrets-controller sealed-secrets/sealed-secrets -n kube-system
```

Una vez instalado, se crearán dos claves:

1. Una clave privada, que se usa para descifrar *secrets*.
2. Una clave pública, que se usa para cifrar *secrets*.

<p align="center">
<img src="images/sealedsecrets.png" alt="Funcionamiento de los sealed secrets" width="750"/>
</p>

Tal y como se refleja en este esquema, el controlador no entra en contacto directo con la aplicación. Simplemente convierte los *sealed secrets* en *secrets* normales, que posteriormente son usados por la aplicación. 

<br>

#### Encriptar los secrets

El controlador que hemos mencionado viene con un comando asociado, llamado **kubeseal**, creado para encriptar los *secrets*. Para instalar este binario, buscamos la última [publicación](https://github.com/bitnami-labs/sealed-secrets/releases) y nos descargamos el paquete .tar.gz de kubeseal.
```
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.17.5/kubeseal-0.17.5-linux-amd64.tar.gz
tar -xf kubeseal-0.17.5-linux-amd64.tar.gz
mv kubeseal /usr/local/bin/
```

Kubeseal utiliza la clave pública creada durante el proceso de instalación para encriptar los *secrets*. Esto implica dos cosas:

1. Kubeseal necesita tener acceso al cluster para cifrar los *secrets*.
2. Los *secrets* encriptados solo se pueden emplear en el cluster en el que se llevó a cabo el proceso de cifrado.

A la hora de cifrar los *secrets* también se tiene en cuenta en espacio de nombre en el que se encuentra el cluster, por lo que se puede afirmar que los *secrets* son **específicos para cada cluster y espacio de nombre**. Si quisieramos usar el mismo *secret* para diferentes clústeres, tendríamos que encriptarlos por cada uno de ellos.

En mi caso, tengo el siguiente secret en formato yaml:
```
apiVersion: v1
kind: Secret
metadata:
  name: mariadb
  namespace: default
type: Opaque
stringData:
    DB_PASSWORD: ***********
    DB_USER: ***********
    MARIADB_PASSWORD: ***********
    MARIADB_ROOT_PASSWORD: ***********
    MARIADB_USER: ***********
```

Para cifrar nuestro *secret* ejecutamos el siguiente comando:
```
kubeseal --controller-name=sealed-secrets-controller --controller-namespace=kube-system --format yaml <bm-secret.yaml> sealed.yaml
```

* **--controller-name**: especificamos el nombre del controlador.
* **--controller-namespace**: indicamos el espacio de nombre donde se encuentra el controlador.
* **--format**: indicamos el formato en el que se encuentra nuestro fichero *secret* (yaml o json).
* Después, indicamos el nombre de nuestro fichero entre los signos "< >".
* Por último, escribimos el nombre del fichero de destino.

Al ejecutar el comando, se creará un nuevo fichero con los valores secretos encriptados:
```
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: mariadb
  namespace: default
spec:
  encryptedData:
    DB_PASSWORD: AgB/Ix/zzCWcslz/iCyBuEkedlgm1+KaR2U11jLvhxXtRA9GbPjx2dOXehDRAL6kA91rgufz7AS/14ZenKqnHvIBxWPVdakcBgF8fENCQHmYkhsWXhBvGF/mYg4RFqpwL4ty6BYevc2QQ2Z2UAPBXXUb+9NG3LQDJo0u04XuZL1on+lniwsY/IonN3RN9MhkZ2l1m/H3m1E5R3ULkaRKS16XXbBo6VUCqw16BjjgN1kHwNtMSKxKKqyErJa3jigVKtCuOG60flONnL08cSwpE4WBKw7fsFiYMI3kQ/k+Zj7P9a52kedi6amAq/V+STKJjr7bZPK1LWQ0RMCvJrlfWb+PtRsEIRyb4sGeA9eoG7Mokvf4g/zxnhVLcU04QRnc4HSg/gchaSrpmFAfpS8bFVSD9rzrg1cIJfX3/vDINlwjRtCP00RyMdo1xXSqQdRaANWnAWl4Kps3b+DBTvrMZ+U7W0rWySTlyqhGVZt6csK84qbR242ucrhrsUiN38GKcUEVJoNEJ8LQNLBpnD2Ry25d/GLphwjOH/XOLqiuspFh8QEpSIZNfN2J5zEPYj8ITZ0sFg/8rUOhufawwvHeIVMnCVBp+J3EJakYout758Cu/zctzJi7FLU7DxQf88xZ/eriCk3ZWjpTfhKq4k5PAt8dAuT9FZY35SMF27EMyoa+hlA9VtgND/0XacZQLWGlxoziyBtPs0ta
    DB_USER: AgDC6SNr7FQli8grygsiMMcftHZSXOZsVGx4/v58laIkgrYEyk/n02PrFBt1zyqspegImxzUtpgLIbvXbIPTEZ7qfJ6+ZdvELNKg5Bkg3ncTOzbxwyHuCbYeITh72xO5j4UCN/QjT6I2htMnW7qiiXzk7Z5/ZQXVbIfwarsYFyxZb6WU+tPhVVCo0fDnfwUXNaEkzA9SLwczB9Ur6XwxCvrAfKq5s/CIG+AYBO08WW8LPPsTM25Yk/e8QxwhGaGjtIdhUxIxX9399WoNkjaGDpvr3rEWMsbR6Yn+3bBUsXHUbgRoaHncRfbUXIcPVcODIGwnUxVwJ7ZGDA3qTrjn8bdGyxynDiwUAPb7HoZP9Mk87xa8zedixI+4or29nmkNmPZdh3fHQR5CJPBdn0AJpPW1OYgY0i6Arli5pynEUomkF/yIJV94+fjpinp9t+VHAKy8eZ3U2Mo364PF9mytvAlDq/NFU1f30sCnuXR8mCM8Hpx1zIYUvoLheA2wwsua937bxX7dxFCPhHxn/i+fG6zDKeu9sAHHryQBiIYBYttx2G1YJ1l9r813Qenn1OhhY5/mOO+cbDhXfvtMtHKAR/KZd7kdzJKhriiz4FmePRhOlNlt3RRzWaOgp8szegNpUUxhaKT0jaSh4N7XbuXHhA4spEsNRkwKKdkqhBiBr9wfYSXNFF6nVKlpcJy8fXldA9bZ82IjoCCq
    MARIADB_PASSWORD: AgA8j0QHq9NOIv3N2GKfzX9W8jaTnd8erw2Kl8FArFTskKdZE2A0AwUVRenBdSLOBtpJ4tYI8WkMruM9DB6izXbzndtlM/tFmBlOEkumAMQCU9kbTcSRWYamKSyWeJEFyDC+GBEBC0APl07tzg3+Ojx+w8ns/7YbDouQ0RvifX1ODRSpPbQxb5nUjTOHT/Oy96B0uecV/S4lbuQmnbQDBr72tXqaBGqFt0PLYMnUFCMu8j5npHG/any+gVchOIEC2lpKLh0hGl4ZiJjMiIqfmnPl6VzWZA9lyhGPw71qfLnnjyJ3UeJaMfW+LooWa1VMGfeRvjoIqFfYo13ciPa5pClyZxYRg8AssznTnZRGmoHP0oc1zYluHmpfaVcTbTIhJg+YT/QjWsdmAmNN35my+Ffz6XatM7eWWvxIXGFSwbpnfeT8Jut7gx8SX4oSbIK3TV9qlG9MFoiJL9aFRA9wel0AgGPHtcSV+pIE3o3zUE+VMKAmYR4QrR1VUpAtdllzHETq5l0gfsVsBGP0z1K6spbGBE9tW9FDY1uel1jZ9eY9kbRliuPO4Xay34trH5VLtonXX4M77nGWaj0znqLoD8RgaU615So9FvL34XPi01F2vTN65e6zlDfjN2In0xCiF9c1we8fqPmuZWwyOB+bTysts/fJHIU8ss9AEPDxBitdm9PkSijwkLMu5AIQWNeRGTPzUg/wH0j6
    MARIADB_ROOT_PASSWORD: AgCzD7Eh92kDg3mg98uDEIuc+fbHwhUiLTp4LKS414qE3agUU635DOaP7vQJXkn/jGWXRMmrUdqvG+p+mW9Z3oedljr6Y8kSU76E5vFPs+PFyBjWtIAa/lkyzgxcbbfB6+/eoUtQc6TU7FI/1BVbwumk7MeIpzo0VVhNbwrbbvWQGbcREgj12chtBDCvqd2zlojLwUmu/sorbtgLgr6E0XB3rDTbFzYI1YXNdfTsSdkS67DEZX1UD/jgQH75IpxvMwOfj8zRSNoghMdUrDS9UXOoSek582dZVOfPxbcZyuUCA37PJwvNTTtFTKGVjNNHPBJm6/952rououaROwGdzsrtwZ0bgRc2+ep+T1cPTJFOuUUCbej8Jt6RwU7Yr41msOhPyX/d0VI3J+0c2Sy4EdManMaOUBkKGP9if90wbc7G3gCM/6IycoRRibIiV4G7kEoRBA9SuimfkoBTJV28yDh1xKwUn3kAV3vo2+zCeIb/N/Mq8bRC243LVIHgC2sK+a6xPgkTaGEp1dFBDwV53un5DSBGMkReSck/KEimHNHkJF1e2uxbwUvHyIIr2eihNHYG4yhijeDrhIaZSnZxse17dYrtCDQOETWABcmASVODgyBRzkVLSHRU3jsezDoW+VPczEJVcHIWLPkQLFZ5mdImC4+ZMFzm0JPZqDGI95hgtUKo7lmfvAYn1Jp/B7e/FyzkF/UT
    MARIADB_USER: AgCMg6T6By/63q9flS0NtP07r+iEX8bI1egFddczNjRBQT2wu7PWXzw46IO8YQZKaecTz2kb0Xug3m7c4dLtHU1OjZBZNBzmj64GpKmjbyo38muMlAs4XNn3XxzRATy15yAVwOMZgHXC86M2ck8sCG0d28qJnXUEo5GctMI+mU6kpyDYVT7mK0ZRF2AKXZDd6KCdFqfGi+6HYeh9rf84h0r84HyhKFIt/rvZmcgK+AX2t4bG6i0dr9wzOV9E14RyV5kCJZzkUUCHacrBzgkeEl2gDMp4KpqczGBvSfQlYBO4birLsbR2o+9cegHdjBFPCP+RZkriDzoWPg+nSfbg+VFrsoSPNl/rzDEwgpvF06QZ5poGYC5uxOmkn9z6KfLmrxpEU18HpXVF1MRv2DfB82EEGvDjl6YLsR4ewXjumNGjT0IzjIcILqYFJ0VA28YqRzqzifvoS/X15iDitzxgncOqYkMaWB91fEnVtdGRzbZcYBKMOcfmH0xzfq9dSyg/eONFD7FKjqKG5TUB9FUWst7q3VdO3P+oLi43JXdSRrz2hDALlFvncjXPLNSyq64QLG+5FF772OHD6excvbPdkQRWGjJI/zWbC+qtXVUAY2+XVVphvh7LoWK6cAHNmfhrSi5EwLHXHE+CXT60gilHlcxrFTTZQQEau/u69Gt7FyGLSWLMypAPjMyoiZuTj5o5d+2iSy+El6Us
  template:
    data: null
    metadata:
      creationTimestamp: null
      name: mariadb
      namespace: default
    type: Opaque
```

Como podemos comprobar, este nuevo fichero se puede almacenar en Git sin comprometer nuestros datos sensibles. Ahora podemos aplicarlo en el cluster:
```
~$ kubectl apply -f sealed.yaml
sealedsecret.bitnami.com/mariadb created
```

Este *secret* ya es parte del cluster, y el controlador lo desencriptará en cuanto la aplicación lo necesite.

<br>

## Conclusiones y propuestas adicionales para el proyecto

<br>

## Bibliografía

AnAr Solutions. (2021, 11 noviembre). Compare Infrastructure as Code - IaC vs Gitops. AnAr Solutions Pvt. Ltd. Recuperado 16 de marzo de 2022, de https://anarsolutions.com/iac-vs-gitops/

Andrada Prieto, J. (s. f.). ¿Qué es GITOPS? Viewnext. Recuperado 15 de marzo de 2022, de https://www.viewnext.com/que-es-gitops/

ArgoCD. (s. f.). Ingress Configuration - Argo CD - Declarative GitOps CD for Kubernetes. Argo CD. Recuperado 11 de abril de 2022, de https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/

argoproj-labs. (2022, 7 abril). GitHub - argoproj-labs/argocd-autopilot: Argo-CD Autopilot. GitHub. Recuperado 11 de abril de 2022, de https://github.com/argoproj-labs/argocd-autopilot

bitnami-labs. (2022, 20 abril). GitHub - bitnami-labs/sealed-secrets: A Kubernetes controller and tool for one-way encrypted Secrets. GitHub. Recuperado 23 de abril de 2022, de https://github.com/bitnami-labs/sealed-secrets#installation-from-source

Decoster, J. (2022, 19 febrero). ArgoCD + Minikube + Ngrok + Github Webhook - jerome.decoster. Medium. Recuperado 17 de abril de 2022, de https://medium.com/@jerome.decoster/argocd-minikube-ngrok-github-webhook-3cd0cc15d559

Dubey, A. (2022, 20 enero). All About ArgoCD, A Beginner’s Guide. DEV Community. Recuperado 2 de abril de 2022, de https://dev.to/abhinavd26/all-about-argocd-a-beginners-guide-33c9Jerez, Á. G. (2020, 29 mayo).

GitHub. (s. f.). Crear un token de acceso personal. Github Docs. Recuperado 11 de abril de 2022, de https://docs.github.com/es/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token

GitHub. (s. f.). Crear webhooks. GitHub Docs. Recuperado 17 de abril de 2022, de https://docs.github.com/es/developers/webhooks-and-events/webhooks/creating-webhooks

Iesgn. (2022, 9 marzo). GitHub - iesgn/curso_kubernetes_cep. GitHub. Recuperado 11 de abril de 2022, de https://github.com/iesgn/curso_kubernetes_cep

Implementando GitOps con ArgoCD. Adictos al trabajo. Recuperado 2 de abril de 2022, de https://www.adictosaltrabajo.com/2020/05/25/implementando-gitops-con-argocd/

Lingeswaran, R. (2021, 26 junio). GitOps vs Infrastructure as Code. UnixArena. Recuperado 15 de marzo de 2022, de https://www.unixarena.com/2021/06/gitops-vs-infrastructure-as-code.html/

Ponce, J. A. (2020, 1 septiembre). Ngrok: una herramienta con la que hacer público tu localhost de forma fácil y rápida. SDOS. Recuperado 17 de abril de 2022, de https://www.sdos.es/blog/ngrok-una-herramienta-con-la-que-hacer-publico-tu-localhost-de-forma-facil-y-rapida

Sharma, A. (2021, 12 agosto). Automatically create multiple applications in Argo CD. Opensource.Com. Recuperado 14 de abril de 2022, de https://opensource.com/article/21/7/automating-argo-cd

Weaveworks. (s. f.). GitOps what you need to know. Recuperado 15 de marzo de 2022, de https://www.weave.works/technologies/gitops/

Weaveworks. (2022, 8 marzo). The Evolution of Configuration Management: IaC vs GitOps. Recuperado 16 de marzo de 2022, de https://www.weave.works/blog/evolution-configuration-management-iac-vs-gitops