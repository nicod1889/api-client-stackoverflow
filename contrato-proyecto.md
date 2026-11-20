# Contrato del Proyecto: api-client-stackoverflow

**Materia:** Redes de las Computadoras II
**Versión:** 1.0
**Fecha:** 20/11/2026

---

## 1. Partes

- **Solicitante:** Alejandro Rodriguez, cátedra de Redes de las Computadoras II.
- **Responsable del desarrollo:** Nicolás Dinolfo (GitHub: `nicod1889`).

Ambas partes acuerdan el presente contrato para definir y ejecutar el proyecto técnico descrito a continuación.

## 2. Objeto del contrato

El Responsable se compromete a desarrollar un **worker ejecutable por línea de comandos, escrito en C (GNU11)**, que:

1. reciba el identificador (o la URL) de una pregunta de StackOverflow,
2. consulte la API pública de Stack Exchange v2.3 mediante HTTPS,
3. valide la respuesta JSON contra un schema mínimo definido en este contrato,
4. persista los metadatos en una base de datos SQLite,
5. conserve el historial de cambios de cada pregunta consultada.

El worker es un consumidor de información pública y **nunca modifica** el contenido de la fuente externa.

## 3. Alcance funcional

### 3.1 Interfaz de línea de comandos

```
so-worker [opciones] <id | url>
```

| Opción | Descripción |
|---|---|
| `<id>` | ID numérico de la pregunta. Ej: `41107297` |
| `<url>` | URL completa. Ej: `https://stackoverflow.com/questions/41107297/titulo-de-la-pregunta` |
| `--db <ruta>` | Ruta de la base SQLite (por defecto `stackoverflow.db`) |
| `--from-file <ruta>` | Lee la respuesta desde un archivo JSON local en lugar de la red (pensado para tests sin conexión) |
| `-v`, `--verbose` | Muestra información detallada del proceso |
| `-h`, `--help` | Muestra la ayuda |

### 3.2 Flujo de ejecución

1. Parsear la entrada y extraer el ID numérico. Si no es válida, terminar con error.
2. Solicitar `https://api.stackexchange.com/2.3/questions/{id}?site=stackoverflow` con `libcurl` (HTTPS, con descompresión gzip habilitada).
3. Verificar que el código HTTP sea 200.
4. Parsear el JSON con `cJSON` y verificar que `items` exista y tenga al menos un elemento.
5. Validar el elemento `items[0]` contra el schema de la sección 3.3.
6. Calcular `asset_uri = https://stackoverflow.com/questions/{question_id}`.
7. Persistir en SQLite:
   - Si el `asset_uri` **no existe**: insertar en `assets`.
   - Si **existe y el `meta_payload` cambió**: guardar el estado anterior en `assets_history` y actualizar `assets` (incluyendo `updated_at`), todo dentro de **una única transacción**.
   - Si **existe y no cambió**: no modificar nada.
8. Informar el resultado por salida estándar y terminar con el código de salida correspondiente.

### 3.3 Schema esperado de la respuesta

Se valida el objeto `items[0]`.

**Campos obligatorios** (si falta alguno o tiene otro tipo, la respuesta es inválida):

| Campo | Tipo |
|---|---|
| `question_id` | entero |
| `title` | cadena |
| `link` | cadena |

**Campos opcionales conocidos** (si están, deben tener el tipo indicado; si no están, se ignoran sin error):

| Campo | Tipo |
|---|---|
| `tags` | arreglo de cadenas |
| `owner` | objeto |
| `is_answered` | booleano |
| `view_count`, `answer_count`, `score` | entero |
| `creation_date`, `last_activity_date` | entero (epoch) |
| `content_license` | cadena |

**Campos desconocidos:** se conservan en el payload y se informa un aviso en modo `--verbose`. Esto permite tolerar que la API agregue campos nuevos sin dejar de funcionar.

### 3.4 Modelo de persistencia

Se utilizan las tablas `assets` y `assets_history` definidas en el README. Los valores almacenados son:

| Campo | Valor |
|---|---|
| `asset_uri` | `https://stackoverflow.com/questions/{question_id}` |
| `title` | Campo `title` de la respuesta |
| `entity` | `question` |
| `provider` | `stackoverflow` |
| `meta_payload` | Objeto `items[0]` en JSON, tal como lo devolvió la API |
| `history_payload` | `meta_payload` anterior al cambio |

Se guarda `items[0]` (y no la respuesta completa) para que campos que cambian siempre, como `quota_remaining`, no generen historial falso.

### 3.5 Manejo de errores

El programa **no debe terminar de forma abrupta** ante respuestas inesperadas. Ante cualquier error informa un mensaje claro por la salida de error estándar, **no escribe datos parciales** en la base y devuelve un código de salida específico:

| Código | Significado |
|---|---|
| 0 | Éxito (asset creado, actualizado o sin cambios) |
| 1 | Uso incorrecto o entrada inválida |
| 2 | Error de red o HTTP distinto de 200 (incluye límite de cuota) |
| 3 | Respuesta inválida: JSON mal formado o schema no cumplido |
| 4 | Pregunta no encontrada (`items` vacío) |
| 5 | Error de base de datos |

## 4. Entregables

1. Código fuente en C, con `Makefile`, dentro del repositorio.
2. `README.md` con dependencias, requisitos, compilación, uso, flujo, tests y estructura.
3. Este contrato (`contrato-proyecto.md`).
4. `workplan.md` con hitos, fechas y diagrama PERT.
5. Suite de tests automatizados con datos de ejemplo.
6. Carpeta `Documentacion/` con: análisis de la base de datos, configuración de la API, implementación final y evidencias (salidas y resultados de tests).
7. Registros de decisiones de arquitectura (ADR), de forma opcional.

## 5. Criterios de aceptación

El contrato se considera cumplido cuando:

1. El proyecto compila sin errores con `make` en un entorno con las dependencias indicadas.
2. Dado un ID o una URL válidos, el worker consulta la API y guarda el registro en `assets`.
3. Al consultar de nuevo la misma pregunta con datos modificados, el estado anterior queda en `assets_history` y `assets` se actualiza.
4. Al consultar de nuevo sin cambios, la base no se modifica.
5. Ante entrada inválida, ID inexistente, error de red o JSON que no cumple el schema, el worker termina con el código de salida correcto y sin dañar la base.
6. Los tests automatizados pasan y cubren los casos de los puntos 2 a 5.
7. La documentación comprometida en la sección 4 está presente en el repositorio.

## 6. Exclusiones

No se incluye en este contrato, salvo acuerdo adicional:

- Interfaz gráfica o web.
- Autenticación OAuth o uso de API key (se utiliza el acceso anónimo de la API).
- Consulta de otras entidades (respuestas, comentarios, usuarios) o de listados y búsquedas de preguntas.
- Ejecución periódica o como servicio (daemon, cron).
- Reintentos automáticos o manejo avanzado de cuotas.
- Modificación de contenido en StackOverflow.
- Despliegue productivo y soporte posterior al período académico.

---

*Firmas / conformidad:*

Solicitante: ______________________  Responsable: ______________________
