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


Veamos cada uno de los procesos:

# <h1 align=center> **Captura del documento** </h1>
Proceso inicial donde el personal de salud tomará captura del documento de identidad del paciente

<p align=center><img src=https://www.elplural.com/uploads/s1/12/75/04/6/imagen-de-un-dni-cuerpo-nacional-de-policia.jpeg width="45%"></p>


# <h1 align=center> **Almacenamiento del documento capturado en Microsoft Azure Blob Storage** </h1>

Entonces la imagen capturada será procesada y almacenada en formato **jpg** en el almacenamiento de Azure: **Azure Blob Storage**

<p align="center">
  <img src="images/almacenamiento.png" alt="almacena"  width="80%"/>
</p>


# <h1 align=center> **Procesamiento de la imagen almacenada mediante Inteligencia Artificial** </h1>

Una vez la imagen almacenada en Azure se procede a la lectura de la misma mediante las siguientes librerias de python: 
```
import easyocr
import re
import numpy as np
import pandas as pd
import cv2
from PIL import Image
```

Estas librerias seran recursos vitales para el manejo de las imagenes a traves del pre procesamiento, en caso la imagen lo requiera, así también para la lectura de texto será útil el uso de **easyocr**

##  **Pre procesamiento de la imagen mediante OCR para el selector de nacionalidad y posterior extracción de datos** 

El Documento de identidad argentino y chileno presentan una marca de agua, la cual dificulta una correcta lectura de los datos, es por ello que en este parte del proceso se procede a realizar un tratamiento de imagen en los documentos de las nacionalidades mencionadas. Se define una funcion, por ejemplo, para cada nacionalidad en la que el tratamiento será; por ejemplo para el caso de argentina: 
```
def procesa_imagen(image_path):
        # Abre la imagen

        image = Image.open(image_path)

        # Convierte la imagen a escala de grises
        gray_image = image.convert('L')

        # Crea una máscara de las partes con intensidad menor a 45
        threshold = 45
        black_mask = gray_image.point(lambda x: 0 if x <= threshold else 255, '1')

        # Aplicamos la máscara a la imagen en escala de grises
        masked_image = Image.new('L', gray_image.size)
        masked_image.paste(gray_image, mask=black_mask)

        # Guardamos la imagen resultante llamado "escala_grises" en un array
        escala_gris = np.array(masked_image)

        # Cerramos la imagen original
        image.close()

        # Aplicar filtros y mejoras de imagen
        # Por ejemplo, eliminación de ruido, ajuste de contraste, etc.
        denoised_image = cv2.fastNlMeansDenoising(escala_gris, None,3, 7, 21)
        return denoised_image
```
Donde el resultado obtenido es una imagen en escala de grises y posterior eliminación de ruido donde se resalta el texto a leer. Posterior a este proceso le suecede el proceso de lectura mediante **OCR** y extracción de datos. A modo ejemplo seguiremos con el caso del documento argentino, y es consecuente a la lectura que se procede a realizar una lista de palabras de cada documento; dicha lista contendrá los datos que queremos que sean eliminados de nuestra lectura, la cual nos detecta: 

<p align="center">
  <img src="images/lectura.png" alt="lee"  width="60%"/>
</p>

Entonces observamos que las palabras : REPUBLICA ARGENTINA, MERCOSUR, DOCUMENTO NACIONAL,... etc. Son palabras recurrentes que no necesitamos en la lectura, es entonces que usamos la **lógica difusa** meidante la libreria fuzzy(thefuzz), la cual nos permite realizar un matcheo(una similitud) entre las palabras que no queremos, con las palabras detectas por el código; asi tenemos:

```
palabras_argentina = ["REPUBLICA ARGENTINA","MERCOSUR","REGISTRO NACIONAL DE LAS PERSONAS","MINISTERIO DEL INTERIOR Y TRANSPORTE","MINISTERIO DEL INTERIOR, OBRAS PUBLICAS Y VIVIENDA","VIVIENDA","Apellido",
                      "Surname","Nombre","Fecha","Name","Sexo","Sex","Nacionalidad","Nationality","Ejempla","Fecha de nacimiento","Date of birth","Fecha de emision","Date of issue","Firma identificado",
                      "Signature","Fecha de vencimiento","Date of expiry","Documento","Document","Tramite","Of ident"]

# Usamos fuzzy wuzzy y mejoramos la similitud de palabras y asi poder encontrar las palabras que no queremos y reducirlas a minuscula
    def buscar_palabras(palabras_similares, palabras):
        resultados = []
        for similares in palabras_similares:
            encontrada = False
            for palabra in palabras:
                similitud = fuzz.ratio(similares, palabra)
                if similitud >= 50:  # Umbral de similitud
                    resultados.append(((palabra.lower())))
                    encontrada = True
                    break

            if not encontrada:
                resultados.append((similares))

        return resultados
```

