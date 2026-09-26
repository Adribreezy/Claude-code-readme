# README.md

# claude-code-runner

Contenedor para ejecutar [Claude Code](https://docs.claude.com/en/docs/claude-code) de forma aislada, con un espacio de trabajo persistente. Pensado para uso personal o de un grupo pequeño y de confianza.

> Claude Code ejecuta comandos y edita ficheros por indicacion de un modelo. Tener acceso a este contenedor equivale a tener una shell dentro, con todas las credenciales que contenga. Lee SECURITY.md antes de compartirlo o exponerlo.

## Contenido del repositorio

```
.
├── Dockerfile
├── docker-compose.yml
├── .env.example        # plantilla sin valores reales
├── .gitignore
├── README.md
└── SECURITY.md
```

`.gitignore` debe incluir como minimo:

```
.env
data/
workspace/
*.pem
*.key
id_*
```

## Como compartirlo

- Comparte la receta (Dockerfile y compose), no la imagen construida. Cada persona construye la suya, asi no redistribuyes binarios ni capas con credenciales olvidadas. Revisa la licencia de cada paquete antes de redistribuir nada.
- Cada persona usa sus propias credenciales de Anthropic. No compartas tu suscripcion, tu sesion ni tu clave de API.
- Un contenedor y un volumen por persona. Nunca compartas volumen entre usuarios.
- Antes de publicar, revisa el historial de git: una credencial subida y borrada despues sigue estando en el historial.

## Requisitos

- Docker Engine 24 o superior y Docker Compose v2
- Una cuenta de Anthropic propia (suscripcion o clave de API)
- Un usuario con UID 1000 en el host, o ajusta `user:` en el compose

## Dockerfile

```dockerfile
FROM node:24-alpine

# Versiones fijas: no uses "latest" en un despliegue compartido
ARG CLAUDE_CODE_VERSION
RUN test -n "$CLAUDE_CODE_VERSION" || (echo "Define CLAUDE_CODE_VERSION" && exit 1)

RUN apk add --no-cache git ca-certificates tmux openssh-client curl bash

RUN npm install -g @anthropic-ai/claude-code@${CLAUDE_CODE_VERSION} \
 && npm cache clean --force

# Interfaz web opcional de terceros. Descomenta solo tras revisar su codigo
# y su modelo de autenticacion (ver SECURITY.md, seccion 6).
# ARG WEBUI_VERSION
# RUN npm install -g claude-code-webui@${WEBUI_VERSION}

ENV HOME=/home/node \
    CLAUDE_CONFIG_DIR=/home/node/.claude

USER node
WORKDIR /workspace

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD claude --version || exit 1

CMD ["tail", "-f", "/dev/null"]
```

## docker-compose.yml

```yaml
services:
  claude:
    build:
      context: .
      args:
        CLAUDE_CODE_VERSION: "X.Y.Z"      # fija una version concreta
    image: claude-code-runner:local
    container_name: claude-code-runner
    user: "1000:1000"
    init: true
    read_only: true
    tmpfs:
      - /tmp:size=256m
      - /home/node/.cache:size=256m,uid=1000,gid=1000
    cap_drop: [ALL]
    security_opt:
      - no-new-privileges:true             # mantiene el perfil seccomp por defecto
    pids_limit: 512
    mem_limit: 4g
    cpus: 2
    env_file: [.env]
    volumes:
      - ./workspace:/workspace
      - ./data/claude:/home/node/.claude
    networks: [runner]
    restart: unless-stopped
    # Solo si activas una interfaz web, y siempre en loopback:
    # ports:
    #   - "127.0.0.1:8080:8080"

networks:
  runner:
    driver: bridge
```

## .env.example

```
# Credenciales de Anthropic: usa UNA de las dos formas
ANTHROPIC_API_KEY=

# Cualquier otra variable que necesites. Usa tokens de minimo privilegio,
# de solo lectura y con caducidad siempre que se pueda.
# EXAMPLE_SERVICE_TOKEN=
```

## Primer arranque

```bash
cp .env.example .env            # edita los valores
mkdir -p workspace data/claude
sudo chown -R 1000:1000 workspace data
docker compose build
docker compose up -d
docker compose exec claude claude   # inicia sesion la primera vez
```

## Acceso

Por defecto solo hay acceso por shell: `docker compose exec claude bash`.
Si necesitas acceso remoto, usa un tunel SSH o una VPN. Si activas una interfaz web,
publicala unicamente en `127.0.0.1` y detras de un proxy inverso con autenticacion
y TLS. Nunca la expongas directamente a internet.

## Configuracion de Claude Code

Coloca en `data/claude/settings.json` reglas de permisos como punto de partida:

```json
{
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(~/.ssh/**)",
      "Bash(sudo:*)",
      "Bash(curl:*)",
      "Bash(wget:*)"
    ],
    "ask": [
      "Bash(git push:*)",
      "Bash(docker:*)"
    ]
  }
}
```

Estas reglas ayudan pero no son una barrera de seguridad. La barrera real es el
aislamiento del contenedor (ver SECURITY.md).

## Actualizacion

1. Cambia `CLAUDE_CODE_VERSION` en el compose tras leer las notas de version.
2. `docker compose build --pull && docker compose up -d`
3. Comprueba que el contenedor esta sano: `docker compose ps`

## Copias de seguridad

Respalda `workspace/`. **No respaldes ni sincronices `data/claude/`** sin cifrarlo:
contiene tu sesion y tu historial.

## Problemas frecuentes

- **Permission denied al escribir**: el propietario de `workspace/` y `data/` no es UID 1000.
- **No guarda la configuracion con `read_only`**: comprueba que `CLAUDE_CONFIG_DIR`
  apunta al volumen montado y que `/home/node/.cache` esta como tmpfs.
- **Cambios de version no se aplican**: reconstruye con `--no-cache`.
