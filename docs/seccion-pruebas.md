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
