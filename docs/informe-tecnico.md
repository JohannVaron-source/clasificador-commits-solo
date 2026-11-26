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



### 1. Diferencias entre el despliegue local y el remoto

Durante el despliegue local, el sistema se ejecutó utilizando Docker Compose, con dos contenedores principales: la API desarrollada con FastAPI y la base de datos PostgreSQL. La API se expuso mediante el puerto 8000 y se comunicó con PostgreSQL a través de la red interna de Docker, utilizando el nombre del servicio `db`.

En el entorno remoto, el despliegue requiere adaptar la configuración a las rutas, puertos y permisos asignados por el servidor. Las variables sensibles, como las credenciales de PostgreSQL, deben configurarse mediante un archivo `.env` creado directamente en el servidor, evitando que este archivo se publique en GitHub. También es necesario comprobar que Docker tenga los permisos adecuados para ejecutar los contenedores.

En cuanto al modelo, la API utiliza Ollama mediante una conexión HTTP. Ollama se encuentra disponible en `0.0.0.0:11434`, utilizando el puerto 11434 para las solicitudes. Esto permite que la API FastAPI pueda comunicarse con el servicio de inferencia desde la red, siempre que la configuración de conectividad y los permisos del servidor lo permitan. De esta manera, el modelo no necesita ejecutarse dentro del mismo contenedor de la API, sino que puede funcionar como un servicio independiente.

### 2. Qué componentes moverían a la nube y cuáles mantendrían, y por qué

Para escalar el proyecto, se propone trasladar los componentes principales a una infraestructura en la nube, permitiendo mejorar la disponibilidad, facilitar el mantenimiento y soportar una mayor cantidad de solicitudes.

- **Base de datos:** Se utilizaría PostgreSQL mediante un servicio administrado en la nube. Esto permitiría delegar tareas como mantenimiento, respaldos, actualizaciones y supervisión, reduciendo la dependencia de un único servidor. Para un entorno de pruebas o de bajo presupuesto, PostgreSQL en Docker puede mantenerse.
- **API:** La API FastAPI se desplegaría en contenedores, permitiendo crear varias réplicas para distribuir las solicitudes y mejorar la disponibilidad. Docker facilitaría la portabilidad entre ambientes locales y remotos.
- **Modelo de IA:** Ollama podría mantenerse en un servidor dedicado con recursos suficientes, especialmente memoria y GPU si se requiere mayor velocidad. Para un sistema con alta demanda, se evaluaría utilizar infraestructura especializada para inferencia o servicios administrados de modelos, comparando rendimiento y costo.

La separación de estos componentes permitiría escalar cada uno de manera independiente. Por ejemplo, se podrían aumentar las réplicas de la API sin necesidad de duplicar la base de datos o el modelo de IA.

### 3. Dónde se ejecutaría el modelo con 1000 peticiones por minuto, y qué costo implica

De acuerdo con la prueba P-09, el modelo `gemma3:270m` presentó una latencia promedio de 3.616 segundos por inferencia, una mediana de 3.549 segundos y un percentil 95 (p95) de 3.901 segundos. Estas mediciones se realizaron de forma secuencial y con la configuración `OLLAMA_KEEP_ALIVE=0`, lo que implica que el modelo se descargaba de memoria después de cada respuesta.

Si una instancia procesara las solicitudes de manera secuencial, su capacidad teórica sería:

60 / 3.616 ≈ 16,6 peticiones por minuto.

Para atender 1000 peticiones por minuto, se necesitarían aproximadamente 61 instancias trabajando de manera secuencial, bajo el supuesto de que todas mantuvieran el mismo rendimiento y sin considerar la sobrecarga adicional.

| Dato | Valor estimado |
|---|---:|
| Demanda requerida | 1000 peticiones/minuto |
| Peticiones por segundo | 16,67 |
| Latencia promedio P-09 | 3,616 segundos |
| Capacidad teórica por instancia secuencial | 16,6 peticiones/minuto |
| Instancias teóricas necesarias | 61 |
| Concurrencia aproximada requerida | 60 peticiones simultáneas |
| Concurrencia considerando p95 | 65 aproximadamente |

Para atender esta demanda, el modelo no se ejecutaría en el equipo local utilizado durante las pruebas, sino en servidores especializados en inferencia, preferiblemente con GPU, suficiente memoria y capacidad para manejar solicitudes concurrentes.

