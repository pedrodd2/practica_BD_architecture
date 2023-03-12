# Brainstorm

- Control del stock en tanques de almacenamiento en tiempo real, para ello tienes que registrar los niveles.
- Poder predecir cuanto vas a tener que comprar/producir.
- Poder predecir ventas.
- Poder predecir perdidas para realizar reparaciones.
- Poder tener registro de variables (presión temperatura) para identificar posibles catástrofes ocurridas

# Diseño del DAaaS
## Definición la estrategia del DAaaS
*Actualmente en las refinerías se tiene un sistema de monitoreo y almacenamiento basado en scada, su arquitectura más típica es la siguiente:*

![](Aspose.Words.97aecb1e-16b5-45d7-b3d4-fda404f0d9e4.001.png)*

*Este sistema se encuentra en la red local de la refinería y para acceder son los operarios los exportan la información, pongamos unos ejemplos para aclarar como se accede a la información desde afuera de la red interna:*

`	`*-Imaginemos que la refinería subcontrata una mejora de un proceso (por ejemplo se requiere añadir una bomba para aumentar la presión de un sistema y así mejorar el rendimiento), esta ingeniería para saber las condiciones del proceso actual tiene dos fuentes una es el P&ID(process and instrument diagram) el cual es un plano donde están registrado las condiciones del proceso, y otra para comprobar que el proceso actualmente funcione de acuerdo a el P&ID es preguntar a la refinería, la cual le pone en contacto con un operario que te proporciona un pantallazo del sistema(en la imagen anterior ‘panel de visualización’) y un histórico que se obtiene de la base de datos de este sistema on-premise.*

`	`*-Para conocer la cantidad de producto almacenado que tiene la planta desde las oficinas centrales: un operario generaría un archivo Excel con cada tanque y su capacidad y se lo enviaría vía email. Como se observa la respuesta es bastante lenta ya que la oficina central debe de ponerse en contacto con la refinería esta generar el informe y luego mandarlo, para una refinería bien, pero si tienes 20 ... el proceso se ralentiza.*

*Por lo que nuestra estrategia se centrara en conectar el sistema actual a la nube para proporcionar acceso tanto en tiempo real como a históricos de la información generada en la refinería.*

## Arquitectura DAaaS
*-[MTU con Apache PLC4X]= En la MTU(unidad terminal maestra) instalaremos Apache PLC4X que es un conector Kafka el cual nos permite conectar nuestro Kafka clúster a el sistema scada. Tendremos un evento por cada sensor en la planta*

*-[Dataproc con Apache kafka]=Lanzaremos una cluster de dataproc con  Apache Kafka el cual usaremos como gestionador del stream de datos. En este cluster también instalaremos el conector con Google cloud storage usando las librerías de Python confluent-kafka google-cloud-storage. Tambien setearemos Kafka para que pueda ser leído por Django.*

*-[Google cloud storage]= Lo utilizaremos como data lake para nuestro almacenamiento de datos, haciendo una vista general tendremos cientos de archivos de cada sensor en la planta con sus valores, pero estos no tienen ninguna relación entre sí.*

*-[Google cloud function]= Usaremos una función escrita en Python para subir los archivos desde GCS a Hive y para enviar los archivos generados el dia anterior a otro tipo de almacenamiento más barato(coldline).*

*-[Google cloud scheduler]= Lo usaremos para ejecutar la función anterior cada día.*

*-[Dataproc con hive y hadoop]= Crearemos un cluster con hive y hadoop que conectaremos con GCS, aquí podremos definir relaciones así como limpiar los datos. Lo utilizaremos para el análisis de datos históricos (cosas que ya han pasado).* 

*-[Django]= instalaremos una aplicación web(será parecido al panel de visualización que esta en refinería) con Django para mostrar los datos en tiempo real , conectaremos Kafka con Django usando la librería Kafka-python. Lo montaremos en una VM en Google cloud una para cada refineria. Lo haremos así tanto para facilitar el acceso del usuario (y permitirle también que acceda a unas refinerías si y a otras no) como para facilitar el desarrollo y la escalabilidad del proyecto.*

## DAaaS Operating Model Design and Rollout
1. La operativa consta de un stream de datos el cual opera en tiempo real y siempre. La ruta del dato seguida por ejemplo para un transmisor de nivel instalado en un tanque de almacenamiento seria la siguiente.
   1. Se crea la señal en el transmisor.
   1. Se envía al MTU.
   1. En el MTU mediante *PLC4X se envía a Kafka*
   1. *Desde Kafka se envía Django*
   1. *Django despliega un interfaz web donde se puede ver para ese tanque el nivel que tiene con lo que la persona que esta inspeccionando puede saber qué nivel tiene el tanque, evitando los inconvenientes explicados en el punto de diseño.*
1. *La otra operativa que tenemos consta de un data warehouse donde se pueden consultar el histórico de datos y analizar los datos. Para ello vamos a distinguir dos partes, la primera los datos que se producen en stream y otra la carga de estos datos al data warehouse.*
   1. *Los datos que se producen en stream sigue la misma operativa mostrada en el punto 1 anterior hasta el punto 1.d, el cual se debe de sustituir por que los datos viajaran desde Kafka al Google Cloud Storage.*
   1. *Para la carga de datos desde GCP a Hive usaremos una Google Cloud function, para ello también tendremos usaremos Google cloud scheduler para lanzar esta función cada día.* 


## Desarrollo de la plataforma DAaaS. (ligera descripción del desarrollo)
Para el desarrollo de la plataforma usaremos el principio de divide y vencerás. Ya que todos los componentes de la plataforma son escalables.

1) Empezaremos por una refinería y para ello usaremos un sensor de ejemplo y buscaremos que todos los componentes de la arquitectura trabajen adecuadamente ya que esto es el core del proyecto. Una vez veamos que nuestro stream de datos desde el inicio al final y la carga de datos al Hive funciona correctamente pasaremos a los siguientes pasos.
1) Estos pasos los podremos ejecutar en paralelo:
   1) Creación de la aplicación web, para ello contaremos con pantallazos de ejemplo del sistema de visualización de la refinería así como los P&ID, cabe mencionar la importancia de los P&ID ya que en ellos aparecen todos los equipos de la refinería así como todas las señales que emiten los instrumentos, tanto las señales como los equipos están identificados mediante un KKS que será de vital importancia para poder conectar los diferentes componentes de la arquitectura, usaremos este mismo sistema de codificación para unir las diferentes señales entre nuestros componentes.
   1) Modelado de Hive: Desarrollaremos y crearemos un esquema con las tablas que relacionen las señales con los equipos y la zona de la refinería.
1) Conexión de cada señal, mediante el sistema de kks uniremos los diferentes componentes.

# Link a Diagrama:



