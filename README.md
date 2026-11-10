# 🔎 Open WebUI + SearXNG en Docker

![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![Open WebUI](https://img.shields.io/badge/Open%20WebUI-main-2563eb)
![SearXNG](https://img.shields.io/badge/SearXNG-latest-3050ff)
![Arch](https://img.shields.io/badge/arm64%20%7C%20amd64-compatible-0ea5e9)

Búsqueda web para **Open WebUI** usando **SearXNG** autoalojado, sin API keys ni cuotas. Cada servicio vive en su propio `docker compose` y se comunican por una red Docker externa compartida (`ai`).

```bash
$ whoami
genbyte  # homelab · self-hosting · docker
```

## 📁 Estructura

```
.
├── open-webui/
│   ├── compose.yml
│   ├── .env                  # WEBUI_SECRET_KEY
│   └── open-webui-data/      # datos persistentes (se crea solo)
└── searxng/
    ├── docker-compose.yml
    └── searxng/
        └── settings.yml
```

## 🧩 Cómo funciona

- Open WebUI y SearXNG están en la red externa `ai` y se ven por nombre de contenedor.
- Open WebUI consulta a SearXNG por el puerto **interno** `8080`.
- El puerto publicado de SearXNG (`8082`) es solo para usar su interfaz web desde el navegador. Puedes quitarlo si no lo necesitas.

## 🚀 Instalación

### 1. Crear la red (una sola vez)

```bash
docker network create ai
```

### 2. SearXNG

```bash
mkdir -p searxng/searxng && cd searxng

cat > searxng/settings.yml <<'EOF'
use_default_settings: true

server:
  secret_key: "CAMBIA_ESTO"
  limiter: false
  image_proxy: false

search:
  formats:
    - html
    - json
EOF

sed -i "s|CAMBIA_ESTO|$(openssl rand -hex 32)|" searxng/settings.yml
```

> ⚠️ El formato `json` es imprescindible. Sin él, Open WebUI recibe un error 403.

`searxng/docker-compose.yml`:

```yaml
services:
  searxng:
    container_name: searxng
    image: searxng/searxng:latest
    ports:
      - "8082:8080"   # host:interno (cambia el 8082 si está ocupado)
    volumes:
      - ./searxng:/etc/searxng
    networks:
      - ai
    restart: unless-stopped

networks:
  ai:
    external: true
```

```bash
docker compose up -d
```

### 3. Open WebUI

`open-webui/.env`:

```env
WEBUI_SECRET_KEY=pon-aqui-una-clave-larga-aleatoria
```

Genera la clave con `openssl rand -hex 32`.

`open-webui/compose.yml`:

```yaml
services:
  openwebui:
    container_name: open-webui
    image: ghcr.io/open-webui/open-webui:main
    ports:
      - "7000:8080"
    environment:
      - WEBUI_SECRET_KEY=${WEBUI_SECRET_KEY}
      - ENABLE_OLLAMA_API=false
      - OLLAMA_API_BASE_URL=""
      - WEBUI_AUTH=True
      - WEBUI_REGISTER=false
      - ENABLE_WEB_SEARCH=true
      - WEB_SEARCH_ENGINE=searxng
      - SEARXNG_QUERY_URL=http://searxng:8080/search?q=<query>
    volumes:
      - ./open-webui-data:/app/backend/data
    networks:
      - ai
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

networks:
  ai:
    external: true
```

```bash
docker compose up -d
```

## ⚙️ Activar la búsqueda en la interfaz

Open WebUI lee las variables de entorno solo en el primer arranque. Si ya tenía configuración guardada, manda la interfaz:

1. **Panel de administración → Ajustes → Búsqueda web**
2. Activa el interruptor **Búsqueda web**.
3. Motor de búsqueda: `searxng`
4. URL de consulta: `http://searxng:8080/search?q=<query>`
5. Resultados: 3-5. Concurrencia: 1-2 (recomendado en Raspberry Pi).
6. **Guardar**.

Después, en un chat nuevo, activa la búsqueda con el botón `+` o el icono del globo.

> 💡 Usa siempre el puerto interno **8080** en esa URL, nunca el publicado (8082).

## ✅ Comprobaciones

```bash
# Contenedores conectados a la red
docker network inspect ai --format '{{range .Containers}}{{.Name}} {{end}}'

# Open WebUI llega a SearXNG y recibe JSON
docker exec open-webui curl -s "http://searxng:8080/search?q=test&format=json" | head -c 200
```

| Servicio | URL |
|---|---|
| Open WebUI | `http://IP-DEL-SERVIDOR:7000` |
| SearXNG | `http://IP-DEL-SERVIDOR:8082` |

## 🛠️ Problemas frecuentes

| Síntoma | Solución |
|---|---|
| `curl` devuelve 403 | Falta `json` en `search.formats`. Añádelo y `docker compose restart searxng`. |
| `Could not resolve host: searxng` | Los contenedores no comparten red. Revisa `networks: - ai` en ambos composes. |
| SearXNG no arranca por permisos | `sudo chown -R 977:977 searxng/searxng` |
| Las variables del compose no hacen efecto | Configúralo en Ajustes → Búsqueda web. Alternativa: `ENABLE_PERSISTENT_CONFIG=False` (los cambios desde la interfaz no se conservarán). |
| No aparece el motor `searxng` | Imagen antigua: `docker compose pull openwebui && docker compose up -d` |
| Un motor te bloquea por exceso de peticiones | Desactívalo en `settings.yml` y deja DuckDuckGo, Brave o Startpage. |

## 🔄 Mantenimiento

```bash
# Actualizar (dentro de cada carpeta)
docker compose pull && docker compose up -d

# Logs
docker logs searxng --tail 50
docker logs open-webui --tail 50
```

## 🔒 Seguridad

- No expongas SearXNG directamente a internet.
- Para acceso remoto, usa una VPN (WireGuard, Tailscale) o un proxy inverso con HTTPS y autenticación.
- Si lo publicas, activa `limiter: true` (requiere Valkey/Redis).
- No subas el archivo `.env` al repositorio: añádelo a `.gitignore`.

## 📚 Referencias

- [SearXNG](https://github.com/searxng/searxng) · [documentación](https://docs.searxng.org/)
- [Open WebUI](https://github.com/open-webui/open-webui) · [documentación](https://docs.openwebui.com/)

## 📬 GENBYTE

[![YouTube](https://img.shields.io/badge/YouTube-GENBYTE-FF0000?logo=youtube&logoColor=white)](https://www.youtube.com/@genbyte?sub_confirmation=1)
[![Blog](https://img.shields.io/badge/Blog-genbyte.blogspot.com-FF5722?logo=blogger&logoColor=white)](https://genbyte.blogspot.com)
[![GitHub](https://img.shields.io/badge/GitHub-jlalib-181717?logo=github)](https://github.com/jlalib)
