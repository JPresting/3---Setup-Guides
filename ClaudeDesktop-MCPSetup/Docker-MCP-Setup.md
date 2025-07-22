# Docker-Based MCP Servers Setup

**Requirements:** Docker Desktop + Container images + Environment variables

Docker-based MCP servers run in isolated containers, providing consistent environments and easy deployment. This approach is ideal for complex services that require specific runtime environments or multiple dependencies.

## Prerequisites

### Docker Desktop Installation

1. **Download Docker Desktop**: Go to [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)

2. **Install Docker Desktop**:
   - Run the installer as Administrator
   - Follow installation wizard
   - Restart computer when prompted

3. **Configure Docker Desktop**:
   - **Enable Auto-start**: Settings → General → "Start Docker Desktop when you log in"
   - **Resource Allocation**: Settings → Resources → Adjust CPU/Memory as needed
   - **WSL2 Integration** (Optional but recommended): Settings → Resources → WSL Integration

4. **Verify Installation**:
```cmd
docker --version
docker ps
```

Should show Docker version and empty container list.

## General Docker MCP Configuration Pattern

### Basic Docker Server Template

```json
"server-name": {
  "command": "docker",
  "args": [
    "run",
    "-i",
    "--rm",
    "-e", "ENV_VAR_NAME",
    "docker-image-name"
  ],
  "env": {
    "ENV_VAR_NAME": "environment-variable-value"
  }
}
```

### Configuration Components

**Command Arguments Explained**:
- `"run"`: Creates and runs a new container
- `"-i"`: Interactive mode (keeps STDIN open)
- `"--rm"`: Automatically remove container when it exits
- `"-e", "ENV_VAR_NAME"`: Expose environment variable to container
- `"docker-image-name"`: The Docker image to run

## Common Docker MCP Servers

### GitHub Integration
**Container**: Official MCP GitHub server

```json
"github": {
  "command": "docker",
  "args": [
    "run",
    "-i",
    "--rm",
    "-e", "GITHUB_PERSONAL_ACCESS_TOKEN",
    "mcp/github"
  ],
  "env": {
    "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_your_github_token_here"
  }
}
```

**Setup Steps**:
1. Create GitHub Personal Access Token:
   - Go to GitHub → Settings → Developer settings → Personal access tokens
   - Generate new token (classic)
   - Select scopes: `repo`, `read:org`, `user:email`
   - Copy token immediately

**Capabilities**: Repository browsing, file operations, issue management, pull requests

### PostgreSQL Database Access
**Container**: Custom PostgreSQL MCP server

```json
"postgresql": {
  "command": "docker",
  "args": [
    "run",
    "-i",
    "--rm",
    "-e", "POSTGRES_CONNECTION_STRING",
    "mcp/postgresql"
  ],
  "env": {
    "POSTGRES_CONNECTION_STRING": "postgresql://username:password@host:5432/database"
  }
}
```

**Setup Requirements**:
- PostgreSQL database accessible from Docker container
- Database connection string with proper credentials
- Network access configured if database is remote

### Redis Cache Integration
**Container**: Redis MCP server for cache operations

```json
"redis": {
  "command": "docker",
  "args": [
    "run",
    "-i",
    "--rm",
    "-e", "REDIS_URL",
    "mcp/redis"
  ],
  "env": {
    "REDIS_URL": "redis://localhost:6379"
  }
}
```

**Capabilities**: Cache operations, data storage, session management

## Advanced Docker Configurations

### Volume Mounts for File Access

For servers that need file system access:

```json
"file-processor": {
  "command": "docker",
  "args": [
    "run",
    "-i",
    "--rm",
    "-v", "C:\\Users\\YOUR_USERNAME\\Documents:/workspace",
    "-e", "WORKSPACE_PATH=/workspace",
    "mcp/file-processor"
  ],
  "env": {
    "WORKSPACE_PATH": "/workspace",
    "API_KEY": "your-api-key"
  }
}
```

**Volume Mount Explained**:
- `-v "C:\\Users\\YOUR_USERNAME\\Documents:/workspace"`: Mounts Windows folder to container path
- Container can access files in your Documents folder via `/workspace`

### Network Configuration

For services that need network access:

```json
"network-service": {
  "command": "docker",
  "args": [
    "run",
    "-i",
    "--rm",
    "--network", "host",
    "-e", "SERVICE_URL",
    "mcp/network-service"
  ],
  "env": {
    "SERVICE_URL": "https://api.service.com"
  }
}
```

