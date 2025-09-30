# Kong API Gateway Hands-On Tutorial

This tutorial uses your actual Kong setup to teach you Kong concepts step by step.

## Prerequisites

- Access to your Kong Gateway: `https://kong-a92568d0c1us8iazk.kongcloud.dev`
- API keys (OpenAI and Anthropic)
- Terminal access
- curl or HTTP client

## Lab 1: Understanding Your Current Setup

### Step 1: Test Your Current Endpoints

```bash
# Test OpenAI endpoint
curl -X POST https://kong-a92568d0c1us8iazk.kongcloud.dev/chat/completions \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_OPENAI_KEY' \
  -d '{
    "messages": [{"role": "user", "content": "Hello from Kong!"}],
    "model": "gpt-4"
  }'

# Test Anthropic endpoint
curl -X POST https://kong-a92568d0c1us8iazk.kongcloud.dev/api/v1/anthropic \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_ANTHROPIC_KEY' \
  -H 'anthropic-version: 2023-06-01' \
  -d '{
    "messages": [{"role": "user", "content": "Hello from Kong!"}],
    "model": "claude-3-haiku-20240307",
    "max_tokens": 100
  }'
```

**Exercise**: Try both requests and observe the differences in authentication headers.

### Step 2: Understand Route Matching

```bash
# This works (matches route)
curl -I https://kong-a92568d0c1us8iazk.kongcloud.dev/chat/completions

# This also works (matches route)  
curl -I https://kong-a92568d0c1us8iazk.kongcloud.dev/api/v1/anthropic

# This fails (no route)
curl -I https://kong-a92568d0c1us8iazk.kongcloud.dev/nonexistent
```

**Exercise**: Try different paths and observe Kong's routing behavior.

## Lab 2: Exploring Kong's Core Concepts

### Services in Your Setup

Your Kong has these services:

1. **OpenAI Service** (`28b1ed19-d052-46c6-84e7-5748b360f61a`)
   - Host: `api.openai.com`
   - Port: `443` 
   - Protocol: `https`
   - Path: `/chat/completions`

2. **Anthropic Service** (`1b8f7fc0-302e-4b66-a24f-3db30d7099e4`)
   - Host: `api.anthropic.com`
   - Port: `443`
   - Protocol: `https` 
   - Path: `/v1/messages`

### Routes in Your Setup

1. **OpenAI Route** 
   - Path: `/chat/completions`
   - Methods: `POST`, `GET`, `PUT`
   - Points to: OpenAI Service

2. **Anthropic Route**
   - Path: `/api/v1/anthropic`
   - Methods: `POST`, `PUT`
   - Points to: Anthropic Service

### Exercise: Route Matching Logic

```bash
# Test different HTTP methods on OpenAI route
curl -X GET https://kong-a92568d0c1us8iazk.kongcloud.dev/chat/completions
curl -X POST https://kong-a92568d0c1us8iazk.kongcloud.dev/chat/completions
curl -X DELETE https://kong-a92568d0c1us8iazk.kongcloud.dev/chat/completions  # Should fail

# Test different HTTP methods on Anthropic route  
curl -X POST https://kong-a92568d0c1us8iazk.kongcloud.dev/api/v1/anthropic
curl -X PUT https://kong-a92568d0c1us8iazk.kongcloud.dev/api/v1/anthropic
curl -X GET https://kong-a92568d0c1us8iazk.kongcloud.dev/api/v1/anthropic   # Should fail
```

## Lab 3: Plugin Behavior Analysis

### Current Plugin Configuration

Your OpenAI service has these plugins:
- `ai-proxy-advanced`: Handles AI model routing
- `ai-semantic-cache`: Caches AI responses

Your Anthropic service has:
- No plugins (direct passthrough)

### Exercise: Observe Plugin Effects

```bash
# Make the same request twice to OpenAI (should show caching)
curl -X POST https://kong-a92568d0c1us8iazk.kongcloud.dev/chat/completions \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_OPENAI_KEY' \
  -d '{
    "messages": [{"role": "user", "content": "What is Kong API Gateway?"}],
    "model": "gpt-4"
  }' \
  -w "Time: %{time_total}s\n"

# Run the same request again immediately
curl -X POST https://kong-a92568d0c1us8iazk.kongcloud.dev/chat/completions \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_OPENAI_KEY' \
  -d '{
    "messages": [{"role": "user", "content": "What is Kong API Gateway?"}],
    "model": "gpt-4"
  }' \
  -w "Time: %{time_total}s\n"
```

**Expected**: The second request should be faster due to caching.

## Lab 4: Security Testing

### Current Security Issues

Your setup has no Kong-level authentication. Let's verify:

```bash
# This should work (no Kong auth required)
curl -X POST https://kong-a92568d0c1us8iazk.kongcloud.dev/api/v1/anthropic \
  -H 'Content-Type: application/json' \
  -d '{
    "messages": [{"role": "user", "content": "test"}],
    "model": "claude-3-haiku-20240307",
    "max_tokens": 10
  }'
```

**Result**: You get an Anthropic auth error, not a Kong auth error.

### Exercise: Understanding Authentication Layers

1. **Kong Level**: ❌ No authentication required
2. **Upstream Level**: ✅ API keys required for OpenAI/Anthropic

This is why you can reach the upstream APIs but need their specific auth headers.

## Lab 5: Request Transformation

### Understanding stripPath Behavior

Both your routes have `stripPath: true`. Let's understand what this means:

**OpenAI Route**:
- Incoming: `POST /chat/completions`
- stripPath: true
- Upstream gets: `POST /chat/completions` (path preserved because service has same path)

**Anthropic Route**:
- Incoming: `POST /api/v1/anthropic` 
- stripPath: true
- Service path: `/v1/messages`
- Upstream gets: `POST /v1/messages`

### Exercise: Path Transformation

```bash
# Verbose curl to see actual upstream request
curl -v -X POST https://kong-a92568d0c1us8iazk.kongcloud.dev/api/v1/anthropic \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_ANTHROPIC_KEY' \
  -H 'anthropic-version: 2023-06-01' \
  -d '{
    "messages": [{"role": "user", "content": "Path test"}],
    "model": "claude-3-haiku-20240307",
    "max_tokens": 10
  }'
```

Look for the `Host` header in the verbose output - it should show `api.anthropic.com`.

## Lab 6: Error Handling and Debugging

### Common Errors and Their Meanings

```bash
# 1. Route not found (404)
curl https://kong-a92568d0c1us8iazk.kongcloud.dev/invalid-path

# 2. Method not allowed (404 or 405)
curl -X DELETE https://kong-a92568d0c1us8iazk.kongcloud.dev/api/v1/anthropic

# 3. Upstream authentication error
curl -X POST https://kong-a92568d0c1us8iazk.kongcloud.dev/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"messages": [{"role": "user", "content": "test"}], "model": "gpt-4"}'

# 4. Upstream service error (wrong model)
curl -X POST https://kong-a92568d0c1us8iazk.kongcloud.dev/api/v1/anthropic \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_ANTHROPIC_KEY' \
  -H 'anthropic-version: 2023-06-01' \
  -d '{
    "messages": [{"role": "user", "content": "test"}],
    "model": "invalid-model",
    "max_tokens": 10
  }'
```

### Exercise: Debugging Headers

```bash
# Use curl -I to see response headers
curl -I -X POST https://kong-a92568d0c1us8iazk.kongcloud.dev/chat/completions

# Look for Kong-specific headers:
# x-kong-request-id: Unique request identifier
# x-kong-upstream-latency: Time spent in upstream
# x-kong-proxy-latency: Time spent in Kong
```

## Lab 7: Performance Testing

### Measure Latency Components

```bash
# Create a simple script to measure response times
cat << 'EOF' > test_latency.sh
#!/bin/bash

echo "Testing OpenAI endpoint:"
curl -o /dev/null -s -w "Total time: %{time_total}s\nConnect time: %{time_connect}s\n" \
  -X POST https://kong-a92568d0c1us8iazk.kongcloud.dev/chat/completions \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_OPENAI_KEY' \
  -d '{
    "messages": [{"role": "user", "content": "Quick test"}],
    "model": "gpt-4"
  }'

echo -e "\nTesting Anthropic endpoint:"
curl -o /dev/null -s -w "Total time: %{time_total}s\nConnect time: %{time_connect}s\n" \
  -X POST https://kong-a92568d0c1us8iazk.kongcloud.dev/api/v1/anthropic \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_ANTHROPIC_KEY' \
  -H 'anthropic-version: 2023-06-01' \
  -d '{
    "messages": [{"role": "user", "content": "Quick test"}],
    "model": "claude-3-haiku-20240307",
    "max_tokens": 50
  }'
EOF

chmod +x test_latency.sh
./test_latency.sh
```

## Lab 8: Advanced Configuration Simulation

### Simulating Rate Limiting (Conceptual)

If rate limiting were configured, here's how it would behave:

```bash
# With rate limiting (100 requests/minute), this loop would eventually fail:
for i in {1..5}; do
  echo "Request $i:"
  curl -s -o /dev/null -w "Status: %{http_code}\n" \
    -X POST https://kong-a92568d0c1us8iazk.kongcloud.dev/chat/completions \
    -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer YOUR_OPENAI_KEY' \
    -d '{
      "messages": [{"role": "user", "content": "Rate limit test"}],
      "model": "gpt-4"
    }'
  sleep 1
done
```

Currently, all requests succeed because no rate limiting is configured.

### Simulating Authentication (Conceptual)

If Kong authentication were configured, requests would look like:

```bash
# Instead of upstream API keys, you'd use Kong API keys:
curl -X POST https://kong-a92568d0c1us8iazk.kongcloud.dev/chat/completions \
  -H 'Content-Type: application/json' \
  -H 'apikey: kong-api-key-here' \
  -d '{
    "messages": [{"role": "user", "content": "Authenticated request"}],
    "model": "gpt-4"
  }'
```

## Lab 9: Monitoring and Observability

### Request Tracking

Every Kong request gets a unique ID. Extract it:

```bash
# Get the Kong request ID from response headers
RESPONSE=$(curl -i -X POST https://kong-a92568d0c1us8iazk.kongcloud.dev/api/v1/anthropic \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_ANTHROPIC_KEY' \
  -H 'anthropic-version: 2023-06-01' \
  -d '{
    "messages": [{"role": "user", "content": "Tracking test"}],
    "model": "claude-3-haiku-20240307",
    "max_tokens": 10
  }')

echo "$RESPONSE" | grep -i "x-kong-request-id"
```

### Latency Analysis

```bash
# Detailed timing information
curl -w "@curl-format.txt" -o /dev/null -s \
  -X POST https://kong-a92568d0c1us8iazk.kongcloud.dev/chat/completions \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_OPENAI_KEY' \
  -d '{
    "messages": [{"role": "user", "content": "Timing test"}],
    "model": "gpt-4"
  }'

# Create curl-format.txt first:
cat << EOF > curl-format.txt
     time_namelookup:  %{time_namelookup}\n
        time_connect:  %{time_connect}\n
     time_appconnect:  %{time_appconnect}\n
    time_pretransfer:  %{time_pretransfer}\n
       time_redirect:  %{time_redirect}\n
  time_starttransfer:  %{time_starttransfer}\n
                     ----------\n
          time_total:  %{time_total}\n
EOF
```

## Lab 10: Best Practices Implementation

### Request Optimization

```bash
# Good: Reuse connections with HTTP/2
curl --http2 -X POST https://kong-a92568d0c1us8iazk.kongcloud.dev/chat/completions \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_OPENAI_KEY' \
  -d '{
    "messages": [{"role": "user", "content": "Optimized request"}],
    "model": "gpt-4"
  }'

# Good: Set reasonable timeouts
curl --max-time 30 --connect-timeout 5 \
  -X POST https://kong-a92568d0c1us8iazk.kongcloud.dev/api/v1/anthropic \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_ANTHROPIC_KEY' \
  -H 'anthropic-version: 2023-06-01' \
  -d '{
    "messages": [{"role": "user", "content": "Timeout test"}],
    "model": "claude-3-haiku-20240307",
    "max_tokens": 10
  }'
```

### Error Handling

```bash
# Robust error handling script
cat << 'EOF' > robust_request.sh
#!/bin/bash

make_request() {
  local response=$(curl -s -w "HTTPSTATUS:%{http_code}" \
    -X POST https://kong-a92568d0c1us8iazk.kongcloud.dev/chat/completions \
    -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer YOUR_OPENAI_KEY' \
    -d '{
      "messages": [{"role": "user", "content": "Error handling test"}],
      "model": "gpt-4"
    }')
  
  local body=$(echo $response | sed -E 's/HTTPSTATUS\:[0-9]{3}$//')
  local status=$(echo $response | tr -d '\n' | sed -E 's/.*HTTPSTATUS:([0-9]{3})$/\1/')
  
  if [ "$status" -eq 200 ]; then
    echo "Success: $body"
  else
    echo "Error ($status): $body"
  fi
}

make_request
EOF

chmod +x robust_request.sh
./robust_request.sh
```

## Summary and Next Steps

### What You've Learned

1. **Kong Basics**: Services, Routes, and Plugins
2. **Request Flow**: How requests flow through Kong
3. **Authentication**: Different auth layers (Kong vs Upstream)
4. **Plugin Effects**: Caching and AI proxy behavior
5. **Debugging**: Reading Kong headers and error messages
6. **Performance**: Measuring latency components

### Security Improvements Needed

1. **Add Kong Authentication**
   - Implement API key authentication
   - Hide upstream API keys in plugin configuration

2. **Add Rate Limiting**
   - Prevent abuse
   - Control costs

3. **Add Monitoring**
   - Request logging
   - Performance metrics

### Next Learning Steps

1. Study Kong's plugin architecture
2. Learn about Kong's Admin API
3. Understand Kong's clustering and deployment
4. Explore advanced routing and load balancing
5. Learn about Kong's enterprise features

### Recommended Reading

- [Kong Plugin Development Guide](https://docs.konghq.com/gateway/latest/plugin-development/)
- [Kong Performance Tuning](https://docs.konghq.com/gateway/latest/production/performance/)
- [Kong Security Best Practices](https://docs.konghq.com/gateway/latest/production/security/)

---

**Congratulations!** You've completed a comprehensive hands-on exploration of your Kong API Gateway setup. You now understand how Kong works and what improvements are needed for production readiness.