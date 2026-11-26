# Informe técnico – Caracterización del modelo local

## Caracterización inicial del modelo

| Dato | Valor |
|---|---|
| Perfil de hardware | Intel Core i5-9300H @ 2.40 GHz, 8 GB de RAM, NVIDIA GeForce GTX 1650 de 4 GB e Intel UHD Graphics 630. Sistema operativo Windows de 64 bits. |
| RAM total del equipo | 8 GB (7,84 GB utilizables) |
| Modelo y etiqueta | `gemma3:270m` |
| Tamaño en disco | 291 MB |
| Latencia de 5 ejecuciones (ms) | 19 694, 735, 268, 280 y 213 ms |
| Latencia promedio | 4 238 ms (4,238 segundos) |
| RAM usada durante la inferencia | 1,3 GiB usados en WSL según `free -h` (medición posterior a la inferencia; no representa necesariamente el pico de consumo). |
| Calidad percibida (1 a 5) | 4/5. El modelo respondió correctamente a las instrucciones sencillas y generó respuestas coherentes, aunque su capacidad es limitada para tareas más complejas. |

## Observaciones

El modelo Gemma 3 270M respondió correctamente a las cinco solicitudes realizadas. La primera ejecución presentó una latencia considerablemente superior a las posteriores, posiblemente debido a la carga inicial del modelo. En las siguientes ejecuciones, los tiempos de respuesta disminuyeron notablemente, llegando a valores inferiores a un segundo.

El equipo utilizado para las pruebas cuenta con un procesador Intel Core i5-9300H, 8 GB de RAM y una tarjeta gráfica NVIDIA GeForce GTX 1650 de 4 GB. El modelo tiene un tamaño de 291 MB en disco y presentó un promedio de latencia de 4,238 segundos en las cinco ejecuciones, incluyendo la primera carga.

La memoria observada en WSL después de la inferencia fue de 1,3 GiB usados, con 2,5 GiB disponibles. Esta medición corresponde al momento posterior a la ejecución y no necesariamente al pico de memoria alcanzado durante la generación.

La calidad percibida fue calificada con 4 sobre 5, debido a que el modelo respondió adecuadamente a instrucciones sencillas, aunque se considera que puede presentar limitaciones al realizar tareas más complejas.

Nota sobre el perfil: el equipo tiene 8 GB de RAM y disco SSD, por lo que corresponde al perfil B. Se usó gemma3:270m, el modelo del perfil C (291 MB), porque el SSD es pequeño y al iniciar el proyecto, tras Windows y las herramientas instaladas, quedaban unos 10 GB libres. Se priorizó un modelo liviano para no agotar el disco. Ese espacio está por debajo de los 15 GB que la guía pide como mínimo para el camino local.

## Plan de pruebas del despliegue

Perfil de hardware: B   Modelo: gemma3:270m   Fecha de ejecucion: 2026-11-26

| ID | Tipo | Que se verifica | Resultado esperado | Obtenido | Estado |
|----|------|-----------------|--------------------|----------|--------|
| P-01 | Funcional | GET /health responde | Codigo 200 y estado ok | Respuesta `{"estado":"ok","base_datos":"ok"}` | Aprobada |
| P-02 | Funcional | POST /clasificar con motor eco | Codigo 200 y tipo correcto | "corrige el error de login" → fix; "agrega el endpoint de historial" → feat; "actualiza el readme" → docs. Latencia de 0 a 2 ms. Las inferencias quedaron registradas (ids 6 a 8) | Aprobada |
| P-03 | Funcional | Motor invalido | Codigo 400 | Codigo 400 al enviar `"motor":"inventado"` | Aprobada |
| P-04 | Acceso | Rol app_ia intenta DROP TABLE | Error de permisos | DELETE rechazado: permission denied for table inferencias. DROP TABLE rechazado: must be owner of table inferencias | Aprobada |
| P-05 | Conectividad | La API resuelve el host db | Devuelve una IP interna | Devolvio `172.18.0.2` | Aprobada |
| P-06 | Disponibilidad | Reinicio del contenedor de BD | La API se recupera sola | Tras `docker compose restart db` y 15 s, `/health` respondio ok sin reiniciar la API | Aprobada |
| P-07 | Persistencia | down y up conservan los datos | Los registros siguen existiendo | Tras `down` y `up -d`, `/inferencias` devolvio los mismos registros (ids 8, 7 y 6, con las mismas fechas) | Aprobada |
| P-08 | Carga | 10 usuarios sobre el motor eco | p95 < 800 ms y errores < 5 % | p95 = 195,26 ms; errores 0,00 % (0 de 595); promedio 116,4 ms; maximo 246,43 ms; 1190 checks superados. `docker stats`: api 16 % CPU y 40,7 MiB; db 9,9 % CPU y 12,6 MiB | Aprobada |
| P-09 | Caracterizacion | 10 inferencias con modelo | Promedio, mediana y p95 | Modelo gemma3:270m. Promedio 3616 ms; mediana 3549 ms; p95 3901 ms; rango 3347 a 3984 ms. 3 de 10 respuestas sin categoria valida ("desconocido") | Aprobada |