Este proceso es común para todas las nacionalidades, donde definimos un umbral de similitud, y una lista de palabras para cada nacionalidad. Es entonces que ahora se tiene la data limpia, los datos que queremos extraer, así vamos extrayendo la data bajo el siguiente formato:
```
data = {

'Nombres': [Nombres],
'Apellidos': [Apellidos],
'Numeros de DNI': [Numeros_de_DNI],
'Fecha de Nacimiento': [Fecha_de_Nacimiento], 
'Fecha de Emision': [Fecha_de_Emision],
'Fecha de Vencimiento': [Fecha_de_Vencimiento],
'Numero de Documento': [Numeros_de_DNI],
'Sexo': [Sexo],
'Pais': ['Argentina']}

```
Es entonces que nuestra funcion principal para cada nacionalidad nos retornara una data en el formato ya mostrado, un formato **JSON**. Pero..

### **Cómo paso mi data de JSON A formato FHIR?**

Es aqui que hacemos uso de las librerias de fhir en python:

```
from fhir.resources.patient import Patient
from fhir.resources.identifier import Identifier
from fhir.resources.contactpoint import ContactPoint
from fhir.resources.address import Address
from fhir.resources.codeableconcept import CodeableConcept
from fhir.resources.coding import Coding
from fhir.resources.humanname import HumanName
from fhir.resources.extension import Extension

```

##  **Transformacion de la captura de datos a formato HL7 FHIR** 


Las librerías mencionadas permitirán crear al **"paciente"** de la siguiente manera:

```
# Resource = Paciente
    patient = Patient()

    # DNI
    identifier = Identifier()
    identifier.system = "http://example.com/patient-identifier"
    identifier.value = Numeros_de_DNI
    patient.identifier = [identifier]

    # Nombre y apellidos
    name = HumanName()
    name.family = Apellidos
    name.given = [str(Nombres)]
    patient.name = [name]

    # Sexo
    patient.gender = Sexo

    # Nacimiento
    #p = construct_fhir_element('Patient', {'birthDate': str(fecha_nacimiento)})
    patient.birthDate = str(Fecha_de_Nacimiento)

    # Ubicacion
    address = Address()
    address.use = "home"
    address.type = "both"
    address.text = "nan"
    address.country = str('Argentina')  # Aqui es condicional pues depende del pais
    patient.address = [address]

    # Contacto
    contact = ContactPoint()
    contact.system = "email"
    contact.value = "nan"
    patient.telecom = [contact]

    # Idioma
    extension = Extension()
    extension.url = "http://hl7.org/fhir/StructureDefinition/patient-preferredLanguage"
    extension.valueCodeableConcept = CodeableConcept()
    coding = Coding()
    coding.system = "http://hl7.org/fhir/ValueSet/languages"
    coding.code = "es"
    coding.display = "Spanish"
    extension.valueCodeableConcept.coding = [coding]
    extension.valueCodeableConcept.text = "Spanish"
    patient.extension = [extension]

    # Convertir a JSON
    patient_json = patient.json()


```

Es imperativo recordar que esta transfomación es posible debido a que previamente se tiene la data en formato **JSON** y también las fechas están en formato americano 

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


##  **Arquitectura Azure Function** 

###  **Creación almacenamiento y contenedores en Azure** 

1.  Ir a portal.azure.com y acceder con usuario
2.  En el panel principal dirigirse a la opción CREAR UN RECURSO.
3.  Buscar en categorías la opción ALMACENAMIENTO.
4.  En la opción CUENTA DE ALMACENAMIENTO crear un recurso.
5.  Despues de creado acceder al mismo, en el panel izquierdo en la opción ALMACENAMIENTO DE DATOS abrir la opción CONTENEDORES.
6.  Para crear el contenedor de llegada de la imagen y subida de archivo con información ya procesada, buscar la opción + CONTENEDOR.
7.  Dado que la logica de la API implica la llegada de una imagen y la subida de la información ya procesada se deben crear dos contenedores dentro la misma cuenta de almacenamiento.

### < **Azure function en ambiente local con Visual Studio Code** 

1. Descargar las herramientas Azure necesarias para la creación de la función:
   
- Azure function core tools: https://azurelessons.com/azure-functions-core-tools/ --> Guia de instalacion y links de descarga
- Extension Azure Visual Studio Code: En la opcion extensiones buscar "Azure functions" del editor Microsoft
- Extension Azure Visual Studio Code: En la opcion extensiones buscar "Azure App Service" del editor Microsoft
- Extension Azure Visual Studio Code: En la opcion extensiones buscar "Azure Storage" del editor Microsoft
- Extension Azure Visual Studio Code: En la opcion extensiones buscar "Azure Resources" del editor Microsoft
  
2. Creacion Azure function local:
   
- En el siguiente link se explica por parte de microsoft la creacion en VS Code para varios lenguajes: https://learn.microsoft.com/es-es/azure/azure-functions/functions-develop-vs-code?tabs=node-v3%2Cpython-v2%2Cisolated-process&pivots=programming-language-python
- Link de video explicativo equipo pasantia: https://drive.google.com/file/d/11FLu_AJ_K8LRq-C0jXLFsEZee4qlA_63/view?usp=sharing
  
