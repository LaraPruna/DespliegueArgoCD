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
	2. [Sincronizar una aplicación en ArgoCD](#sincronizar-una-aplicación-en-argocd)
	3. [El bucle de reconciliación](#el-bucle-de-reconciliación)
6. [Conclusiones y propuestas adicionales para el proyecto](#conclusiones-y-propuestas-adicionales-para-el-proyecto)
7. [Dificultades encontradas en el proyecto](#dificultades-encontradas-en-el-proyecto)
8. [Bibliografía](#bibliografia)

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
argocd app create guestbook \
--project default \
--repo https://github.com/LaraPruna/guestbook/ \
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

### Sincronizar una aplicación en ArgoCD

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

### El bucle de reconciliación

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

#### Crear un *webhook* en GitHub

Desde el repositorio de nuestra aplicación en GitHub, nos dirigimos a la pestaña "Settings" > "Webhooks" y añadimos uno nuevo:

* **Payload URL**: introducimos la URL de nuestro repositorio seguido de la ruta `/api/webhook`.
* **Content type**: por defecto, nos encontraremos el valor "application/x-www-form-urlencoded", que no está soportado por la librería usada por ArgoCD para gestionar los *webhooks*. Por lo tanto, cambiaremos este valor por "application/json".
* De manera opcional, podemos configurar el *webhook* con un ***secret***, en cuyo caso, introduciríamos un valor arbitrario en este campo. En el siguiente apartado hablamos de esto con más detalle.



<br>

## Conclusiones y propuestas adicionales para el proyecto

<br>

## Dificultades encontradas en el proyecto

<br>

## Bibliografía

AnAr Solutions. (2021, 11 noviembre). Compare Infrastructure as Code - IaC vs Gitops. AnAr Solutions Pvt. Ltd. Recuperado 16 de marzo de 2022, de https://anarsolutions.com/iac-vs-gitops/

Andrada Prieto, J. (s. f.). ¿Qué es GITOPS? Viewnext. Recuperado 15 de marzo de 2022, de https://www.viewnext.com/que-es-gitops/

ArgoCD. (s. f.). Ingress Configuration - Argo CD - Declarative GitOps CD for Kubernetes. Argo CD. Recuperado 11 de abril de 2022, de https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/

argoproj-labs. (2022, 7 abril). GitHub - argoproj-labs/argocd-autopilot: Argo-CD Autopilot. GitHub. Recuperado 11 de abril de 2022, de https://github.com/argoproj-labs/argocd-autopilot

Dubey, A. (2022, 20 enero). All About ArgoCD, A Beginner’s Guide. DEV Community. Recuperado 2 de abril de 2022, de https://dev.to/abhinavd26/all-about-argocd-a-beginners-guide-33c9Jerez, Á. G. (2020, 29 mayo).

GitHub. (s. f.). Crear un token de acceso personal. Github Docs. Recuperado 11 de abril de 2022, de https://docs.github.com/es/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token

Iesgn. (2022, 9 marzo). GitHub - iesgn/curso_kubernetes_cep. GitHub. Recuperado 11 de abril de 2022, de https://github.com/iesgn/curso_kubernetes_cep

Implementando GitOps con ArgoCD. Adictos al trabajo. Recuperado 2 de abril de 2022, de https://www.adictosaltrabajo.com/2020/05/25/implementando-gitops-con-argocd/

Lingeswaran, R. (2021, 26 junio). GitOps vs Infrastructure as Code. UnixArena. Recuperado 15 de marzo de 2022, de https://www.unixarena.com/2021/06/gitops-vs-infrastructure-as-code.html/

Sharma, A. (2021, 12 agosto). Automatically create multiple applications in Argo CD. Opensource.Com. Recuperado 14 de abril de 2022, de https://opensource.com/article/21/7/automating-argo-cd

Weaveworks. (s. f.). GitOps what you need to know. Recuperado 15 de marzo de 2022, de https://www.weave.works/technologies/gitops/

Weaveworks. (2022, 8 marzo). The Evolution of Configuration Management: IaC vs GitOps. Recuperado 16 de marzo de 2022, de https://www.weave.works/blog/evolution-configuration-management-iac-vs-gitops