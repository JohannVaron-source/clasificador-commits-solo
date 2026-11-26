## Propuesta de escalamiento (máximo dos páginas)

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
