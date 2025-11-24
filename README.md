# Traefik Proxmox Provider

![Traefik Proxmox Provider](https://raw.githubusercontent.com/nx211/traefik-proxmox-provider/main/.assets/logo.png)

A Traefik provider that automatically configures routing based on Proxmox VE virtual machines and containers.

## Features

- Automatically discovers Proxmox VE virtual machines and containers
- Configures routing based on VM/container metadata
- Supports HTTP, HTTPS, TCP, and UDP endpoints
- Configurable polling interval
- SSL validation options
- Logging configuration
- Full support for Traefik's routing, middleware, and TLS options

## Installation

1. Add the plugin to your Traefik configuration:

```yaml
experimental:
  plugins:
    traefik-proxmox-provider:
      moduleName: github.com/NX211/traefik-proxmox-provider
      version: v0.7.0
```

2. Configure the provider in your dynamic configuration:

```yaml
# Dynamic configuration
providers:
  plugin:
    traefik-proxmox-provider:
      pollInterval: "30s"
      apiEndpoint: "https://proxmox.example.com"
      apiTokenId: "root@pam!traefik_prod"
      apiToken: "your-api-token"
      apiLogging: "info"
      apiValidateSSL: "true"
```

## Configuration

### Provider Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `pollInterval` | `string` | `"30s"` | How often to poll the Proxmox API for changes |
| `apiEndpoint` | `string` | - | The URL of your Proxmox VE API |
| `apiTokenId` | `string` | - | The API token ID (e.g., "root@pam!traefik_prod") |
| `apiToken` | `string` | - | The API token secret |
| `apiLogging` | `string` | `"info"` | Log level for API operations ("debug" or "info") |
| `apiValidateSSL` | `string` | `"true"` | Whether to validate SSL certificates |

## Proxmox API Token Setup

The Traefik Proxmox Provider needs an API token with specific permissions to read VM and container information. Here's how to set up the proper token and permissions:

```bash
# Create a role for Traefik provider with minimum required permissions
pveum role add traefik-provider -privs "VM.Audit,VM.Monitor,Sys.Audit,Datastore.Audit"

# Create an API token for your user (replace with your actual username)
pveum user token add root@pam traefik_prod

# Assign the role to the token for all paths
pveum acl modify / -token 'root@pam!traefik_prod' -role traefik-provider
```

Make sure to save the API token value when it's displayed, as it won't be shown again.

## Usage

1. Create an API token in Proxmox VE as described above

2. Configure the provider in your Traefik configuration:
   - Set the `apiEndpoint` to your Proxmox VE server URL
   - Set the `apiTokenId` and `apiToken` from step 1
   - Adjust other options as needed

3. **Very Important**: Add Traefik labels to your VMs/containers:
   - Edit your VM/container in Proxmox VE
   - Go to the "Summary" page and edit the "Notes" box by clicking the pencil icon
   - Add one Traefik label per line with the format `traefik.key=value`
   - **At minimum** add `traefik.enable=true` to enable Traefik for this VM/container

4. Restart Traefik to load the new configuration

## VM/Container Labeling

The provider looks for Traefik labels in the VM/container notes field. Each line in the Notes field starting with `traefik.` will be treated as a Traefik label.

### Required Labels

- `traefik.enable=true` - Without this label, the VM/container will be ignored

### Common Labels

#### HTTP Services
- `traefik.http.routers.<name>.rule=Host(`myapp.example.com`)` - The router rule for this service
- `traefik.http.services.<name>.loadbalancer.server.port=8080` - The port to route traffic to (defaults to 80)

#### TCP Services
- `traefik.tcp.routers.<name>.rule=HostSNI(`myapp.example.com`)` - The TCP router rule for this service
- `traefik.tcp.services.<name>.loadbalancer.server.port=443` - The port to route TCP traffic to (defaults to 443)

#### UDP Services
- `traefik.udp.routers.<name>.service=<service-name>` - Links UDP router to a service
- `traefik.udp.services.<name>.loadbalancer.server.port=53` - The port to route UDP traffic to (defaults to 53)

### Advanced Label Examples

#### Named Routers and Services

```
traefik.enable=true
traefik.http.routers.myapp.rule=Host(`myapp.example.com`)
traefik.http.services.appservice.loadbalancer.server.port=8080
traefik.http.routers.myapp.service=appservice
```

#### EntryPoints

```
traefik.http.routers.myapp.entrypoints=websecure
```

#### Middlewares

```
traefik.http.routers.myapp.middlewares=compression,auth@file
```

#### TLS Configuration

```
traefik.http.routers.myapp.tls=true
traefik.http.routers.myapp.tls.certresolver=myresolver
traefik.http.routers.myapp.tls.domains=example.com
traefik.http.routers.myapp.tls.options=tlsoptions@file
```

#### Health Checks

```
traefik.http.services.myservice.loadbalancer.healthcheck.path=/health
traefik.http.services.myservice.loadbalancer.healthcheck.interval=10s
traefik.http.services.myservice.loadbalancer.healthcheck.timeout=5s
```

#### Sticky Sessions

```
traefik.http.services.myservice.loadbalancer.sticky.cookie.name=session
traefik.http.services.myservice.loadbalancer.sticky.cookie.secure=true
traefik.http.services.myservice.loadbalancer.sticky.cookie.httponly=true
```

#### HTTPS Backend Services

```
traefik.http.services.myservice.loadbalancer.server.scheme=https
```

#### TCP-specific Options

```
traefik.tcp.services.myservice.loadbalancer.terminationdelay=100
traefik.tcp.services.myservice.loadbalancer.proxyprotocol.version=2
traefik.tcp.routers.myrouter.middlewares=ipwhitelist@file
```

#### Advanced Mixed Protocol Setup

```
traefik.enable=true
# HTTP service
traefik.http.routers.web.rule=Host(`app.example.com`)
traefik.http.routers.web.entrypoints=websecure
traefik.http.routers.web.tls=true
traefik.http.services.web.loadbalancer.server.port=8080
# TCP service for database
traefik.tcp.routers.db.rule=HostSNI(`app.example.com`)
traefik.tcp.routers.db.entrypoints=database
traefik.tcp.routers.db.tls.passthrough=true
traefik.tcp.services.db.loadbalancer.server.port=5432
# UDP service for monitoring
traefik.udp.routers.metrics.entrypoints=metrics
traefik.udp.services.metrics.loadbalancer.server.port=8125
```

### Full Example of VM/Container Notes

```
My application server
Some notes about this server

traefik.enable=true
traefik.http.routers.myapp.rule=Host(`myapp.example.com`)
traefik.http.routers.myapp.entrypoints=websecure
traefik.http.routers.myapp.middlewares=auth@file,compression
traefik.http.routers.myapp.tls=true
traefik.http.routers.myapp.tls.certresolver=myresolver
traefik.http.services.myapp.loadbalancer.server.port=8080
traefik.http.services.myapp.loadbalancer.healthcheck.path=/health
```

## How It Works

1. The provider connects to your Proxmox VE cluster via API
2. It discovers all running VMs and containers on all nodes
3. For each VM/container, it reads the notes field looking for Traefik labels
4. If `traefik.enable=true` is found, it analyzes the labels to determine which protocols to configure
5. Based on label prefixes (`traefik.http.*`, `traefik.tcp.*`, `traefik.udp.*`), it creates appropriate routers and services
6. The provider attempts to get IP addresses for the VM/container
7. For HTTP/HTTPS: IPs are used as server URLs (e.g., `http://192.168.1.10:8080`)
8. For TCP/UDP: IPs are used as server addresses (e.g., `192.168.1.10:5432`)
9. If no IPs are found, hostnames are used as fallback
10. This process repeats according to the configured poll interval

## Examples

### Basic Configuration

```yaml
providers:
  plugin:
    traefik-proxmox-provider:
      pollInterval: "30s"
      apiEndpoint: "https://proxmox.example.com"
      apiTokenId: "root@pam!traefik_prod"
      apiToken: "your-api-token"
      apiLogging: "debug"  # Use debug for troubleshooting
      apiValidateSSL: "true"
```

### VM/Container Label Examples

#### HTTP Examples

Simple web server:
```
traefik.enable=true
traefik.http.routers.app.rule=Host(`myapp.example.com`)
```

Secure website with HTTPS:
```
traefik.enable=true
traefik.http.routers.secure.rule=Host(`secure.example.com`)
traefik.http.routers.secure.entrypoints=websecure
traefik.http.routers.secure.tls=true
traefik.http.routers.secure.tls.certresolver=dnschallenge
```

API with authentication and rate limiting:
```
traefik.enable=true
traefik.http.routers.api.rule=Host(`api.example.com`)
traefik.http.routers.api.middlewares=auth@file,ratelimit@file
traefik.http.services.api.loadbalancer.server.port=3000
```

Multiple hosts with path-based routing:
```
traefik.enable=true
traefik.http.routers.multi.rule=Host(`example.com`,`www.example.com`) && PathPrefix(`/api`)
traefik.http.routers.multi.priority=100
```

#### TCP Examples

TCP service with TLS passthrough:
```
traefik.enable=true
traefik.tcp.routers.db.rule=HostSNI(`db.example.com`)
traefik.tcp.routers.db.entrypoints=db-tcp
traefik.tcp.routers.db.tls.passthrough=true
traefik.tcp.services.db.loadbalancer.server.port=5432
```

HTTPS with TLS termination at Traefik:
```
traefik.enable=true
traefik.tcp.routers.https.rule=HostSNI(`app.example.com`)
traefik.tcp.routers.https.entrypoints=websecure
traefik.tcp.routers.https.tls=true
traefik.tcp.routers.https.tls.certresolver=dnschallenge
traefik.tcp.services.https.loadbalancer.server.port=8443
```

SSH service:
```
traefik.enable=true
traefik.tcp.routers.ssh.rule=HostSNI(`*`)
traefik.tcp.routers.ssh.entrypoints=ssh
traefik.tcp.services.ssh.loadbalancer.server.port=22
```

#### UDP Examples

DNS server:
```
traefik.enable=true
traefik.udp.routers.dns.entrypoints=dns
traefik.udp.routers.dns.service=dns-service
traefik.udp.services.dns-service.loadbalancer.server.port=53
```

Game server:
```
traefik.enable=true
traefik.udp.routers.game.entrypoints=game-udp
traefik.udp.routers.game.service=game-service
traefik.udp.services.game-service.loadbalancer.server.port=7777
```

DHCP server:
```
traefik.enable=true
traefik.udp.routers.dhcp.entrypoints=dhcp
traefik.udp.routers.dhcp.service=dhcp-service
traefik.udp.services.dhcp-service.loadbalancer.server.port=67
```

## Troubleshooting

If your services aren't being discovered:

1. Enable debug logging by setting `apiLogging: "debug"`
2. Check that VMs/containers have `traefik.enable=true` in their notes field
3. Verify that VMs/containers are in the "running" state
4. Check that the provider can successfully connect to your Proxmox API
5. Verify the API token has sufficient permissions
6. Check the Traefik logs for any errors related to entrypoints or middleware references

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
