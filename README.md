# Traefik with Consul Service Discovery

This setup allows services running on remote Docker hosts in your home network to automatically register with Traefik through Consul.

## Architecture

- **Main Host**: Runs Traefik and Consul server
- **Remote Hosts**: Run Consul agents that register local services with the Consul server
- **Service Discovery**: Traefik reads service catalog from Consul and automatically creates routes with TLS

## Main Host Setup (Traefik + Consul Server)

### 1. Create the External Docker Network

Before starting Traefik, you must create the external Docker network:

```bash
docker network create traefik
```

### 2. Required Environment Variables

Add to your `.env` file by copying the [.env.template](.env.template) and adapting the values.

```bash
cp .env.template .env
```

### 3. Start the Services

```bash
cd traefik-consul
docker compose up -d
```

This will start:
- Traefik (reverse proxy with automatic TLS)
- Consul (service discovery)
- Logrotate (log management)

### 4. Access Consul UI

- URL: `https://consul.home.ditschers.de` or `https://consul.home.ditscher.me`
- Authentication: Use the CONSUL_BASIC_AUTH credentials

## Remote Host Setup

On each remote machine that should register services with Traefik:

### 1. Install Consul Agent

Create a `docker-compose.yml` for the Consul agent:

```yaml
services:
  consul-agent:
    image: consul:latest
    restart: unless-stopped
    command: agent -retry-join=<MAIN_HOST_IP> -bind='{{ GetInterfaceIP "eth0" }}'
    network_mode: host
    volumes:
      - ./consul-data:/consul/data
    environment:
      - CONSUL_BIND_INTERFACE=eth0
```

Replace `<MAIN_HOST_IP>` with the IP address of your main Traefik host.

### 2. Register Services with Consul

You have two options:

#### Option A: Using Consul Service Definitions (Recommended)

Create service definition files in `./consul-config/services/`:

```json
{
  "service": {
    "name": "my-app",
    "tags": [
      "traefik.enable=true",
      "traefik.http.routers.myapp.rule=Host(`myapp.home.ditschers.de`) || Host(`myapp.home.ditscher.me`)",
      "traefik.http.routers.myapp.entrypoints=websecure",
      "traefik.http.routers.myapp.tls=true",
      "traefik.http.routers.myapp.tls.certresolver=letsencrypt",
      "traefik.http.services.myapp.loadbalancer.server.port=8080"
    ],
    "port": 8080,
    "address": "<REMOTE_HOST_IP>",
    "check": {
      "http": "http://<REMOTE_HOST_IP>:8080/health",
      "interval": "10s"
    }
  }
}
```

Mount this in your Consul agent:

```yaml
services:
  consul-agent:
    image: consul:latest
    restart: unless-stopped
    command: agent -retry-join=<MAIN_HOST_IP> -bind='{{ GetInterfaceIP "eth0" }}'
    network_mode: host
    volumes:
      - ./consul-data:/consul/data
      - ./consul-config:/consul/config  # Add this line
    environment:
      - CONSUL_BIND_INTERFACE=eth0
```

#### Option B: Using Registrator (Automatic Docker Service Registration)

Use Registrator to automatically register Docker containers:

```yaml
services:
  consul-agent:
    image: consul:latest
    restart: unless-stopped
    command: agent -retry-join=<MAIN_HOST_IP> -bind='{{ GetInterfaceIP "eth0" }}'
    network_mode: host
    volumes:
      - ./consul-data:/consul/data
    environment:
      - CONSUL_BIND_INTERFACE=eth0

  registrator:
    image: gliderlabs/registrator:latest
    restart: unless-stopped
    command: -internal consul://localhost:8500
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock
    depends_on:
      - consul-agent
    network_mode: host
```

Then add labels to your Docker services:

```yaml
services:
  myapp:
    image: myapp:latest
    ports:
      - "8080:8080"
    environment:
      SERVICE_NAME: myapp
      SERVICE_TAGS: traefik.enable=true,traefik.http.routers.myapp.rule=Host(`myapp.home.ditschers.de`),traefik.http.routers.myapp.entrypoints=websecure,traefik.http.routers.myapp.tls=true,traefik.http.routers.myapp.tls.certresolver=letsencrypt,traefik.http.services.myapp.loadbalancer.server.port=8080
```

## Service Tags Reference

Common Traefik tags to use:

```
traefik.enable=true
traefik.http.routers.<service-name>.rule=Host(`<service>.home.ditschers.de`) || Host(`<service>.home.ditscher.me`)
traefik.http.routers.<service-name>.entrypoints=websecure
traefik.http.routers.<service-name>.tls=true
traefik.http.routers.<service-name>.tls.certresolver=letsencrypt
traefik.http.routers.<service-name>.middlewares=secure-headers@file
traefik.http.services.<service-name>.loadbalancer.server.port=<port>
traefik.http.services.<service-name>.loadbalancer.server.scheme=http
```

For TCP services (like MQTT):

```
traefik.enable=true
traefik.tcp.routers.<service-name>.rule=HostSNI(`<service>.home.ditschers.de`)
traefik.tcp.routers.<service-name>.entrypoints=mqtt-tls
traefik.tcp.routers.<service-name>.tls=true
traefik.tcp.routers.<service-name>.tls.certresolver=letsencrypt
traefik.tcp.services.<service-name>.loadbalancer.server.port=1883
```

## Verification

1. Check Consul UI to see registered services
2. Check Traefik dashboard to see discovered routes
3. Test accessing your service via the configured hostname

## Troubleshooting

### Consul Agent Not Connecting

- Check firewall rules (ports 8300, 8301, 8302, 8500, 8600)
- Verify the main host IP is correct
- Check logs: `docker compose logs consul-agent`

### Services Not Appearing in Traefik

- Verify service has `traefik.enable=true` tag
- Check Consul UI to confirm service is registered
- Review Traefik logs: `docker compose logs traefik`
- Ensure service has correct port and address

### TLS Certificate Issues

- Let's Encrypt has rate limits (5 certificates per week for same domain)
- Ensure DNS points to your Traefik host
- Check port 80 is accessible for ACME challenge

## Network Requirements

Ensure these ports are accessible between hosts:

- **8300**: Consul Server RPC
- **8301**: Consul Serf LAN
- **8500**: Consul HTTP API (if accessing remotely)
- **8600**: Consul DNS (optional)

Your application ports must also be accessible from the Traefik host.
