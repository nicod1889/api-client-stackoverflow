# api-client-stackoverflow

Cliente worker de línea de comandos, escrito en C, que consume la API pública de StackOverflow y persiste los metadatos de las preguntas en una base de datos SQLite.

Proyecto educativo para la materia **Redes de las Computadoras II**. Actúa únicamente como consumidor de información pública y no modifica el contenido de StackOverflow.

> El compromiso formal de lo que hace (y no hace) la aplicación está en [`contrato-proyecto.md`](contrato-proyecto.md). Los hitos y fechas están en [`workplan.md`](workplan.md).

---

## Objetivo

Construir un worker capaz de:

- consumir una API pública externa por HTTPS,
- procesar y validar respuestas en formato JSON,
- almacenar metadatos de forma persistente en una base SQL,
- conservar el historial de cambios de cada recurso.

El proveedor asignado es StackOverflow (Stack Exchange API v2.3). El proyecto emula un sistema liviano de persistencia tipo *"second brain"*, capaz de agregar fuentes externas y heterogéneas bajo un modelo de almacenamiento común (`assets`).

---

## Dependencias

Además de la biblioteca estándar de C (glibc), el proyecto usa:

| Librería | Uso | Paquete (Debian/Ubuntu) |
|---|---|---|
| libcurl | Cliente HTTPS | `libcurl4-openssl-dev` |
| cJSON | Parseo de JSON | `libcjson-dev` |
| SQLite3 | Base de datos | `libsqlite3-dev` |

## Requisitos previos

- Linux (desarrollado y probado en Ubuntu/Debian).
- Compilador `gcc` con soporte GNU11 y `make`.
- Las tres librerías de la tabla anterior.
- Conexión a internet para consultas reales (los tests no la necesitan).
- Opcional: `sqlite3` (cliente de consola) para inspeccionar la base.

```bash
sudo apt install build-essential libcurl4-openssl-dev libcjson-dev libsqlite3-dev sqlite3
```

## Compilación y uso

```bash
make          # compila y genera ./so-worker
make test     # ejecuta los tests
make clean    # borra binarios y objetos
```

```bash
./so-worker 41107297
./so-worker https://stackoverflow.com/questions/41107297/titulo-de-la-pregunta
./so-worker --db mi_base.db -v 41107297
./so-worker --from-file tests/fixtures/ok.json 41107297
```

### Comandos y opciones

| Opción | Descripción |
|---|---|
| `<id>` / `<url>` | ID numérico o URL completa de la pregunta |
| `--db <ruta>` | Base SQLite (por defecto `stackoverflow.db`) |
| `--from-file <ruta>` | Usa un JSON local en lugar de la red (tests offline) |
| `-v`, `--verbose` | Salida detallada |
| `-h`, `--help` | Ayuda |

### Códigos de salida

| Código | Significado |
|---|---|
| 0 | Éxito (creado, actualizado o sin cambios) |
| 1 | Uso incorrecto o entrada inválida |
| 2 | Error de red o HTTP distinto de 200 |
| 3 | Respuesta inválida (JSON mal formado o schema no cumplido) |
| 4 | Pregunta no encontrada |
| 5 | Error de base de datos |

---

## Contrato de la API

**Endpoint:**

```
https://api.stackexchange.com/2.3/questions/{id}?site=stackoverflow
```

`{id}` es el identificador numérico de la pregunta. La API responde siempre comprimida con gzip, por lo que el cliente debe habilitar la descompresión. El acceso anónimo tiene un límite de cuota diario (el campo `quota_max` de la respuesta indica 300).

**Respuesta esperada (resumida):**

```json
{
  "items": [
    {
      "tags": ["html", "artificial-intelligence", "dashboard"],
      "owner": { "user_id": 15249147, "display_name": "joy.kumagai" },
      "is_answered": false,
      "view_count": 4,
      "answer_count": 0,
      "score": 0,
      "last_activity_date": 1781568903,
      "creation_date": 1781568903,
      "question_id": 79959398,
      "link": "https://stackoverflow.com/questions/79959398/...",
      "title": "Which program should I use to access Anthropic ..."
    }
  ],
  "has_more": false,
  "quota_max": 300,
  "quota_remaining": 299
}
```

