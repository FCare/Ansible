# Ansible - Traefik Network Deployment

**Ansible** - Named after Ursula K. Le Guin's instantaneous communication device from the Hainish Cycle, this deployment connects all your Docker services through a unified Traefik reverse proxy network.

## 🌐 Architecture

```
                               ┌─────────────────┐
                               │     Traefik     │
                               │  (Reverse Proxy) │
                               └────────┬────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    │                   │                   │
          ┌─────────▼──────┐                       ┌──────▼──────┐
          │ Voight-Kampff  │                       │  TTS/STT/   │
          │  (Auth Service) │                       │  LLM/Asst   │
          └────────────────┘                       └─────────────┘
               API Keys                             Protected by VK
```

### Services

1. **Traefik** (`traefik.mon_url.com`) - Reverse proxy with automatic HTTPS
2. **Voight-Kampff** (`auth.mon_url.com`) - API Key authentication service
3. **TTS Service** (`tts.mon_url.com`) - Text-to-Speech (protected by API keys)
4. **STT Service** (`stt.mon_url.com`) - Speech-to-Text (protected by API keys)
5. **LLM Service** (`llm.mon_url.com`) - Language Model (protected by API keys)
6. **Assistant Backend** (`assistant.mon_url.com`) - Assistant API (protected by API keys)

## 🔐 Authentication Strategy

All services (TTS, STT, LLM, Assistant) are protected by **Voight-Kampff** API key authentication:
- Fine-grained access control per service
- Multiple API keys with different scopes
- Bearer token authentication via HTTP headers

## 🚀 Getting Started

### Prerequisites

- Docker and Docker Compose installed
- A domain name pointing to your server
- Ports 80 and 443 available

### Installation

1. **Clone/Navigate to the ansible directory**
   ```bash
   cd ansible
   ```

2. **Update Traefik email for Let's Encrypt certificates**
   ```bash
   nano traefik/traefik.yml
   # Change line 33: email: your-email@example.com
   ```

3. **Update your domain in docker-compose.yml**
   ```bash
   sed -i 's/mon_url.com/yourdomain.com/g' docker-compose.yml
   ```

4. **Create required directories and set permissions**
   ```bash
   mkdir -p traefik/logs
   touch traefik/acme.json
   chmod 600 traefik/acme.json
   ```

5. **Update your service images**
   Edit `docker-compose.yml` and replace:
   - `your-tts-image:latest`
   - `your-stt-image:latest`
   - `your-llm-image:latest`
   - `your-assistant-image:latest`
   
   With your actual Docker images.

6. **Start the services**
   ```bash
   docker-compose up -d
   ```

7. **Check logs**
   ```bash
   docker-compose logs -f
   ```

## 🔑 Managing API Keys

### Create an API Key

```bash
curl -X POST https://auth.mon_url.com/keys \
  -H "Content-Type: application/json" \
  -d '{
    "key_name": "mobile-app",
    "user": "john",
    "scopes": ["tts", "stt", "llm", "assistant"],
    "expires_in_days": 365
  }'
```

Response:
```json
{
  "id": 1,
  "key_name": "mobile-app",
  "api_key": "kXy8vZ3mQ9wR5tN7pL4jH2fG6sA1dK0bC...",
  "user": "john",
  "scopes": ["tts", "stt", "llm", "assistant"],
  "is_active": true,
  "created_at": "2026-02-10T15:00:00",
  "last_used": null,
  "expires_at": "2027-02-10T15:00:00"
}
```

**⚠️ Important:** Save the `api_key` value - it won't be shown again!

### List All API Keys

```bash
curl https://auth.mon_url.com/keys
```

### Delete an API Key

```bash
curl -X DELETE https://auth.mon_url.com/keys/1
```

### Enable/Disable an API Key

```bash
curl -X PATCH https://auth.mon_url.com/keys/1/toggle
```

## 📱 Using API Keys

### Making Requests to Protected Services

Include the API key in the `Authorization` header:

```bash
curl https://tts.mon_url.com/api/synthesize \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"text": "Hello world"}'
```

### Python Example

```python
import requests

API_KEY = "your-api-key-here"
headers = {
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json"
}

# TTS Service
response = requests.post(
    "https://tts.mon_url.com/api/synthesize",
    headers=headers,
    json={"text": "Hello world"}
)

# STT Service
response = requests.post(
    "https://stt.mon_url.com/api/transcribe",
    headers=headers,
    files={"audio": open("audio.wav", "rb")}
)
```

## 🎯 Scopes and Permissions

Each API key can have specific scopes limiting access to services:

- `tts` - Text-to-Speech service only
- `stt` - Speech-to-Text service only
- `llm` - Language Model service only
- `assistant` - Assistant backend only
- `*` - All services (wildcard)

**Example: Create a key with limited access**
```bash
curl -X POST https://auth.mon_url.com/keys \
  -H "Content-Type: application/json" \
  -d '{
    "key_name": "tts-only-key",
    "user": "demo",
    "scopes": ["tts"],
    "expires_in_days": 30
  }'
```

This key can **only** access `tts.mon_url.com`, attempts to access other services will be denied.

## 🔧 Maintenance

### View Logs

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f traefik
docker-compose logs -f voight-kampff

# Traefik access logs
tail -f traefik/logs/access.log
```

### Restart Services

```bash
# All services
docker-compose restart

# Specific service
docker-compose restart traefik
```

### Update Services

```bash
docker-compose pull
docker-compose up -d
```

### Backup

```bash
# Backup Voight-Kampff database (API keys)
cp voight-kampff/data/voight-kampff.db voight-kampff.db.backup

# Backup Traefik certificates
cp traefik/acme.json acme.json.backup
```

## 🐛 Troubleshooting

### SSL Certificates Not Generated

1. Check Traefik logs: `docker-compose logs traefik`
2. Verify your domain DNS points to your server
3. Ensure ports 80 and 443 are open
4. Check `traefik/acme.json` permissions: `chmod 600 traefik/acme.json`

### API Key Not Working

1. Verify the key exists: `curl https://auth.mon_url.com/keys`
2. Check it's active and not expired
3. Verify the scope includes the service you're accessing
4. Check Voight-Kampff logs: `docker-compose logs voight-kampff`

### Service Not Accessible

1. Check if container is running: `docker-compose ps`
2. Verify Traefik labels: `docker inspect <container_name>`
3. Check Traefik dashboard: `https://traefik.mon_url.com`
4. Review Traefik logs: `docker-compose logs traefik`

## 📊 Network Details

- **Network Name:** `ansible`
- **Driver:** bridge
- All services communicate internally using container names
- Only Traefik exposes ports 80/443 to the host

## 🔐 Security Recommendations

1. **Use strong API keys** (automatically generated by Voight-Kampff)
2. **Set expiration dates** for temporary access
3. **Regularly review** active API keys
4. **Enable firewall** to only allow ports 80/443
5. **Keep services updated** with `docker-compose pull`
6. **Monitor access logs** regularly
7. **Backup** the Voight-Kampff database
8. **Change Traefik dashboard authentication** in `traefik/dynamic.yml`

## 📚 References

- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Let's Encrypt](https://letsencrypt.org/)

---

**Named after:** Ansible (Ursula K. Le Guin's Hainish Cycle) - Instantaneous communication across vast distances  
**Authentication:** Voight-Kampff (Blade Runner) - The test to determine humanity
