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
	1. [Instalación de ArgoCD mediante declaraciones](#instalación-de-argocd-mediante-declaraciones)
	2. [Instalación de ArgoCD mediante Autopilot](#instalación-de-argocd-mediante-autopilot)
5. [Desarrollo del proyecto](#desarrollo-del-proyecto)
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
2. Nadie tiene acceso directo al cluster de Kubernetes, y hay un segundo repositorio Git con todas las declaraciones que definen la aplicación.
3. Otra persona o un sistema automatizado cambia las declaraciones en el segundo repositorio Git.
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
2. Almacenamos las declaraciones en Git, sean del tipo que sean. ArgoCD es bastante abierto a este respecto, soporta tanto declaraciones simples de Kubernetes como *charts* de Helm, definiciones de Kustomize y otros sistemas de plantillas.
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

### Instalación de ArgoCD mediante declaraciones

Si solo queremos ArgoCD para hacer pruebas y experimentar con la herramienta, podemos desplegar ArgoCD directamente con el fichero install.yaml que nos ofrece ArgoCD en su [repositorio de GitHub](https://github.com/argoproj/argo-cd/tree/master/manifests). Creamos un espacio de nombre y aplicamos dicho archivo en el mismo.
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

En el mismo repositorio tenemos disponible otro archivo yaml llamado "namespace-install.yaml", con el que instalaremos ArgoCD como en el caso anterior. La única diferencia es que este está pensado para entornos con muchos usuarios repartidos entre diferentes equipos y proyectos, entre los cuales se han dividido los recursos del cluster mediante el uso de **espacios de nombre**. En este sentido, namespace-install.yaml permite instalar ArgoCD en un espacio de nombre concreto, y no requiere que tengamos privilegios a nivel de cluster. Esta es la mejor opción si queremos instalar ArgoCD en un solo cluster pero que también pueda gestionar otros externos.

<br>

### Instalación de ArgoCD mediante Autopilot

<p align="center">
<img src="images/argo_autopilot.png" alt="ArgoCD en piloto automático" width="450"/>
</p>

Si queremos trabajar directamente en un entorno de producción, tenemos a nuestra disposición la herramienta [Autopilot](https://github.com/argoproj-labs/argocd-autopilot), que además de instalar la propia aplicación de ArgoCD y desplegarla en un cluster de Kubernetes, crea un repositorio (o un directorio en uno que ya exista) de GitOps para que se gestione a sí misma.

Una vez instalado ArgoCD, ya podremos crear nuestros proyectos y aplicaciones con esta misma herramienta, que guardará todas las definiciones en el repositorio creado. Por su parte, ArgoCD notará los cambios en dicho repositorio y los aplicará al cluster, dando lugar a un proceso de despliegue continuo.

<br>

## Desarrollo del proyecto

<br>

## Conclusiones y propuestas adicionales para el proyecto

<br>

## Dificultades encontradas en el proyecto

<br>

## Bibliografía

AnAr Solutions. (2021, 11 noviembre). Compare Infrastructure as Code - IaC vs Gitops. AnAr Solutions Pvt. Ltd. Recuperado 16 de marzo de 2022, de https://anarsolutions.com/iac-vs-gitops/

Andrada Prieto, J. (s. f.). ¿Qué es GITOPS? Viewnext. Recuperado 15 de marzo de 2022, de https://www.viewnext.com/que-es-gitops/

Dubey, A. (2022, 20 enero). All About ArgoCD, A Beginner’s Guide. DEV Community. Recuperado 2 de abril de 2022, de https://dev.to/abhinavd26/all-about-argocd-a-beginners-guide-33c9Jerez, Á. G. (2020, 29 mayo).

Implementando GitOps con ArgoCD. Adictos al trabajo. Recuperado 2 de abril de 2022, de https://www.adictosaltrabajo.com/2020/05/25/implementando-gitops-con-argocd/

Lingeswaran, R. (2021, 26 junio). GitOps vs Infrastructure as Code. UnixArena. Recuperado 15 de marzo de 2022, de https://www.unixarena.com/2021/06/gitops-vs-infrastructure-as-code.html/

Weaveworks. (s. f.). GitOps what you need to know. Recuperado 15 de marzo de 2022, de https://www.weave.works/technologies/gitops/

Weaveworks. (2022, 8 marzo). The Evolution of Configuration Management: IaC vs GitOps. Recuperado 16 de marzo de 2022, de https://www.weave.works/blog/evolution-configuration-management-iac-vs-gitops