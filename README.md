<h1 align=center> HL7-FHIR_creacion-de-pacientes </h1>
<p align=center><img src=https://www.goup.cloud/wp-content/uploads/2020/07/Go_Up_Cloud-300x210.png></p>

## Juan Esteban García.
## Email: juanestebangarciarodriguez@gmail.com
## Usuario GitHub: juesga1987
## Link video API : https://drive.google.com/file/d/15GJj8pvzrTOCVSnCr76groU5xm5__3-U/view?usp=share_link
## Dado el nivel de confidencialidad del proyecto no es posible compartir el código que permitió dicho desarrollo

El presente proyecto fue desarrollado para la empresa de consultoría GO UP CLOUD y tiene como objetivo la captura de un documento identidad para ser procesado mediante técnicas de **Inteligencia Artificial** con el fin de obtener la lectura de sus datos. Dicha información generada es almacenada en un **Repositorio Clínico** llamado **FHIR** a través de **Azure Health Data Services**. Esta información resultante será de consulta inmediata desde cualquier entidad conectada a FHIR a través de Azure Health Data Services.

El siguiente diagrama muestra el camino desde la toma de imagen hasta el almacenamiento o consulta del paciente:


<p align="center">
  <img src="arquitectura Goupcloud.png" alt="arquitect"  width="80%"/></p>

# <h1 align=center> **Captura del documento** </h1>
Proceso inicial donde el personal de salud tomará captura del documento de identidad del paciente

<p align=center><img src=https://www.elplural.com/uploads/s1/12/75/04/6/imagen-de-un-dni-cuerpo-nacional-de-policia.jpeg width="45%"></p>


# <h1 align=center> **Almacenamiento del documento capturado en Microsoft Azure Blob Storage** </h1>

Entonces la imagen capturada será procesada y almacenada en formato **jpg** en el almacenamiento de Azure: **Azure Blob Storage**

<p align="center">
  <img src="almacenamiento.png" alt="almacena"  width="80%"/>
</p>


# <h1 align=center> **Procesamiento de la imagen almacenada mediante Inteligencia Artificial** </h1>

Una vez la imagen almacenada en Azure se procede a la lectura de la misma mediante herramientas de inteligencia artificial (EASY OCR, Pillow, Tesseract) y logicas de difusión (The Fuzz)

##  **Pre procesamiento de la imagen mediante OCR para el selector de nacionalidad y posterior extracción de datos** 
</p>
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
'Pais': ['pais']}
```

##  **Transformacion de la captura de datos a formato HL7 FHIR** 

Las librerías mencionadas permitirán crear al **"paciente"**.

# <h1 align=center> **Almacenamiento de la data transformada en Microsoft Azure Blob Storage** </h1>

Una vez que la data esta en tipo diccionario en formato HL7 FHIR se procede a almacenar automáticamente los datos en Azure Blob Storage

# <h1 align=center> **Creación y despliegue del Azure Function para la automatización de procesos** </h1>

En este paso procedemos a la creación de la Azure Function, la cual permitirá automatizar el proceso de lectura, definiendo un trigger(disparador) que detectará una imagen ingresada, la cual automaticamente pasará a ser definida por nuestra función de lectura.

# <h1 align=center> **Uso de Docker para el despliegue de servidor de HAPI FHIR** </h1>

Para la complementar la elaboracion de este proyecto se uso el despliegue de un servidor FHIR usando HAPI FHIR JPA.
el cual es una implementación completa del estándar HL7 FHIR para la interoperabilidad sanitaria en Java.

