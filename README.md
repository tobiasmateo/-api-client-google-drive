# api-client-googledrive

Cliente worker para consumir la API de Google Drive y guardar metadatos en una base de datos SQL.

---

# Objetivo

El objetivo de este proyecto es construir una aplicación worker capaz de:

- consumir una API externa,
- procesar respuestas en formato JSON,
- almacenar metadatos de forma persistente en una base de datos SQL.

El proveedor asignado para esta implementación es Google Drive.

El proyecto busca emular un sistema ligero de persistencia tipo "second brain" (segundo cerebro), capaz de agregar fuentes de datos externas y heterogéneas bajo un modelo de almacenamiento común.

---

# Alcance

**Incluye:**

- Leer los metadatos de un archivo de Drive a partir de su ID o de su URL.
- Guardar el recurso y su historial de cambios en SQLite.
- Trabajar con archivos compartidos públicamente ("cualquier persona con el enlace"), usando una API key.

**No incluye:**

- Descargar ni leer el contenido de los archivos.
- Crear, modificar o borrar archivos.
- Acceder a archivos privados (haría falta OAuth 2.0).
- Listar carpetas ni sincronizarlas.

---

# Contrato de la API

### Proveedor

API de Google Drive (v3).

### Endpoint

```
https://www.googleapis.com/drive/v3/files/{fileId}?fields=id,name,mimeType,size,createdTime,modifiedTime,webViewLink,md5Checksum&key={API_KEY}
```

Donde {fileId} es el identificador único del archivo de Drive. Si la entrada es una URL, como `https://drive.google.com/file/d/{fileId}/view` o `https://docs.google.com/document/d/{fileId}/edit`, el ID es el segmento que viene después de `/d/`.

La API key se lee de una variable de entorno y no se incluye en el repositorio.

### Respuesta Esperada de la API

La API devuelve los metadatos del archivo solicitado como un objeto JSON. El parámetro `fields` define qué campos se incluyen.
Ejemplo de respuesta:

```
{
  "id": "10sPcOoL-uCYmFVOKJssAtfF0tqEgwjAcyhWo7BG-8Y0",
  "name": "Comandos CPT",
  "mimeType": "application/vnd.google-apps.document",
  "webViewLink": "https://docs.google.com/document/d/10sPcOoL-uCYmFVOKJssAtfF0tqEgwjAcyhWo7BG-8Y0/edit?usp=drivesdk",
  "createdTime": "2025-12-13T20:40:30.398Z",
  "modifiedTime": "2026-10-30T19:06:42.778Z",
  "size": "252660"
}
```

Algunos campos no vienen en todos los archivos. Por ejemplo, `md5Checksum` no aparece en los archivos nativos de Google, como el de este ejemplo.

### Autenticación y Seguridad

**Mecanismo:** API key de Google Cloud, que se manda en el parámetro `key` de la URL.

- **Dónde se guarda la clave:** en la variable de entorno `GDRIVE_API_KEY`. No va en el código ni en el repositorio.
- **Restricción de la clave:** en Google Cloud Console la clave está limitada a la API de Google Drive, así no sirve para otros servicios si alguien la consigue.
- **Qué puede leer:** solo metadatos de archivos compartidos como "cualquier persona con el enlace". Si el archivo no es público, la API devuelve error (en las pruebas fue 404) y el worker no guarda nada.
- **Conexión:** siempre por HTTPS, sin desactivar la verificación del certificado en `libcurl`.
- **Logs:** como la clave va en la URL, el worker no imprime la URL completa de la solicitud.
- **Contenido de los archivos:** no se descarga, solo se piden los metadatos.

---

### Modelo de Persistencia

El worker almacena los datos utilizando el siguiente esquema SQL:

```
CREATE TABLE IF NOT EXISTS assets (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    asset_uri TEXT UNIQUE NOT NULL,
    title TEXT,
    entity TEXT NOT NULL,
    provider TEXT NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    meta_payload TEXT
);

CREATE TABLE IF NOT EXISTS assets_history (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    asset_id INTEGER NOT NULL,
    changed_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    history_payload TEXT NOT NULL,
    FOREIGN KEY (asset_id) REFERENCES assets (id) ON DELETE CASCADE
);
```

### Campos Almacenados

| Campo             | Descripción                                    |
| ----------------- | ---------------------------------------------- |
| **asset_uri**     | URL canónica única del archivo (`webViewLink`) |
| **title**         | Nombre del archivo (`name`)                    |
| **entity**        | Tipo de entidad lógica (`file`)                |
| **provider**      | Proveedor de datos (`googledrive`)             |
| **meta_payload**  | Payload JSON crudo devuelto por la API         |

---

### Historial de Snapshots

El proyecto también soporta la persistencia histórica de los metadatos de los assets. Cuando un asset existente se actualiza, el estado anterior se puede almacenar en la tabla `assets_history` como un snapshot en formato JSON.

Esto permite:

- Un seguimiento básico de cambios.
- La preservación del historial de metadatos.
- La auditoría de la evolución de los datos de la API a lo largo del tiempo.

> **Nota:** La fuente externa original nunca se modifica.

---

### Arquitectura Planificada

El flujo de trabajo esperado de la aplicación es:

1. Recibir el ID de un archivo o una URL completa como entrada a través de la CLI.
2. Parsear la entrada para extraer el ID único.
3. Realizar una solicitud HTTPS segura a la API de Google Drive utilizando `libcurl`.
4. Recibir la respuesta JSON. Si el código HTTP no es 200, mostrar el error y terminar sin guardar nada.
5. Extraer los campos de metadatos relevantes (`webViewLink`, `name`, etc.).
6. Persistir el recurso en la base de datos SQLite.
7. Almacenar el payload JSON completo dentro de `meta_payload` para futuros procesamientos polimórficos.
8. Opcionalmente, preservar las versiones anteriores en la tabla `assets_history` si el asset ya existía.

---

### Tecnologías

- **Lenguaje:** C (Estándar GNU11)
- **Librería de cliente HTTP:** `libcurl`
- **Parser de JSON:** `cJSON`
- **Base de datos:** SQLite (`libsqlite3`)

---

### Estado Actual

Estado actual del repositorio:

- Repositorio inicializado.
- Alcance y contrato de la API definidos.
- Esquema de la base de datos definido.
- Modelo de persistencia documentado.
- Arquitectura en proceso de desarrollo.

*La implementación se encuentra actualmente en curso.*

---

### Notas

Este proyecto está destinado a fines educativos para la materia Redes de las Computadoras II. El worker solo consume metadatos de archivos públicos y no modifica el contenido de Google Drive.
