# DearBook Local Development Configuration Review

## Current Configuration Issues for Local Development

### 1. Backend Environment Configuration
**File**: `backend/.env.example`
- ✅ Correct internal service URLs (using service names)
- ✅ Database connection properly configured
- ⚠️ Domain configuration uses `.docker.localhost` (good for local)

### 2. Frontend Environment Configuration
**File**: `frontend/docker-compose.yml`
```yaml
environment:
  - VITE_REVERB_APP_KEY=dearbook
  - VITE_REVERB_HOST=reverb.docker.localhost
  - VITE_API_HOST=api.docker.localhost
```
- ✅ Properly configured for local development

### 3. Traefik Configuration
**File**: `backend/docker-compose.yml`
- ✅ Uses `.docker.localhost` domains
- ✅ SSL certificates configured with mkcert
- ✅ Proper service routing

### 4. Database Configuration
**File**: `backend/config/database.php`
- ✅ Uses environment variables
- ✅ PostgreSQL connection configured

### 5. WebSocket Configuration
**File**: `backend/config/reverb.php`
- ✅ Configured for HTTP in local development
- ✅ Proper host and port settings

## Port Mappings & Service Communication

### Internal Service Communication (Docker Network)
- **TimescaleDB**: `timescaledb:5432`
- **Redis**: `redis:6379`
- **Ollama**: `ollama:11434`
- **ComfyUI**: `comfyui:8188`
- **Reverb**: `reverb:8080`

### External Access (via Traefik)
- **Frontend**: https://dearbook.docker.localhost
- **API**: https://api.docker.localhost
- **Assets**: https://assets.docker.localhost
- **Horizon**: https://horizon.docker.localhost
- **ComfyUI**: https://comfyui.docker.localhost
- **Reverb WebSocket**: wss://reverb.docker.localhost

## Required Setup Steps for Local Development

### 1. SSL Certificate Generation
```bash
mkcert -install -cert-file ./backend/traefik/tls/cert.pem -key-file ./backend/traefik/tls/key.pem "*.docker.localhost" docker.localhost
```

### 2. Environment File Setup
```bash
cd backend
cp .env.example .env
# Generate APP_KEY
./php artisan key:generate
```

### 3. Docker Network Setup
The backend docker-compose.yml creates a `traefik-network` that the frontend needs to connect to.

### 4. Database Initialization
```bash
cd backend
./php artisan migrate:fresh
./php artisan storage:link
```

### 5. AI Model Setup
```bash
cd backend
./ollama pull llama3.2:3b
./ollama pull mxbai-embed-large
```

## Potential Configuration Issues

### 1. Frontend API Communication
**File**: `frontend/src/api.ts`
```typescript
const API_HOST = import.meta.env.VITE_API_HOST
// Uses: https://${API_HOST}/...
```
- ✅ Properly uses environment variable

### 2. WebSocket Connection
**File**: `frontend/src/main.ts`
```typescript
window.Echo = new Echo({
    broadcaster: 'reverb',
    key: import.meta.env.VITE_REVERB_APP_KEY,
    wsHost: import.meta.env.VITE_REVERB_HOST,
    wsPort: 80,
    wssPort: 443,
    forceTLS: true,
})
```
- ✅ Properly configured for HTTPS/WSS

### 3. CORS Configuration
**File**: `backend/config/cors.php`
```php
'allowed_origins' => [ '*' ],
```
- ✅ Allows all origins (good for development)

## Network Architecture

```
┌─────────────────┐    ┌─────────────────┐
│   Frontend      │    │   Traefik       │
│   (Vue.js)      │◄──►│   (Reverse      │
│   Port: 8080    │    │    Proxy)       │
└─────────────────┘    │   Ports: 80,443 │
                       └─────────┬───────┘
                                 │
                       ┌─────────▼───────┐
                       │   Backend       │
                       │   (Laravel)     │
                       │   FrankenPHP    │
                       └─────────┬───────┘
                                 │
        ┌────────────────────────┼────────────────────────┐
        │                        │                        │
┌───────▼───────┐    ┌───────────▼──────┐    ┌───────────▼──────┐
│  TimescaleDB  │    │     Redis        │    │    Reverb        │
│  Port: 5432   │    │   Port: 6379     │    │   Port: 8080     │
└───────────────┘    └──────────────────┘    └──────────────────┘
        │                        │
┌───────▼───────┐    ┌───────────▼──────┐
│    Ollama     │    │    ComfyUI       │
│  Port: 11434  │    │   Port: 8188     │
└───────────────┘    └──────────────────┘
```

## Files That Need Configuration

### Critical Configuration Files:
1. `backend/.env` - Main backend configuration
2. `backend/traefik/tls/cert.pem` & `key.pem` - SSL certificates
3. `frontend/docker-compose.yml` - Frontend environment variables

### Optional Configuration Files:
1. `backend/config/app.php` - Application settings
2. `backend/config/database.php` - Database configuration
3. `backend/config/reverb.php` - WebSocket configuration
4. `backend/config/cors.php` - CORS settings

## Startup Order Dependencies

1. **Traefik** (reverse proxy)
2. **TimescaleDB** (database)
3. **Redis** (cache/queue)
4. **Ollama** (AI models)
5. **ComfyUI** (image generation)
6. **Backend Services** (PHP, Queue, Reverb, Scheduler)
7. **Frontend** (Vue.js app)

The current docker-compose configuration handles these dependencies correctly with the `depends_on` and network configurations.