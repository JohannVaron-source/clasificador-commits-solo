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


## Plan de pruebas del despliegue

Perfil de hardware: ____   Modelo: ____   Fecha de ejecucion: ____

| ID | Tipo | Que se verifica | Resultado esperado | Obtenido | Estado |
|----|------|-----------------|--------------------|----------|--------|
| P-01 | Funcional | GET /health responde | Codigo 200 y estado ok | | |
| P-02 | Funcional | POST /clasificar con motor eco | Codigo 200 y tipo correcto | | |
| P-03 | Funcional | Motor invalido | Codigo 400 | | |
| P-04 | Acceso | Rol app_ia intenta DROP TABLE | Error de permisos | | |
| P-05 | Conectividad | La API resuelve el host db | Devuelve una IP interna | | |
| P-06 | Disponibilidad | Reinicio del contenedor de BD | La API se recupera sola | | |
| P-07 | Persistencia | down y up conservan los datos | Los registros siguen existiendo | | |
| P-08 | Carga | 10 usuarios sobre el motor eco | p95 < 800 ms y errores < 5 % | | |
| P-09 | Caracterizacion | 10 inferencias con modelo | Promedio, mediana y p95 | | |

### Analisis del cuello de botella

(Compare P-08 con P-09 con SUS numeros: donde se pierde el tiempo, en la API,
en la base de datos o en la inferencia, y por que.)

### Propuestas de mejora

1. (Propuesta y justificacion)
2. (Propuesta y justificacion)


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
