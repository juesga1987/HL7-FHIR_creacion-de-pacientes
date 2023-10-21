<h1 align=center> HL7-FHIR_creacion-de-pacientes </h1>
<p align=center><img src=https://www.goup.cloud/wp-content/uploads/2020/07/Go_Up_Cloud-300x210.png></p>

El presente proyecto tiene como objetivo la captura de un documento identidad para ser procesado mediante técnicas de **Inteligencia Artificial** con el fin de obtener la lectura de sus datos. Dicha información generada es almacenada en un **Repositorio Clínico** llamado **FHIR** a través de **Azure Health Data Services**. Esta información resultante será de consulta inmediata desde cualquier entidad conectada a FHIR a través de Azure Health Data Services.

El siguiente diagrama muestra el camino desde la toma de imagen hasta el almacenamiento o consulta del paciente:


<p align="center">
  <img src="arquitectura Goupcloud.png" alt="arquitect"  width="80%"/></p>



Los procesos que veremos son:
1. [Captura del documento](#Captura-del-documento)
2. [Almacenamiento del documento capturado en Microsoft Azure Blob Storage](#Almacenamiento-del-documento-capturado-en-Microsoft-Azure-Blob-Storage)
3. [Procesamiento de la imagen almacenada mediante Inteligencia Artificial](#Procesamiento-de-la-imagen-almacenada-mediante-Inteligencia-Artificial)
   1. [Pre procesamiento de la imagen mediante OCR para el selector de nacionalidad y posterior extracción de datos](#Pre-procesamiento-de-la-imagen-mediante-OCR-para-el-selector-de-nacionalidad-y-posterior-extracción-de-datos)
   2. [Transformacion de la captura de datos a formato HL7 FHIR](#Transformacion-de-la-captura-de-datos-a-formato-HL7-FHIR)
4. [Almacenamiento de la data transformada en Microsoft Azure Blob Storage](#Almacenamiento-de-la-data-transformada-en-Microsoft-Azure-Blob-Storage)
5. [Creación y despliegue del Azure Function para la automatización de procesos](#Creación-del-Azure-Function-para-la-automatización-de-procesos)
   1. [Arquitectura Azure Function](#Arquitectura-Azure-Function)
      + [Creación almacenamiento y contenedores en Azure](#Creación-almacenamiento-y-contenedores-en-Azure)
      + [Azure function en ambiente local con Visual Studio Code](#Azure-function-en-ambiente-local-con-Visual-Studio-Code)
6. [Uso de Docker para el despliegue de servidor de HAPI FHIR](#Uso-de-Docker-para-el-despliegue-de-servidor-de-HAPI-FHIR)
    1. [Creación de la base de datos PostgreSQL(base de datos relacional) para almacenar al nuevo paciente ingresado](#Creación-de-la-base-de-datos-PostgreSQL(base-de-datos-relacional)-para-almacenar-al-nuevo-paciente-ingresado)


# <h1 align=center> **Captura del documento** </h1>
Proceso inicial donde el personal de salud tomará captura del documento de identidad del paciente

<p align=center><img src=https://www.elplural.com/uploads/s1/12/75/04/6/imagen-de-un-dni-cuerpo-nacional-de-policia.jpeg width="45%"></p>


# <h1 align=center> **Almacenamiento del documento capturado en Microsoft Azure Blob Storage** </h1>

Entonces la imagen capturada será procesada y almacenada en formato **jpg** en el almacenamiento de Azure: **Azure Blob Storage**

<p align="center">
  <img src="images/almacenamiento.png" alt="almacena"  width="80%"/>
</p>


# <h1 align=center> **Procesamiento de la imagen almacenada mediante Inteligencia Artificial** </h1>

Una vez la imagen almacenada en Azure se procede a la lectura de la misma mediante herramientas de inteligencia artificial (EASY OCR, Pillow, Tesseract) y logicas de difusión (The Fuzz)

##  **Pre procesamiento de la imagen mediante OCR para el selector de nacionalidad y posterior extracción de datos** 

<p align="center">
  <img src="images/lectura.png" alt="lee"  width="60%"/>
</p>
Este proceso es común para todas las nacionalidades, donde definimos un umbral de similitud, y una lista de palabras para cada nacionalidad. Es entonces que ahora se tiene la data limpia, los datos que queremos extraer, así vamos extrayendo la data bajo el siguiente formato:

data = {

'Nombres': [Nombres],
'Apellidos': [Apellidos],
'Numeros de DNI': [Numeros_de_DNI],
'Fecha de Nacimiento': [Fecha_de_Nacimiento], 
'Fecha de Emision': [Fecha_de_Emision],
'Fecha de Vencimiento': [Fecha_de_Vencimiento],
'Numero de Documento': [Numeros_de_DNI],
'Sexo': [Sexo],
'Pais': ['pais']}

##  **Transformacion de la captura de datos a formato HL7 FHIR** 

Las librerías mencionadas permitirán crear al **"paciente"** de la siguiente manera:

# <h1 align=center> **Almacenamiento de la data transformada en Microsoft Azure Blob Storage** </h1>

Una vez que la data esta en tipo diccionario en formato HL7 FHIR se procede a almacenar automáticamente los datos en Azure Blob Storage

<p align="center">
  <img src="images/alm_blob.png" alt="almacena"  width="80%"/>
</p>


# <h1 align=center> **Creación y despliegue del Azure Function para la automatización de procesos** </h1>

En este paso procedemos a la creación de la Azure Function, la cual permitirá automatizar el proceso de lectura, definiendo un trigger(disparador) que detectará una imagen ingresada, la cual automaticamente pasará a ser definida por nuestra función de lectura. Para mayor detalle :

<div style="text-align: right; color: silver; font-size: 1.2em; font-weight: bold;">
  <a href="https://github.com/GoUpCloud/HL7-FHIR_creacion-de-pacientes/tree/Azure-Function" style="color: silver; text-decoration: none;">
    Informacion de Azure Function
  </a>
</div>

# <h1 align=center> **Uso de Docker para el despliegue de servidor de HAPI FHIR** </h1>

Para la complementar la elaboracion de este proyecto se uso el despliegue de un servidor FHIR usando HAPI FHIR JPA.
el cual es una implementación completa del estándar HL7 FHIR para la interoperabilidad sanitaria en Java.

