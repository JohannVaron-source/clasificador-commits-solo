# Clasificador de mensajes de commit

Servicio de inferencia que recibe el texto de un mensaje de commit y lo clasifica como `feat`, `fix`, `docs`, `test`, `chore` o `refactor`, usando un motor por reglas (eco) o un modelo de lenguaje local ejecutado con Ollama. Cada petición se registra en PostgreSQL. Todo va contenerizado con Docker y validado por GitHub Actions.

## Equipo

| Integrante | Perfil de hardware |
|---|---|
| [COMPLETAR] | [A / B / C / D] |

## Requisitos

- 4 GB de RAM como mínimo y 15 GB libres en disco
- Virtualización habilitada
- Windows 10 versión 2004 o superior (con WSL2 y Ubuntu 24.04) o cualquier distribución de Linux
- Docker Engine y Docker Compose, Git
- Ollama (solo para el motor `ollama`)

## Instalación

```bash
# 1) Clonar
git clone git@github.com:USUARIO/REPOSITORIO.git
cd REPOSITORIO

# 2) Preparar el entorno (instala utilidades, Docker y Ollama)
chmod +x setup.sh diagnostico.sh
./setup.sh
# Cierre y reabra la terminal para usar docker sin sudo

# 3) (Opcional, motor ollama) Descargar el modelo de su perfil
ollama pull gemma3:270m          # perfil C; A: qwen2.5-coder:1.5b, B: qwen2.5:0.5b

# 4) Variables de entorno
cp .env.example .env
nano .env                        # complete los valores

# 5) Levantar la solución
docker compose up -d --build
```

> Si usa el motor `ollama` desde Docker, Ollama debe escuchar en `0.0.0.0` (variable `OLLAMA_HOST`).

## Cómo verificar

```bash
docker compose ps                         # db (healthy) y api (running)
curl http://localhost:8000/health         # {"estado":"ok","base_datos":"ok"}
curl -X POST http://localhost:8000/clasificar \
  -H "Content-Type: application/json" \
  -d '{"texto":"corrige el error de login","motor":"eco"}'
curl "http://localhost:8000/inferencias?limite=5"
```

Documentación interactiva: http://localhost:8000/docs

## Endpoints

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/health` | Estado del servicio y de la base de datos |
| POST | `/clasificar` | Clasifica un mensaje y registra la inferencia |
| GET | `/inferencias` | Últimas inferencias registradas |

## Pruebas

```bash
pip install -r requirements.txt
pytest -v
```

Prueba de carga con k6: `tests/carga/prueba_carga.js`. Caracterización del modelo: `tests/carga/caracterizar_modelo.py`.

## Solución de problemas

[COMPLETAR: mínimo cinco errores REALES que le ocurrieron al equipo]

| Error | Causa | Cómo se resolvió |
|---|---|---|
| | | |
| | | |
| | | |
| | | |
| | | |

## Documentación

- [Manual técnico](docs/manual-tecnico.md)
- [Informe técnico](docs/informe-tecnico.md)

## Video de demostración

[COMPLETAR: enlace al video de 3 a 5 minutos]