**Network Options**:
- `--network host`: Use host networking (shares host's network)
- `--network bridge`: Use Docker's default bridge network
- Custom networks: Create with `docker network create`

### Port Mapping

For services that expose ports:

```json
"web-service": {
  "command": "docker",
  "args": [
    "run",
    "-i",
    "--rm",
    "-p", "8080:80",
    "-e", "API_KEY",
    "mcp/web-service"
  ],
  "env": {
    "API_KEY": "your-api-key"
  }
}
```

**Port Mapping**: `-p "8080:80"` maps host port 8080 to container port 80

## Docker Management

### Image Management

**List available images**:
```cmd
docker images
```

**Pull specific image**:
```cmd
docker pull mcp/github
```

**Remove unused images**:
```cmd
docker image prune
```

### Container Management

**List running containers**:
```cmd
docker ps
```

**Stop container**:
```cmd
docker stop container-name
```

**View container logs**:
```cmd
docker logs container-name
```

### Cleanup

**Remove all stopped containers**:
```cmd
docker container prune
```

**Remove unused volumes**:
```cmd
docker volume prune
```

**Complete cleanup** (use carefully):
```cmd
docker system prune -a
```

## Troubleshooting Docker MCP Servers

### Common Issues

**"Docker daemon not running"**:
- Start Docker Desktop application
- Check if Docker service is running in Services (Windows)
- Restart Docker Desktop if needed

**"Image not found" errors**:
- **Pull image manually**: `docker pull image-name`
- **Check image name**: Verify spelling and registry
- **Network issues**: Ensure internet connectivity for image downloads

**"Permission denied" or volume mount failures**:
- **Windows paths**: Use forward slashes in volume mounts: `C:/Users/...`
- **Drive sharing**: Enable drive sharing in Docker Desktop settings
- **File permissions**: Ensure files are accessible to Docker

**Container exits immediately**:
- **Check logs**: `docker logs container-name`
- **Interactive debugging**: Add `-it` flags and run bash: `docker run -it image-name /bin/bash`
- **Environment variables**: Verify all required env vars are set

**Network connectivity issues**:
- **Host networking**: Try `--network host` for local services
- **DNS issues**: Check container can resolve external hostnames
- **Firewall**: Ensure Windows firewall allows Docker

### Performance Optimization

**Resource Allocation**:
- **Increase memory**: Docker Desktop → Settings → Resources → Memory
- **CPU allocation**: Adjust CPU cores available to Docker
- **Disk space**: Monitor Docker disk usage

**Image Optimization**:
- **Use specific tags**: Avoid `latest` tag for reproducibility
- **Clean up regularly**: Remove unused images and containers
- **Multi-stage builds**: For custom images, use multi-stage builds

## Security Considerations

### Container Security

1. **Use official images**: Prefer official MCP server images when available
2. **Regular updates**: Keep Docker Desktop and images updated
3. **Minimal permissions**: Don't run containers as root unless necessary
4. **Network isolation**: Use appropriate network configurations
5. **Secret management**: Use Docker secrets for sensitive data (advanced)

### Environment Variables

1. **Secure storage**: Keep API keys and tokens secure
2. **Local only**: Environment variables are visible to container processes
3. **Rotation**: Regularly rotate API keys and tokens
4. **Logging**: Be careful not to log sensitive environment variables

## Custom Docker Images

### Creating Custom MCP Server Images

For advanced users who want to create custom MCP servers:

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

**Build custom image**:
```cmd
docker build -t my-mcp-server .
```

**Use in configuration**:
```json
"custom-server": {
  "command": "docker",
  "args": ["run", "-i", "--rm", "my-mcp-server"],
  "env": {}
}
```

## Docker vs NPX Servers

### When to Use Docker

**Advantages**:
- **Isolated environment**: No conflicts with system dependencies
- **Consistent deployment**: Same environment across different machines
- **Complex dependencies**: Better for servers with many requirements
- **Version control**: Specific image tags ensure reproducible deployments

**Disadvantages**:
- **Resource overhead**: Higher memory and disk usage
- **Slower startup**: Container initialization takes time
- **Complexity**: More complex setup and troubleshooting

### When to Use NPX

**Advantages**:
- **Lighter weight**: Lower resource usage
- **Faster startup**: Direct execution without containerization
- **Simpler troubleshooting**: Direct access to logs and processes

**Disadvantages**:
- **Dependency conflicts**: Potential conflicts with system packages
- **Environment variations**: Different behavior across machines
- **Windows issues**: APPDATA and PATH complications

## Finding Docker MCP Images

### Official Sources
- **Docker Hub**: [hub.docker.com](https://hub.docker.com) - search "mcp"
- **GitHub Container Registry**: Check MCP server repositories
- **Model Context Protocol Registry**: [modelcontextprotocol.io](https://modelcontextprotocol.io)

### Verification
Before using Docker images:
1. **Check source**: Verify image publisher and repository
2. **Review Dockerfile**: Examine image build process if available  
3. **Scan for vulnerabilities**: Use `docker scan image-name`
4. **Test safely**: Try with minimal permissions first
5. **Community feedback**: Look for user reviews and experiences

Docker-based MCP servers provide powerful capabilities with isolated, reproducible environments, making them ideal for complex integrations and enterprise deployments.