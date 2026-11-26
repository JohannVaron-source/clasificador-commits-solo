# Manual técnico — Clasificador de mensajes de commit

Dirigido a quien va a **mantener** la solución.

## 1. Arquitectura

```
Cliente (navegador / Postman)
        |  HTTP :8000
        v
+---------------------------+
| API FastAPI  (api-ia)     |  contenedor, usuario appuser (sin privilegios)
| POST /clasificar          |
| GET  /health              |
| GET  /inferencias         |
+------+--------------+-----+
       |              |
  motor "eco"     motor "ollama"
  (reglas, sin    HTTP -> host.docker.internal:11434
   modelo)        +------------------------------+
       |          | Ollama (en el equipo, fuera  |
       |          | de Docker) gemma3 / qwen2.5  |
       |          +------------------------------+
       v
+---------------------------+
| PostgreSQL 16 (db-ia)     |  red interna red-ia, puerto 5432
| tabla inferencias         |  volumen persistente pgdata
+---------------------------+
```

| Componente | Función | Puerto |
|---|---|---|
| API FastAPI | Recibe el texto del commit, lo clasifica con el motor elegido, registra la inferencia y responde en JSON. | 8000 (publicado) |
| Motor eco | Clasificación por expresiones regulares. Línea base de comparación; funciona sin modelo ni memoria extra. | — |
| Motor ollama | Clasificación con un LLM local mediante la API de Ollama. | 11434 (en el equipo) |
| PostgreSQL | Guarda cada inferencia. | 5432 (solo red interna de Compose) |

Orquestación: `docker-compose.yml` define los servicios `db` y `api`, la red `red-ia` y el volumen `pgdata`. La API espera a que la base de datos pase su `healthcheck` antes de arrancar.

## 2. Seguridad

**Puertos expuestos.** Solo el 8000 (API) se publica hacia el equipo anfitrión, porque es el único que los clientes necesitan. PostgreSQL no se publica: solo es alcanzable desde la red interna `red-ia`. Ollama escucha en el puerto 11434 del equipo anfitrión.

**Roles en la base de datos.**

| Rol | Uso | Privilegios |
|---|---|---|
| `postgres` | Administración, respaldos | Todos. No lo usa la aplicación. |
| `app_ia` | Conexión de la API | `CONNECT`, `USAGE` en el esquema `public`, `SELECT` e `INSERT` en `inferencias`, `USAGE`/`SELECT` en su secuencia. Sin `UPDATE`, `DELETE` ni `DROP`. |

**Secretos.** Las contraseñas viven en el archivo `.env`, que está en `.gitignore` y `.dockerignore` y nunca se sube al repositorio. En el repositorio solo existe `.env.example`, con los nombres de las variables y sin valores.

**Si se filtra una contraseña:** (1) cambiarla de inmediato en PostgreSQL (`ALTER ROLE ... PASSWORD ...`) y en el `.env`; (2) reiniciar la API con `docker compose restart api`; (3) si llegó al historial de Git, tratarla como comprometida aunque se borre el archivo, y rotarla; (4) revisar la tabla `inferencias` en busca de actividad anómala.

## 3. Endpoints

| Método | Ruta | Entrada | Respuesta | Errores |
|---|---|---|---|---|
| GET | `/health` | — | `{"estado":"ok","base_datos":"ok"}` | 503 si la base de datos no responde |
| POST | `/clasificar` | JSON: `texto` (obligatorio), `motor` (`eco` u `ollama`, opcional; por defecto el de `MOTOR_POR_DEFECTO`) | `motor`, `modelo`, `entrada`, `tipo`, `latencia_ms` | 400 si el motor no es válido; 422 si falta `texto`; 500 si Ollama no responde |
| GET | `/inferencias` | Query: `limite` (entero, por defecto 20) | Lista de las últimas inferencias, de más reciente a más antigua | 500 si falla la base de datos |

Documentación interactiva: `http://localhost:8000/docs`.

## 4. Modelo de datos

Tabla `inferencias`:

