# Gonka MLNode Public Image

This Docker image is a security-enhanced wrapper around the `product-science/mlnode` gonka MLNode image, designed specifically for rental GPU services used in GNK mining.

## Problem Statement

When deploying MLNode instances on rental GPU services (Spheron.ai, Vast.ai, RunPod, Hyperstac, TensorDock, etc.), users typically need to open ports 5000 and 8000 for POC (Proof of Concept) and inference services. This creates a critical security vulnerability:

- **Direct Access**: MLNode becomes directly accessible from the internet, bypassing the network node proxy
- **Free Inference**: Users can make inference requests directly without going through the network node, resulting in revenue loss
- **DoS Vulnerability**: The exposed ports are vulnerable to Denial of Service attacks

## Solution

This image integrates an **FRP (Fast Reverse Proxy) client** that automatically establishes a secure tunnel to an FRP server running on the network node. This ensures:

- ✅ All traffic is routed through the network node proxy
- ✅ No direct access to MLNode services from the internet
- ✅ Proper authentication and rate limiting through the network node
- ✅ Protection against direct DoS attacks

## Architecture

```txt
Internet → Network Node (FRP Server) → FRP Tunnel → MLNode Container (FRP Client) → MLNode Services
```

The FRP client creates reverse tunnels for:

- **Inference port**: Exposed as `1{CLIENT_ID}` on the FRP server (maps to local port 8081)
- **POC port**: Exposed as `2{CLIENT_ID}` on the FRP server (maps to local port 5050)

## Requirements

### Environment Variables

#### Required Variables

- **`API_NODE`**: API node address in format `host:port` (e.g., `api.example.com:8080` or `192.168.1.1:8080`)
- **`FRP_SERVER`**: FRP server address in format `host:port` (e.g., `frps.example.com:7000` or `192.168.1.1:7000`)
- **`SECRET_FRP_TOKEN`**: Authentication token for FRP server connection
- **`CLIENT_ID`**: Four-digit client identifier (0001-9999), used to generate unique remote ports

#### Optional Variables

- **`NODE_ID`**: Node identifier (defaults to `CLIENT_ID` if not set)
- **`TENSOR_PARALLEL_SIZE`**: Tensor parallel size (1-4, default: 1)
- **`MODEL_NAME`**: HuggingFace model name (default: `Qwen/Qwen3-32B-FP8`)
- **`FRP_CONFIG_DIR`**: FRP configuration directory (default: `/etc/frp`)
- **`HF_HOME`**: HuggingFace cache directory (default: `/data/hf-cache`)
- **`HOST_UID`**: Host user ID for container user (default: 1000)
- **`HOST_GID`**: Host group ID for container group (default: 1001)
- **`LOG_DIR`**: Log directory for test servers (default: `/tmp/logs`)
- **`UBUNTU_TEST`**: Set to `"true"` to run test HTTP servers instead of actual MLNode (for testing)

## Port Mapping

The `CLIENT_ID` variable determines the remote ports exposed on the FRP server:

- **Inference port**: `1{CLIENT_ID}` (e.g., if `CLIENT_ID=0001`, remote port is `10001`)
- **POC port**: `2{CLIENT_ID}` (e.g., if `CLIENT_ID=0001`, remote port is `20001`)

These ports are mapped to local container ports:

- Local inference: `8081` → Remote: `1{CLIENT_ID}`
- Local POC: `5050` → Remote: `2{CLIENT_ID}`

## Usage

### Basic Example

```bash
docker run -d \
  -e API_NODE=api.example.com:8080 \
  -e FRP_SERVER=frps.example.com:7000 \
  -e SECRET_FRP_TOKEN=your-secret-token \
  -e CLIENT_ID=0001 \
  -e MODEL_NAME=Qwen/Qwen3-32B-FP8 \
  -e TENSOR_PARALLEL_SIZE=1 \
  ghcr.io/your-org/gonka-mlnode-public:latest
```

### With Custom Configuration

```bash
docker run -d \
  -e API_NODE=192.168.1.100:8080 \
  -e FRP_SERVER=frps.internal:7000 \
  -e SECRET_FRP_TOKEN=secure-token-123 \
  -e CLIENT_ID=0042 \
  -e NODE_ID=node-42 \
  -e MODEL_NAME=Qwen/Qwen3-32B-FP8 \
  -e TENSOR_PARALLEL_SIZE=2 \
  -e HF_HOME=/data/models \
  -v /host/models:/data/models \
  ghcr.io/your-org/gonka-mlnode-public:latest
```

### Testing Mode

For testing without actual MLNode services:

```bash
docker run -d \
  -e API_NODE=api.example.com:8080 \
  -e FRP_SERVER=frps.example.com:7000 \
  -e SECRET_FRP_TOKEN=test-token \
  -e CLIENT_ID=0001 \
  -e UBUNTU_TEST=true \
  ghcr.io/your-org/gonka-mlnode-public:latest
```

## How It Works

1. **Validation**: The startup script validates all required environment variables and their formats
2. **FRP Configuration**: Creates an FRP client configuration file with:
   - Connection to the specified FRP server
   - Reverse tunnel mappings for inference and POC ports
   - Authentication using the provided token
3. **FRP Client**: Starts the FRP client process to establish the tunnel
4. **Nginx Proxy**: Starts Nginx to proxy requests to the MLNode services
5. **MLNode Services**: Starts the actual MLNode application (or test servers if in test mode)

## Network Flow

```txt
External Request
    ↓
Network Node (FRP Server) on port 1{CLIENT_ID} or 2{CLIENT_ID}
    ↓
FRP Tunnel
    ↓
Container: Nginx on port 8081 (inference) or 5050 (POC)
    ↓
Container: MLNode on port 8080 (inference) or 5000 (POC)
```

## Security Features

- **No Direct Exposure**: MLNode services are never directly exposed to the internet
- **Token Authentication**: FRP connection requires a secret token
- **Network Isolation**: All traffic must go through the network node

## Troubleshooting

### Check FRP Connection

```bash
docker logs <container-id> | grep frpc
```

### Verify Port Mapping

The logs will show:

```sh
Writing /etc/frp/frpc.ini for server <host>:<port>...
Starting frpc with config /etc/frp/frpc.ini...
```

### Test HTTP Servers (if UBUNTU_TEST=true)

```bash
docker exec <container-id> tail -f /tmp/logs/http_server_*.log
```

## Building

This image is built on top of a base image. To build:

```bash
# Build base image first
docker build -f Dockerfile.base -t ghcr.io/your-org/gonka-mlnode-public-base:latest .

# Build main image
docker build -t ghcr.io/your-org/gonka-mlnode-public:latest .
```

## License

This project is licensed under the MIT License.

## Support

For support, questions, or issues, please open a GitHub issue in this repository.