3. Deploy en ecosistema AZURE:
- Links a videos explicativos:
  https://drive.google.com/file/d/1Ex9PxDjJGQVukX8yNBj1CbIFEfXYF5Qw/view?usp=sharing
  https://drive.google.com/file/d/1pmm7JR_PbOd4aLNxA2DVwUo8b0i5bXEU/view?usp=sharing
  https://drive.google.com/file/d/1X4inetn1debOa1ZP1RbjFY2inFS_ZzOS/view?usp=sharing



# <h1 align=center> **Uso de Docker para el despliegue de servidor de HAPI FHIR** </h1>


Para la complementar la elaboracion de este proyecto se uso el despliegue de un servidor FHIR usando HAPI FHIR JPA.
el cual es una implementación completa del estándar HL7 FHIR para la interoperabilidad sanitaria en Java.


### Pre requisitos

Para poder desarrollar este ambiente se utilizo lo siguiente:

- Se reviso el siguiente  **[Repositorio](https://github.com/hapifhir/hapi-fhir-jpaserver-starter)** de hapifhir.
- Se hizo un fork del repositorio de GitHub para poder modificarlo y poder personalizar el proyecto y guardar los resultados en GitHub. 

- Se uso la imagen de hapi-fhir-jpaserver el cual fue extraido de [hapiproject/hapi Docker Hub](https://hub.docker.com/r/hapiproject/hapi)

### Despliegue de la imagen de  [Docker Hub](https://hub.docker.com/r/hapiproject/hapi)

Para obtener la version mas actualizada de`hapi-fhir-jpaserver` esta es una imagen que se encuentra publicado en Docker Hub . Para ejecutar la imagen de Docker publicada desde DockerHub:

``` Terminal
docker pull hapiproject/hapi:latest
docker run -p 8080:8080 hapiproject/hapi:latest
```
Explicacion de codigo:
- El primer comando traeran la imagen de Docker publicada en DockerHub.
- El segundo comando ejecutará la imagen el contenedor asignando el puerto 8080 del contenedor al puerto 8080 en el host. Una vez en ejecución, puede acceder a `http://localhost:8080/` en el navegador para acceder a la interfaz de usuario del servidor HAPI FHIR o usar `http://localhost:8080/fhir/` como URL base para sus solicitudes REST.

Observacion:
- Al modificar puerto asignado, debe cambiar la configuración utilizada por HAPI para tener la propiedad/valor `hapi.fhir.tester` correcto.

### Docker Compose

Este es el contenedor de docker que contiene dos servicios
#### 1) fhir
Este servicio de fhir es un servicio que provee las funcionalidades para poder manipular arhivos HL7 FHIR, como la creacion y actualizacion de, en nuestro caso, pacientes medicos.

Contiene una imagen hapi, localizada en el puerto 8080 (donde posteriormente se haran las consultas), un archivo de configuracion, y una dependencia de una base de datos.

#### 2) db
La base de datos de la que depende hapi, es una base de datos PostgreSQL, en entorno de ejecucion por defecto, en un puerto variable y con un archivo de configuracion.


```yml 
version: '3.7'

services:
  fhir:
    container_name: fhir
    image: "hapiproject/hapi"
    ports:
      - "8080:8080"
    configs:
      - source: hapi
        target: /app/config/application.yaml
      - source: hapi-extra-classes
        target: /app/extra-classes
    depends_on:
      - db

  db:
    image: postgres
    environment:
      POSTGRES_PASSWORD: admin
      POSTGRES_USER: admin
      POSTGRES_DB: hapi
    ports:
      - "5000:5432"
    volumes:
      - ./hapi.postgress.data:/var/lib/postgresql/data

configs:
  hapi:
     file: ./hapi.application.yaml
  hapi-extra-classes:
     file: ./hapi-extra-classes
```


Este archivo Docker-Compose define dos servicios: uno para un servidor FHIR y otro para una base de datos PostgreSQL. Además, establece configuraciones personalizadas para el servicio del servidor FHIR mediante archivos locales. El servicio del servidor FHIR depende del servicio de la base de datos PostgreSQL, y se realizan mapeos de puertos y asociaciones de volúmenes para configurar estos servicios.

### Docker compose-customization
Finalmente,para personalizar el docker compose y para que guarde en el contenedor que hemos desplegado
se realiza las siguientes modificaciones en el archivo .yaml:
```.yaml
spring:
  datasource:
    url: 'jdbc:postgresql://db:5432/hapi'
    username: admin
    password: admin
    driverClassName: org.postgresql.Driver
  jpa:
    properties:
      hibernate.dialect: ca.uhn.fhir.jpa.model.dialect.HapiFhirPostgres94Dialect
      hibernate.search.enabled: false
hapi:
  fhir:
    custom-bean-packages: the.package.containing.your.interceptor
    custom-interceptor-classes: the.package.containing.your.interceptor.YourInterceptor
```



###  **Creación de la base de datos PostgreSQL(base de datos relacional) para almacenar al nuevo paciente ingresado** 