### Analisis del cuello de botella

Comparando P-08 con P-09, la inferencia del modelo es el cuello de botella, no la API ni la base de datos.

- **Servicio sin modelo (P-08, motor eco):** promedio de 116,4 ms y p95 de 195,26 ms por peticion, con 10 usuarios simultaneos y 0 % de errores. La clasificacion en si toma de 0 a 2 ms (campo `latencia_ms`); el resto corresponde al recorrido HTTP por WSL y Docker y a abrir una conexion nueva a PostgreSQL con su `INSERT` en cada peticion.
- **Servicio con modelo (P-09, motor ollama):** promedio de 3616 ms y p95 de 3901 ms, de forma secuencial. Es unas 31 veces el promedio del motor eco; aproximadamente el 97 % del tiempo de cada peticion se gasta en la inferencia.
- **Por que tarda tanto:** con `OLLAMA_KEEP_ALIVE=0` el modelo se descarga de memoria al terminar cada respuesta, asi que cada peticion paga de nuevo la carga. Es coherente con que las diez mediciones sean casi iguales (3347 a 3984 ms) y con la ficha de la semana 1: en ese momento la configuracion no estaba aplicada y el modelo permanecia en memoria, por lo que las peticiones siguientes a la primera tardaron entre 213 y 735 ms.
- **Condiciones distintas:** P-08 se midio con 10 usuarios concurrentes y P-09 de forma secuencial; las cifras indican el orden de magnitud, no una comparacion exacta.

**Calidad de la clasificacion.** Para los mismos 10 mensajes, el modelo acerto 4 de 10 frente al tipo que se esperaria (3 respuestas no contuvieron ninguna categoria valida y 3 fueron otro tipo), mientras que el motor eco acerto 10 de 10. La comparacion favorece al eco porque los mensajes se redactaron con las palabras clave de sus reglas, asi que no demuestra que las reglas sean mejores en general; si muestra que un modelo de 270 millones de parametros es limitado en esta tarea.

### Propuestas de mejora

1. **Mantener el modelo cargado en memoria** (`OLLAMA_KEEP_ALIVE=5m`) en equipos con memoria suficiente. Con 8 GB de RAM y un modelo de 291 MB cabe sin problema, y evita pagar la carga en cada peticion: la ficha de la semana 1 muestra tiempos de 213 a 735 ms con el modelo residente, frente a unos 3,6 s ahora. El costo es mantener cerca de 1,3 GiB ocupados.
2. **Usar el motor eco como filtro previo y una cache para mensajes repetidos.** El eco responde en milisegundos y acierta en los casos que coinciden con sus reglas; el modelo solo se llamaria cuando el eco no reconoce el mensaje, y un mensaje ya clasificado no se recalcularia.
3. **Reutilizar conexiones a PostgreSQL** (pool de conexiones) en lugar de abrir una nueva por peticion, para reducir la parte de los 116 ms que no es clasificacion.
4. **Limitar la concurrencia hacia el modelo.** Con `OLLAMA_NUM_PARALLEL=1` las peticiones al modelo se atienden de una en una, asi que una cola con limite evita que muchas peticiones simultaneas se acumulen.


## Propuesta de escalamiento

> Responda con SUS observaciones del despliegue en el servidor remoto. Las preguntas guía están abajo.

### 1. Diferencias entre el despliegue local y el remoto
Rutas, puertos (¿cuál le asignaron?), secretos (¿cómo creó el `.env` en el servidor?), permisos y disponibilidad del modelo (¿había Ollama en el servidor?).

[COMPLETAR]

### 2. Qué componentes moverían a la nube y cuáles mantendrían, y por qué
Piense en: base de datos (servicio administrado vs. contenedor), API, modelo.

[COMPLETAR]

### 3. Dónde se ejecutaría el modelo con 1000 peticiones por minuto, y qué costo implica
Use sus mediciones de latencia (P-09): si una inferencia tarda X segundos, ¿cuántas peticiones por minuto resiste una instancia? ¿Qué implicaría usar GPU, un servicio administrado o varias réplicas?

[COMPLETAR]

### 4. Disponibilidad del servicio y conservación de los datos
Réplicas de la API, reinicio automático, respaldos periódicos fuera del servidor, copia de la base.

[COMPLETAR]

### 5. Protección de datos personales (Ley 1581 de 2012)
La tabla `inferencias` guarda el texto que escriben los usuarios, que podría contener datos personales. Considere: autorización y finalidad del tratamiento, información al titular, seguridad, tiempo de conservación y derecho de supresión (nota: el rol `app_ia` no puede borrar registros).

[COMPLETAR]
