Este proyecto tiene como objetivo contar con las fuentes de información centralizadas para generar estándares de programación para los diferentes lenguajes de programación.
El primer paso es investigar la información de los estándares de programación en general, sin ser para un lenguaje en específico.
El segundo paso es investigar los estándares de programación específicos para el lenguaje que se desea trabajar.
Este proyecto será fuende de consulta de información de referencia para otros proyectos que necesiten construir sus propios estándares de programación, tomando esta guia como una referencia base.
Serán agentes de IA los encargados de construir dichos estándares.

En el directorio common estan los estándares de programación generales, que aplican para todos los lenguajes.

Despues, se debe crear un sub directorio para cada lenguaje de programación.
La idea es que no haya un estándar fijo en este proyecto, sino que haya un conjunto de diferestes estándares que sirvan de referencia, que el cliente pueda tomar los que desee y construir su propio estandar de desarrollo.

Por lo tanto es mejor en ves de generar un solo documento con diferentes secciones, generar un documento por cada sección.

Para mantener el orden, se crearán sub carpetas dentro de cada lenguaje, así por ejemplo, para el lenguaje Java se crea una carpeta java, dentro de ella, para reglas generales van en la carpeta common, estandares de base de datos en otra, etc.

La idea es que puedas como base, todos los estándares de un lenguaje y solo sobreescribir, eliminar o agregar lo que necesites.

Estándares específicos sobre escriben estandares generales.

La idea es que cuando un agente ejecute el skill principal generador de estandares, el parámetro principal será el lenguaje, luego se le mostará una lista con cada una de las sub carpetas del lenguaje con un checkbox, teniendo aún la posibilidad de navegar dentro y marcar o desmarcar cada uno de los estandares internos, que como ya dijimos será un archivo para cada sección de dicho estandar.

Escribe un Readme.md detallando este proceso que te describo, para que sirva de contexto a los agenets de IA que consulten el contenido de este repositorio.


El archivo ContitutiveElements.md no se va a ir a ningun lado, es solo la guía para la elaboración de un estandar.
Por lo tanto, considero que solo debe existir un comando en este repositorio:
/standard_research

Este comando recibirá como parámetro el lenguaje de programación acerca del cual se quiere hacer una investigación para recopilar sus estandares de programación, para esto, antes que nada, debe leer las instrucciones de que es lo que tiene que hacer. Para esto, con base en el Readme.md genera un archivo en inglés: Instructions.md con las instrucciónes que debe seguir el agente para recopilar los estándares del lenguaje de programación específico.

Entonces el comando /standard_research leerá las instrucciones de este archivo y sabrá que tiene que seguir el flujo:
1. Validar si existe ya la información del estandar para este lenguaje, si existe mostrar la fecha de creación, ultima revisión y preguntar si desea que su agente de IA revise la recopilación de información ya existente, esto debido a la rápida evolución de agente de IA, lo cuál permitira perfeccionar aún más la recopilación de información ya existente. Si no quiere hacer una revisión, el comando simplemente termina.
2. Si no existe, Leerá el archivo ContitutiveElements.md
2. Comenzará a realizar la investigación acerca de cada punto indicado en el ContitutiveElements.md
3. Dejar los documentos por cada tema investigado en la carpeta correspondiente.

Escribe el comando, agrega la referencia de uso en el Readme.md es lo único que tiene que hacer el cliente que quiera usar este repo.  Así ejecutar este comando debe dar como resultado en un directorio poblado con información de estándares de programación para un lenguaje específico.

El nombre de este proyecto será: StandardLibrary

Solo es eso, una librería de estándares.

En un proyecto independiente llamado: StandardBuilder será el encargado de leer este StandarLibrary y generar un output ya con los skills, comandos, workflows, etc. todas las herramientas necesarias para que un agente de IA siga los estándares de desarrollo.  Agrega una nota en el Readme.md detallando esto, para que el cliente sepa que el siguiente paso es buscar el repositorio StandardBuilder.

Agrega al principio de todos los archivos de este proyecto la sección:
´´´
---
title: {Document name}
status: draft
version: 0.0.1
created: 2026-07-05
updated: 2026-07-05
---
´´´

Para llevar el seguimiento de los cambios.


El comando /standard_research debe escribir un archivo .md en el deberá escribir una tabla y el contenido será la lista de puntos de investigación del archivo ConstituentElements.md para poder llevar un seguimiento de lo que los agentes están realizando en ese momento.
La primera columna es un icono para indicar el estatus de la investigación, vacio que es sin investigar, icono de ok que es realizada, un icono de error si hubo error, un icono de warning si hubo algun problema.
La siguiente columna es una descripcion corta del punto que se investiga.
La siguiente columna es una descripción larga que describa el problema encontrado en la investigación, o vacío si esta ok.
Las últimas columnas serán hora de inicio y hora de fin de investigación.

De esta forma el proceso puede cortarse y re iniciarse donde se quedo.
Este comando no tiene que ser secuencial, debe poder ejecutarse con N sub agentes investigando simultáneamente, el segundo parámetro del comando indicará cuantos sub agentes realizarán la investigación, si no proporciona el parámetro, por default serán 3.