Se podrían evaluar tres alternativas:

1. **Servidor dedicado con GPU:** permitiría ejecutar el modelo con mayor velocidad y atender varias inferencias simultáneamente. Su costo dependería del tipo de GPU, memoria, horas de uso y proveedor.
2. **Servicio administrado de inferencia:** evitaría administrar directamente el hardware y facilitaría la escalabilidad, pero el costo dependería del volumen de solicitudes, tokens procesados o tiempo de ejecución.
3. **Varias réplicas del modelo:** permitirían distribuir las solicitudes entre diferentes instancias, reduciendo la saturación de un único servidor y mejorando la capacidad de procesamiento.

La cantidad de instancias calculada es una estimación teórica basada en la medición P-09. En un entorno real sería necesario realizar pruebas de carga concurrentes, debido a que el rendimiento cambia dependiendo de la GPU, la memoria, la configuración de Ollama, el tamaño del modelo y la cantidad de solicitudes simultáneas.
### 4. Disponibilidad del servicio y conservación de los datos

Para mejorar la disponibilidad del sistema y evitar la pérdida de información, se implementarían mecanismos de redundancia, recuperación y respaldo.

- **Réplicas de la API:** se desplegarían varias instancias de FastAPI detrás de un balanceador de carga, permitiendo distribuir las peticiones y evitar que la caída de una instancia interrumpa todo el servicio.
- **Reinicio automático:** se configuraría una política de reinicio para los contenedores, de manera que puedan recuperarse automáticamente ante fallos.
- **Respaldos periódicos:** se realizarían copias de seguridad de la base de datos PostgreSQL de forma programada y se almacenarían fuera del servidor principal.
- **Conservación de datos:** se mantendría un respaldo de la base de datos que permita recuperar las inferencias y demás información registrada en caso de daños, fallos o pérdida del servidor.
- **Monitoreo:** se supervisaría el estado de la API, la base de datos, el modelo, el uso de CPU y memoria, además de las latencias y errores del servicio.

En el despliegue actual, el proyecto ya cuenta con un volumen persistente de PostgreSQL (`pgdata`), lo cual permite conservar los datos al recrear los contenedores. Sin embargo, un volumen local no reemplaza los respaldos externos, ya que si se pierde el servidor también podrían perderse los datos almacenados en él.

### 5. Protección de datos personales (Ley 1581 de 2012)

La tabla `inferencias` almacena los textos enviados por los usuarios para clasificarlos. Aunque estos textos corresponden principalmente a mensajes de commit, podrían incluir información personal, como nombres, correos electrónicos, identificadores u otros datos que permitan identificar a una persona.

Por este motivo, el sistema debería implementar medidas para cumplir con los principios de protección de datos personales establecidos en la Ley 1581 de 2012:

- **Autorización y finalidad:** se informaría al usuario que el texto enviado será procesado para realizar la clasificación y almacenado con fines de registro, evaluación y funcionamiento del sistema. Se obtendría la autorización correspondiente cuando sea exigible.
- **Información al titular:** se dispondría de una política de tratamiento de datos que explique qué información se recolecta, para qué se utiliza, cuánto tiempo se conserva y cómo ejercer los derechos correspondientes.
- **Seguridad:** se protegería el acceso a la base de datos mediante credenciales seguras, permisos limitados, conexiones protegidas y restricciones para evitar consultas o modificaciones no autorizadas.
- **Tiempo de conservación:** se establecería un periodo definido para conservar los registros de inferencias, evitando almacenar información indefinidamente cuando ya no sea necesaria para la finalidad establecida.
- **Derecho de supresión:** se habilitaría un procedimiento para atender solicitudes de eliminación de datos personales cuando legalmente proceda. Actualmente, el rol `app_ia` no tiene permisos para borrar registros, por lo que este proceso tendría que ejecutarse mediante un mecanismo administrativo controlado, sin otorgarle permisos de eliminación indiscriminados a la aplicación.

Además, se podría reducir la cantidad de información almacenada, evitando registrar datos innecesarios, y utilizar anonimización o eliminación de datos identificables cuando sea posible.

Estas medidas ayudarían a proteger la privacidad de los usuarios, reducir riesgos de exposición de información y mejorar la seguridad del sistema.
