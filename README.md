# Clasificador de mensajes de commit

Servicio de inferencia que recibe el texto de un mensaje de commit y lo clasifica como `feat`, `fix`, `docs`, `test`, `chore` o `refactor`, usando un motor por reglas (eco) o un modelo de lenguaje local ejecutado con Ollama. Cada petición se registra en PostgreSQL. Todo va contenerizado con Docker y validado por GitHub Actions.

## Equipo

| Integrante | Perfil de hardware |
|---|---|
| [Johan Andres varon] | [B] |

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

| Error | Causa | Cómo se resolvió |
|---|---|---|
| `fatal: detected dubious ownership in repository at '//wsl.localhost/...'` | Se ejecutó Git de Windows (PowerShell) sobre archivos que viven dentro de WSL, y Windows los ve como una carpeta de otro dueño. | Se abrió la terminal de Ubuntu y se ejecutaron ahí todos los comandos de Git, dentro de la carpeta del proyecto. |
| `error: src refspec feat/contenerizacion does not match any` | La rama no existía en la copia local: el commit nunca se hizo porque Git falló antes y los comandos se ejecutaron desde `~` y no desde la carpeta del repo. | Se entró a la carpeta del proyecto, se creó la rama con `git checkout -b`, se hizo el commit y luego el `git push origin <rama>`. |
| `remote: Password authentication is not supported for Git operations` | El remoto estaba configurado por HTTPS, y GitHub ya no acepta la contraseña de la cuenta para Git. | Se configuró una llave SSH en GitHub y se cambió el remoto con `git remote set-url origin git@github.com:USUARIO/REPO.git`. |
| `BLE001 Do not catch blind exception: Exception` (ruff, el CI fallaba en `calidad`) | En `/health` se capturaba `Exception` de forma genérica, algo que la regla de estilo no permite. | Se cambió por `except psycopg2.Error:`, se volvió a correr `ruff check app/` hasta ver `All checks passed!` y se subió el arreglo a la rama. |
| `invalid-syntax: Expected an expression` en `app/main.py` | Al editar la línea con nano el archivo quedó dañado (sangría y final de archivo corruptos). | Se restauró con `git checkout -- app/main.py` y se repitió el cambio con `sed -i`, que no abre el editor. |
| El CI fallaba en el job `pruebas` | El archivo `pytest.ini` no se había subido al repositorio (no aparecía en `git ls-files`), así que pytest no encontraba el módulo `app`. | Se creó `pytest.ini` con `pythonpath = .`, se hizo commit y push, y el CI volvió a correr. |
| Archivos `*:Zone.Identifier` dentro del repositorio | Windows los crea al copiar archivos descargados hacia WSL, y se colaron en un `git add`. | Se quitaron con `git rm --cached` y se agregó `*:Zone.Identifier` al `.gitignore`. |

## Documentación

- [Manual técnico](docs/manual-tecnico.md)
- [Informe técnico](docs/informe-tecnico.md)