### Schema validado

Se valida el objeto `items[0]`:

- **Obligatorios:** `question_id` (entero), `title` (cadena), `link` (cadena).
- **Opcionales conocidos:** `tags`, `owner`, `is_answered`, `view_count`, `answer_count`, `score`, `creation_date`, `last_activity_date`, `content_license`. Si están, deben tener el tipo correcto.
- **Campos desconocidos:** se conservan y se avisa en modo verbose.

Si el formato de la API cambia y el schema no se cumple, el worker termina con código 3, informa el error y **no escribe nada** en la base. El detalle completo está en la sección 3.3 del contrato.

---

## Modelo de persistencia

```sql
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

| Campo | Valor |
|---|---|
| `asset_uri` | `https://stackoverflow.com/questions/{question_id}` |
| `title` | Título de la pregunta |
| `entity` | `question` |
| `provider` | `stackoverflow` |
| `meta_payload` | Objeto `items[0]` en JSON, tal como lo devolvió la API |
| `history_payload` | `meta_payload` anterior a un cambio (snapshot) |

> Nota: SQLite exige `PRAGMA foreign_keys = ON;` en cada conexión para que se respete la clave foránea; el worker lo activa al abrir la base.

---

## Flujo de la aplicación

```
CLI (<id | url>)
      |
      v
Parsear entrada -> ID numérico        (error: código 1)
      |
      v
HTTPS GET a la API (libcurl)          (error: código 2)
      |
      v
Parsear JSON y validar schema         (error: código 3 / 4)
      |
      v
¿Existe el asset_uri en la base?
      |
      +-- No ------------> INSERT en assets
      |
      +-- Sí, sin cambios -> no hace nada
      |
      +-- Sí, con cambios -> snapshot en assets_history + UPDATE assets
      |                      (una sola transacción)    (error: código 5)
      v
Informar resultado y terminar
```

---

## Tests

- **Tests de integración con script de shell** (`tests/run_tests.sh`): ejecutan `so-worker --from-file` con respuestas JSON de ejemplo (`tests/fixtures/`) y verifican el código de salida y el contenido de la base.
- **Tests unitarios en C** del parseo de entrada (ID y URL) y de la validación del schema.
- No requieren internet: la red se reemplaza por archivos locales.

Casos cubiertos: creación, actualización con historial, consulta sin cambios, ID inválido, URL inválida, `items` vacío, campo obligatorio faltante, tipo incorrecto, JSON mal formado, campo desconocido y respuesta HTTP con error (simulada).

---

## Estructura del proyecto

```
api-client-stackoverflow/
├── README.md
├── contrato-proyecto.md
├── workplan.md
├── Makefile
├── src/
│   ├── main.c          # CLI y flujo general
│   ├── input.c         # parseo de ID / URL
│   ├── http.c          # cliente libcurl
│   ├── schema.c        # validación del JSON (cJSON)
│   └── db.c            # SQLite: assets y assets_history
├── include/            # headers (.h)
├── db/
│   └── schema.sql      # definición de tablas
├── tests/
│   ├── run_tests.sh
│   ├── unit/
│   └── fixtures/       # respuestas JSON de ejemplo
└── Documentacion/
    ├── 01-BDD.md
    ├── 02-API_SETUP.md
    ├── 03-ImplementacionFinal.md
    ├── 04-Evidencias.md
    └── adr/            # decisiones de arquitectura (opcional)
```

## Documentación

- [Contrato del proyecto](contrato-proyecto.md)
- [Workplan](workplan.md)
- [Base de datos (análisis)](Documentacion/01-BDD.md)
- [Configuración de la API](Documentacion/02-API_SETUP.md)
- [Implementación final](Documentacion/03-ImplementacionFinal.md)
- [Evidencias de implementación](Documentacion/04-Evidencias.md)

---

## Estado actual

- [x] Repositorio inicializado
- [x] Contrato de la API y modelo de persistencia definidos
- [x] Contrato formal del proyecto
- [ ] Workplan
- [ ] Implementación
- [ ] Tests
- [ ] Documentación final y evidencias
