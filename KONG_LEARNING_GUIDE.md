# Kong API Gateway Learning Guide

## Table of Contents
- [What is Kong API Gateway?](#what-is-kong-api-gateway)
- [Core Concepts](#core-concepts)
- [Your Current Setup](#your-current-setup)
- [Architecture Overview](#architecture-overview)
- [Working Examples](#working-examples)
- [Security Considerations](#security-considerations)
- [Next Steps](#next-steps)
- [Troubleshooting](#troubleshooting)

## What is Kong API Gateway?

Kong API Gateway is a cloud-native, fast, scalable, and distributed microservice proxy. It acts as a middleware layer between your clients and your backend services, providing:

- **Traffic Management**: Route requests to appropriate services
- **Authentication & Authorization**: Secure your APIs
- **Rate Limiting**: Control API usage
- **Monitoring & Analytics**: Track API performance
- **Plugin Ecosystem**: Extend functionality with plugins

## Core Concepts

### 1. **Control Plane vs Data Plane**
- **Control Plane**: Configuration management (Kong Manager/Konnect)
- **Data Plane**: Request processing (Kong Gateway instances)

### 2. **Key Components**

#### **Services**
A service is an abstraction of an upstream application/microservice.

```yaml
Service Example:
  Name: anthropic
  Host: api.anthropic.com
  Port: 443
  Protocol: https
  Path: /v1/messages
```

#### **Routes**
Routes define how requests are sent to services based on HTTP attributes.

```yaml
Route Example:
  Name: anthropic
  Paths: ["/api/v1/anthropic"]
  Methods: ["POST", "PUT"]
  Service: anthropic
```

#### **Plugins**
Plugins extend Kong's functionality (authentication, rate limiting, etc.).

```yaml
Plugin Example:
  Name: ai-proxy-advanced
  Config:
    provider: anthropic
    auth:
      header_name: x-api-key
      header_value: sk-ant-...
```

#### **Consumers**
Entities that use your APIs (users, applications, etc.).

## Your Current Setup

### Architecture Overview
```
Client Request
      ↓
Kong Gateway (kong-a92568d0c1us8iazk.kongcloud.dev)
      ↓
Routes:
├── /chat/completions → OpenAI Service → api.openai.com/chat/completions
└── /api/v1/anthropic → Anthropic Service → api.anthropic.com/v1/messages
```

### Current Services & Routes

#### 1. OpenAI Configuration
- **Service**: `AIManagerModelService_1758636499840`
- **Route**: `/chat/completions`
- **Upstream**: `api.openai.com/chat/completions`
- **Plugin**: `ai-proxy-advanced` with multiple targets

#### 2. Anthropic Configuration
- **Service**: `anthropic`
- **Route**: `/api/v1/anthropic`  
- **Upstream**: `api.anthropic.com/v1/messages`
- **Plugin**: None (manual auth required)

### Request Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        Client                               │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                Kong Gateway                                 │
│  (kong-a92568d0c1us8iazk.kongcloud.dev)                   │
├─────────────────────┬───────────────────────────────────────┤
│                     │                                       │
│  Route Matching     │   Plugin Processing                   │
│                     │                                       │
│  /chat/completions ─┼─→ ai-proxy-advanced                   │
│  /api/v1/anthropic ─┼─→ (no plugin)                        │
│                     │                                       │
└─────────────────────┼───────────────────────────────────────┘
                      │
                      ▼
           ┌──────────────────────┐
           │   Service Routing    │
           └──────────┬───────────┘
                      │
         ┌────────────┴─────────────┐
         ▼                          ▼
┌─────────────────┐      ┌─────────────────┐
│   OpenAI API    │      │  Anthropic API  │
│                 │      │                 │
│ api.openai.com  │      │api.anthropic.com│
│/chat/completions│      │  /v1/messages   │
└─────────────────┘      └─────────────────┘
```

## Working Examples

### 1. OpenAI Request
```bash
curl -X POST https://kong-a92568d0c1us8iazk.kongcloud.dev/chat/completions \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer sk-proj-YOUR_OPENAI_KEY' \
  -d '{
    "messages": [
      {
        "role": "user",
        "content": "Hello, how are you?"
      }
    ],
    "model": "gpt-4"
  }'
```

**Response**: ✅ Success (with OpenAI response)

### 2. Anthropic Request
```bash
curl -X POST https://kong-a92568d0c1us8iazk.kongcloud.dev/api/v1/anthropic \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: sk-ant-YOUR_ANTHROPIC_KEY' \
  -H 'anthropic-version: 2023-06-01' \
  -d '{
    "messages": [
      {
        "role": "user",
        "content": "Hello, how are you?"
      }
    ],
    "model": "claude-3-haiku-20240307",
    "max_tokens": 1024
  }'
```

**Response**: ✅ Success (with Claude response)

## Security Considerations

### Current Security Posture
- ❌ **No Kong Authentication**: Anyone can access your gateway
- ❌ **Exposed API Keys**: Clients must provide upstream API keys
- ❌ **No Rate Limiting**: Unlimited requests allowed
- ❌ **No IP Whitelisting**: Open to all IPs

### Recommended Security Improvements

#### 1. Add Kong Authentication
```yaml
# Example: API Key Authentication
plugins:
  - name: key-auth
    config:
      key_names: ["apikey"]
```

#### 2. Configure Rate Limiting
```yaml
plugins:
  - name: rate-limiting
    config:
      minute: 100
      hour: 1000
```

#### 3. Proper AI Proxy Configuration
```yaml
# Configure ai-proxy-advanced to handle auth automatically
plugins:
  - name: ai-proxy-advanced
    config:
      targets:
        - auth:
            allow_override: false  # Don't allow client override
            header_name: "x-api-key"
            header_value: "YOUR_ANTHROPIC_KEY"
```

## Plugin Architecture Deep Dive

### Your Current Plugin Configuration

#### AI Proxy Advanced Plugin
```yaml
Plugin ID: cc9cde13-047f-4f12-9029-1b428c2f359d
Service: 28b1ed19-d052-46c6-84e7-5748b360f61a (OpenAI Service)
Config:
  targets:
    1. OpenAI Target:
       - provider: openai
       - auth: null (requires client to provide auth)
    
    2. Anthropic Target:
       - provider: anthropic
       - auth: 
           header_name: x-api-key
           header_value: sk-ant-...
       - options:
           max_tokens: 1024
           anthropic_version: /v1/messages
```

### Plugin Execution Flow
```
Request → Route Match → Service → Plugin Chain → Upstream
                                     ↓
                          ┌─────────────────────────┐
                          │   Plugin Processing     │
                          │                         │
                          │ 1. Authentication       │
                          │ 2. Rate Limiting        │
                          │ 3. AI Proxy Advanced    │
                          │ 4. Caching              │
                          │ 5. Logging              │
                          └─────────────────────────┘
```

## Next Steps

### 1. **Immediate Improvements**
- [ ] Add Kong authentication (API keys or JWT)
- [ ] Configure rate limiting
- [ ] Set up proper upstream authentication in plugins
- [ ] Add request/response logging

### 2. **Advanced Features**
- [ ] Implement caching for AI responses
- [ ] Set up load balancing between multiple AI providers
- [ ] Configure monitoring and alerting
- [ ] Add request transformation plugins

### 3. **Learning Path**
1. **Basics**: Understand Services, Routes, and Plugins
2. **Security**: Implement authentication and authorization
3. **Performance**: Add caching and rate limiting
4. **Monitoring**: Set up logging and analytics
5. **Advanced**: Custom plugins and complex routing

## Troubleshooting

### Common Issues

#### 1. **404 Not Found**
- **Cause**: Route not matching request path/method
- **Fix**: Check route configuration and path matching

#### 2. **503 Service Unavailable**
- **Cause**: Upstream service unreachable
- **Fix**: Verify service host/port configuration

#### 3. **Authentication Errors**
- **Cause**: Missing or invalid API keys
- **Fix**: Check plugin configuration and auth headers

#### 4. **Plugin Not Working**
- **Cause**: Plugin not applied to correct service/route
- **Fix**: Verify plugin scoping configuration

### Debug Commands

#### Check Configuration
```bash
# List all services
kong-admin-api GET /services

# List all routes  
kong-admin-api GET /routes

# List all plugins
kong-admin-api GET /plugins

# Check specific service
kong-admin-api GET /services/{service-id}
```

#### Monitor Requests
```bash
# Real-time logs
tail -f /var/log/kong/access.log

# Check plugin execution
tail -f /var/log/kong/error.log
```

## Kong vs Other API Gateways

| Feature | Kong | AWS API Gateway | Azure API Management |
|---------|------|-----------------|---------------------|
| **Deployment** | Self-hosted/Cloud | Cloud-only | Cloud/Hybrid |
| **Performance** | Very High | High | Medium |
| **Plugin Ecosystem** | 100+ plugins | Limited | Good |
| **Cost** | Free/Paid tiers | Pay-per-request | Subscription |
| **Learning Curve** | Moderate | Easy | Complex |

## Resources

### Official Documentation
- [Kong Documentation](https://docs.konghq.com/)
- [Kong Plugin Hub](https://docs.konghq.com/hub/)
- [Kong Admin API](https://docs.konghq.com/gateway/latest/admin-api/)

### Community
- [Kong Community Forum](https://discuss.konghq.com/)
- [Kong GitHub](https://github.com/Kong/kong)
- [Kong YouTube Channel](https://www.youtube.com/c/KongInc)

### Learning Labs
- [Kong Academy](https://education.konghq.com/)
- [Hands-on Labs](https://github.com/Kong/kong-demos)

---

## Quick Reference Card

### Essential Commands
```bash
# Health check
curl -i http://kong:8001/status

# Add service
curl -X POST http://kong:8001/services \
  --data name=my-service \
  --data url=http://example.com

# Add route
curl -X POST http://kong:8001/services/my-service/routes \
  --data paths[]=/my-path

# Add plugin
curl -X POST http://kong:8001/services/my-service/plugins \
  --data name=rate-limiting \
  --data config.minute=100
```

### Your Gateway URLs
- **OpenAI**: `https://kong-a92568d0c1us8iazk.kongcloud.dev/chat/completions`
- **Anthropic**: `https://kong-a92568d0c1us8iazk.kongcloud.dev/api/v1/anthropic`

### Authentication Headers
- **OpenAI**: `Authorization: Bearer sk-proj-...`
- **Anthropic**: `x-api-key: sk-ant-...` + `anthropic-version: 2023-06-01`