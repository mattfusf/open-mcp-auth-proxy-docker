# Docker Deployment Guide

This guide explains how to deploy the MCP Authorization Proxy using Docker and Docker Compose.

## Prerequisites

- Docker 20.10 or later
- Docker Compose 2.0 or later

## Quick Start

### 1. Basic Deployment (Demo Mode)

The simplest way to get started is using demo mode with the included example MCP server:

```bash
# Build and start services
docker-compose up -d

# View logs
docker-compose logs -f

# Stop services
docker-compose down
```

This will:
- Start the MCP Authorization Proxy on `http://localhost:8080`
- Start the example Python Echo MCP server on `http://localhost:8000`
- Use demo Asgardeo credentials for authentication

### 2. Custom Configuration

#### Using Environment Variables

Copy the example environment file and customize it:

```bash
cp .env.example .env
# Edit .env with your settings
```

#### Using Custom config.yaml

Edit the `config.yaml` file to configure:
- OAuth provider settings
- Backend MCP server URL
- CORS settings
- Transport mode (SSE, stdio, streamable HTTP)
- Protected resource metadata

```bash
# Edit config.yaml
vim config.yaml

# Restart services to apply changes
docker-compose restart proxy
```

## Deployment Modes

### Demo Mode (Default)

Uses Asgardeo sandbox for testing:

```bash
docker-compose up -d
```

Or explicitly:

```bash
docker-compose run --rm proxy --demo
```

### Asgardeo Mode

For your own Asgardeo organization:

1. Update `config.yaml` with your Asgardeo details:
   ```yaml
   demo:
     org_name: "your-org-name"
     client_id: "your-client-id"
     client_secret: "your-client-secret"
   ```

2. Start with Asgardeo mode:
   ```bash
   docker-compose run --rm -p 8080:8080 proxy --asgardeo
   ```

### Stdio Mode

For MCP servers that use stdio transport:

1. Update `config.yaml`:
   ```yaml
   transport_mode: "stdio"
   stdio:
     enabled: true
     user_command: "npx -y @modelcontextprotocol/server-github"
     env:
       - "GITHUB_PERSONAL_ACCESS_TOKEN=your-token"
   ```

2. Start with stdio mode:
   ```bash
   docker-compose run --rm -p 8080:8080 proxy --stdio
   ```

## Using Only the Proxy

If you have your own MCP server running elsewhere:

1. Update `config.yaml`:
   ```yaml
   base_url: "http://your-mcp-server:8000"
   port: 8000
   ```

2. Start only the proxy:
   ```bash
   docker-compose up -d proxy
   ```

## Building Custom Images

### Build the Proxy Image

```bash
docker build -t mcp-auth-proxy:latest .
```

### Build the Example MCP Server Image

```bash
docker build -f resources/Dockerfile.mcp-server -t mcp-echo-server:latest ./resources
```

## Advanced Configuration

### Using Docker Compose Override

Create a `docker-compose.override.yml` for local customizations:

```yaml
version: '3.8'

services:
  proxy:
    environment:
      - DEBUG=true
    ports:
      - "9090:8080"  # Custom port mapping
```

### Connecting to External Services

To connect to external MCP servers or databases:

```yaml
services:
  proxy:
    extra_hosts:
      - "external-mcp-server:192.168.1.100"
```

### Using External Config

Mount a custom config from outside the project:

```yaml
services:
  proxy:
    volumes:
      - /path/to/your/config.yaml:/app/config.yaml:ro
```

## Health Checks

Both services include health checks:

```bash
# Check proxy health
curl http://localhost:8080/.well-known/protected-resource-metadata

# Check service status
docker-compose ps
```

## Troubleshooting

### View Logs

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f proxy
docker-compose logs -f mcp-server
```

### Restart Services

```bash
# Restart all
docker-compose restart

# Restart specific service
docker-compose restart proxy
```

### Rebuild After Code Changes

```bash
# Rebuild and restart
docker-compose up -d --build

# Force rebuild
docker-compose build --no-cache
```

### Common Issues

1. **Port already in use**
   - Change the port mapping in `docker-compose.yml` or `.env`
   - Example: `"9090:8080"` instead of `"8080:8080"`

2. **Connection refused to MCP server**
   - Ensure the MCP server is running: `docker-compose ps`
   - Check network connectivity: `docker-compose exec proxy ping mcp-server`
   - Verify `base_url` in `config.yaml` uses service name: `http://mcp-server:8000`

3. **Authentication failures**
   - Verify OAuth provider credentials in `config.yaml`
   - Check JWKS URI is accessible from the container
   - Review logs: `docker-compose logs -f proxy`

## Production Deployment

For production use:

1. **Use secrets management**
   - Don't commit credentials to `config.yaml`
   - Use Docker secrets or environment variables
   - Consider using external secret managers

2. **Enable HTTPS**
   - Use a reverse proxy (nginx, Traefik, Caddy)
   - Configure TLS certificates
   - Update `proxy_base_url` and CORS settings

3. **Resource limits**
   ```yaml
   services:
     proxy:
       deploy:
         resources:
           limits:
             cpus: '0.5'
             memory: 512M
           reservations:
             cpus: '0.25'
             memory: 256M
   ```

4. **Logging**
   - Configure log drivers for centralized logging
   - Use log aggregation tools (ELK, Splunk, etc.)

5. **Monitoring**
   - Implement health check endpoints
   - Use Prometheus for metrics
   - Set up alerts for service failures

## Example: Production Setup with Nginx

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      - proxy
    networks:
      - mcp-network

  proxy:
    build: .
    expose:
      - "8080"
    networks:
      - mcp-network
    restart: always
```

## Security Best Practices

1. **Run as non-root** (already configured in Dockerfile)
2. **Use read-only volumes** for configuration files
3. **Limit network exposure** - only expose necessary ports
4. **Keep images updated** - regularly rebuild with latest base images
5. **Scan for vulnerabilities** - use `docker scan` or similar tools
6. **Use minimal base images** - Alpine Linux reduces attack surface

## References

- [Main README](./README.md)
- [Auth0 Integration](./docs/integrations/Auth0.md)
- [Keycloak Integration](./docs/integrations/keycloak.md)
- [MCP Protocol Specification](https://modelcontextprotocol.io/)
