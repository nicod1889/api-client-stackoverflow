# api-client-stackoverflow

Cliente worker simple de API para consumir la API pública de StackOverflow y persistir metadatos en una base de datos SQL.

---

# Objetivo

El objetivo de este proyecto es construir una aplicación worker capaz de:

* consumir una API pública externa,
* procesar respuestas en formato JSON,
* almacenar metadatos de forma persistente en una base de datos SQL.

El proveedor asignado para esta implementación es StackOverflow.

El proyecto busca emular un sistema ligero de persistencia tipo "second brain" (segundo cerebro), capaz de agregar fuentes de datos externas y heterogéneas bajo un modelo de almacenamiento común.

---

# Contrato de la API

### Proveedor

API de StackOverflow (Stack Exchange API v2.3).

### Endpoint

```text
[https://api.stackexchange.com/2.3/questions/](https://api.stackexchange.com/2.3/questions/){id}?site=stackoverflow

```

Donde {id} representa el identificador numérico único de la pregunta de StackOverflow.
Ejemplo:

```text

[https://api.stackexchange.com/2.3/questions/41107297?site=stackoverflow](https://api.stackexchange.com/2.3/questions/41107297?site=stackoverflow)

```

Respuesta Esperada de la API
La API devuelve metadatos en formato JSON que describen la pregunta solicitada dentro de un array llamado items.
Ejemplo de respuesta:

```text

{
  "items": [
    {
      "tags": [
        "html",
        "artificial-intelligence",
        "dashboard"
      ],
      "owner": {
        "account_id": 20764881,
        "reputation": 11,
        "user_id": 15249147,
        "user_type": "registered",
        "profile_image": "https://www.gravatar.com/avatar/ddc1cdb397261ade9ced7434b5fd3b98?s=256&d=identicon&r=PG&f=y&so-version=2",
        "display_name": "joy.kumagai",
        "link": "https://stackoverflow.com/users/15249147/joy-kumagai"
      },
      "is_answered": false,
      "view_count": 4,
      "answer_count": 0,
      "score": 0,
      "last_activity_date": 1781568903,
      "creation_date": 1781568903,
      "question_id": 79959398,
      "content_license": "CC BY-SA 4.0",
      "link": "https://stackoverflow.com/questions/79959398/which-program-should-i-use-to-access-anthropic-with-an-api-key-that-does-not-set",
      "title": "Which program should I use to access Anthropic with an API key that does not set token limits?"
    }
  ],
  "has_more": false,
  "quota_max": 300,
  "quota_remaining": 299
}

```

Modelo de Persistencia
El worker almacena los datos utilizando el siguiente esquema SQL:

```text

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

| Campo | Descripción |
| :--- | :--- |
| **asset_uri** | URL canónica única de la pregunta |
| **title** | Título de la pregunta |
| **entity** | Tipo de entidad lógica (`question`) |
| **provider** | Proveedor de datos (`stackoverflow`) |
| **meta_payload** | Payload JSON crudo devuelto por la API |

---

### Historial de Snapshots

El proyecto también soporta la persistencia histórica de los metadatos de los assets. Cuando un asset existente se actualiza, el estado anterior se puede almacenar en la tabla `assets_history` como un snapshot en formato JSON.

Esto permite:
* Un seguimiento básico de cambios.
* La preservación del historial de metadatos.
* La auditoría de la evolución de los datos de la API a lo largo del tiempo.

> **Nota:** La fuente externa original nunca se modifica.

---

### Arquitectura Planificada

El flujo de trabajo esperado de la aplicación es:

1. Recibir el ID de una pregunta o una URL completa como entrada a través de la CLI.
2. Parsear la entrada para extraer el ID numérico único.
3. Realizar una solicitud HTTPS segura a la API de StackOverflow utilizando `libcurl`.
4. Recibir y descomprimir la respuesta JSON.
5. Extraer los campos de metadatos relevantes (`link`, `title`, etc.).
6. Persistir el recurso en la base de datos SQLite.
7. Almacenar el payload JSON completo dentro de `meta_payload` para futuros procesamientos polimórficos.
8. Opcionalmente, preservar las versiones anteriores en la tabla `assets_history` si el asset ya existía.

---

### Tecnologías

* **Lenguaje:** C (Estándar GNU11)
* **Librería de cliente HTTP:** `libcurl`
* **Parser de JSON:** `cJSON`
* **Base de datos:** SQLite (`libsqlite3`)

---

### Estado Actual

Estado actual del repositorio:
* Repositorio inicializado.
* Contrato de la API definido.
* Esquema de la base de datos definido.
* Modelo de persistencia documentado.
* Arquitectura en proceso de desarrollo.

*La implementación se encuentra actualmente en curso.*

---

### Notas

Este proyecto está destinado a fines educativos para la materia Redes de las Computadoras II. El worker actúa únicamente como un consumidor de información pública y no modifica el contenido de StackOverflow.