| Campo | Tipo | Significado |
|---|---|---|
| `id` | SERIAL, clave primaria | Identificador autoincremental |
| `fecha` | TIMESTAMP, por defecto `NOW()` | Momento en que se registró la inferencia |
| `motor` | VARCHAR(20) | `eco` u `ollama` |
| `modelo` | VARCHAR(120) | `reglas-v1` para eco; el nombre del modelo de Ollama en el otro caso |
| `entrada` | TEXT | Mensaje de commit enviado por el usuario |
| `salida` | TEXT | Tipo asignado (`feat`, `fix`, `docs`, `test`, `chore`, `refactor` o `desconocido`) |
| `latencia_ms` | INTEGER | Tiempo de clasificación en milisegundos |

## 5. Respaldo y restauración

**Procedimiento probado:**

```bash
# 1) Crear el respaldo
mkdir -p backups
docker compose exec -T db pg_dump -U postgres iadb > backups/respaldo_$(date +%F).sql

# 2) Restaurar (la tabla debe estar vacía o la base recién creada)
cat backups/respaldo_FECHA.sql | docker compose exec -T db psql -U postgres -d iadb

# 3) Verificar
docker compose exec -T db psql -U postgres -d iadb -c "SELECT COUNT(*) FROM inferencias;"
```

El respaldo es un volcado de `pg_dump` que incluye la definición de la tabla y sus datos. Al restaurar sobre una base que ya tiene la tabla, PostgreSQL muestra los errores `relation "inferencias" already exists` y `multiple primary keys for table "inferencias" are not allowed`: son esperados, porque esos pasos ya existen, y no impiden que se carguen las filas (`COPY`). Si la tabla aún tiene datos, la carga fallaría por claves duplicadas, por lo que antes hay que vaciarla o usar una base nueva.

**Prueba realizada (2026-11-26):**

| Paso | Registros en `inferencias` |
|---|---|
| Antes del desastre simulado | 614 |
| Tras `TRUNCATE inferencias` (desastre simulado) | 0 |
| Tras restaurar el respaldo | 614 |

El conteo volvió al valor original, por lo que el respaldo es restaurable.

**Periodicidad propuesta:** diaria, con retención de 7 días y una copia semanal guardada fuera del equipo.
**Responsable:** Johan Andres Varon (integrante único del equipo).

Los archivos `.sql` contienen datos de usuarios: `backups/` está en `.gitignore` y no se sube al repositorio.

## 6. Decisiones de diseño y limitaciones

**Decisiones**

- **Dockerfile multi-etapa:** la primera etapa instala dependencias y la segunda solo ejecuta, con lo que la imagen final es más liviana y no arrastra herramientas de construcción.
- **Usuario sin privilegios (`appuser`):** si alguien vulnera la aplicación, no obtiene permisos de administrador dentro del contenedor.
- **Motor eco:** funciona sin modelo ni memoria extra, sirve como línea base de comparación y permite que la solución opere en equipos de 4 GB o cuando Ollama no está disponible.
- **Privilegios mínimos en la base de datos:** la API solo necesita leer e insertar; limitar el rol reduce el daño de un error o de una inyección SQL.
- **Credenciales en `.env`:** separan configuración y código, y permiten usar valores distintos por entorno sin tocar el repositorio.
- **Volumen `pgdata`:** los datos sobreviven a `docker compose down`.
- **Healthcheck en la base de datos:** evita que la API arranque antes de que PostgreSQL esté lista.

**Limitaciones conocidas**

- La API no tiene autenticación: cualquiera con acceso al puerto 8000 puede clasificar y leer el historial.
- `/inferencias` acepta cualquier valor de `limite`, sin tope máximo.
- `db/init.sql` contiene la contraseña del rol `app_ia` en texto plano; en producción debería inyectarse desde un secreto.
- Las reglas del motor eco son sencillas: un mensaje que no coincide con ninguna se clasifica como `chore`.
- Con `OLLAMA_KEEP_ALIVE=0` el modelo se descarga tras cada respuesta, por lo que cada petición al motor ollama paga la carga: en la caracterización con el servicio desplegado, promedio de 3616 ms y p95 de 3901 ms (frente a 213 a 735 ms con el modelo residente en memoria).
- Ollama corre fuera de Docker, por lo que la solución depende de que esté instalado y accesible desde los contenedores.
- Se guardan en texto plano los mensajes que envían los usuarios; ver consideraciones de la Ley 1581 de 2012 en el informe técnico.
- `.env.example` incluye contraseñas de ejemplo para desarrollo: deben cambiarse en un servidor real, y como `db/init.sql` fija la clave de `app_ia`, hay que cambiarla también allí.
