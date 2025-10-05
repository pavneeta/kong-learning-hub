# Kong AI Gateway - Secure MCP Traffic Architecture

## Flow Diagram

```mermaid
flowchart TD
    %% External entities
    Client[Client Application]
    OpenAI[OpenAI API<br/>gpt-4o]
    MCPServer[MCP Server<br/>GitHub Tools]
    
    %% Kong Gateway components
    subgraph Kong["Kong AI Gateway Enterprise"]
        direction TB
        
        %% Gateway core
        Gateway[Kong Gateway Core]
        
        %% Plugins
        subgraph Plugins["Kong Plugins"]
            AIProxy[AI Proxy Advanced Plugin]
            KeyAuth[Key Authentication Plugin]
            RateLimit[Rate Limiting<br/>Security Policies]
        end
        
        %% Services and Routes
        subgraph Services["Kong Services"]
            ExampleService[example-clean-service]
        end
        
        %% Consumer management
        subgraph Consumers["Kong Consumers"]
            Consumer[Consumer: alex<br/>API Key: secret-key]
        end
    end
    
    %% External tools
    GitHub[GitHub Repository<br/>Tools & Resources]
    
    %% Flow connections
    Client -->|1. Request with API Key| Gateway
    Gateway -->|2. Authenticate| KeyAuth
    KeyAuth -->|3. Validate Consumer| Consumer
    Gateway -->|4. Apply Security| RateLimit
    Gateway -->|5. Route to Service| ExampleService
    ExampleService -->|6. AI Proxy Config| AIProxy
    
    %% AI interactions
    AIProxy -->|7. Forward to LLM| OpenAI
    OpenAI -->|8. Generate Response<br/>with Tool Calls| AIProxy
    
    %% MCP integration
    AIProxy -->|9. Direct Tool Invocation<br/>No Backend Routing| MCPServer
    MCPServer -->|10. Execute Tools| GitHub
    GitHub -->|11. Return Data| MCPServer
    MCPServer -->|12. Tool Response| AIProxy
    
    %% Response flow
    AIProxy -->|13. Complete Response| ExampleService
    ExampleService -->|14. Apply Policies| Gateway
    Gateway -->|15. Final Response| Client
    
    %% Configuration flow
    subgraph Config["Configuration Management"]
        DecK[decK CLI Tool<br/>Declarative Config]
        ConfigFiles[YAML Config Files<br/>_format_version: "3.0"]
    end
    
    DecK -->|Deploy Configuration| Kong
    ConfigFiles -->|Define Services<br/>Plugins & Consumers| DecK
    
    %% Styling
    classDef client fill:#e1f5fe
    classDef kong fill:#f3e5f5
    classDef external fill:#fff3e0
    classDef plugin fill:#e8f5e8
    classDef config fill:#fce4ec
    
    class Client client
    class Kong,Gateway,Services,Consumers kong
    class OpenAI,MCPServer,GitHub external
    class Plugins,AIProxy,KeyAuth,RateLimit plugin
    class Config,DecK,ConfigFiles config
```

## Architecture Components

### 1. Client Layer
- **Client Application**: Makes authenticated requests to Kong Gateway
- **Authentication**: Uses API key-based authentication

### 2. Kong AI Gateway Enterprise
- **Gateway Core**: Main routing and policy enforcement engine
- **AI Proxy Advanced Plugin**: 
  - Routes requests to OpenAI LLM
  - Handles MCP tool invocations directly
  - No custom backend routing required
- **Key Authentication Plugin**: Validates API keys against consumers
- **Rate Limiting**: Applies security policies and traffic controls
- **Consumer Management**: Defines clients with API keys

### 3. External Services
- **OpenAI API**: Provides LLM capabilities (gpt-4o model)
- **MCP Server**: Exposes standardized tools for GitHub integration
- **GitHub Repository**: Target system for tool operations

### 4. Configuration Management
- **decK CLI**: Declarative configuration management tool
- **YAML Configuration**: Version-controlled service definitions

## Key Security Features

### Authentication Flow
1. Client sends request with API key
2. Kong validates key against consumer registry
3. Request proceeds only if authenticated

### MCP Integration Benefits
- **Direct Tool Routing**: AI-generated tool calls go directly to MCP server
- **No Backend Complexity**: Eliminates need for custom routing code
- **Standardized Interface**: MCP provides consistent tool exposure
- **Kong Security**: All traffic benefits from Kong's security policies

### Configuration Example

```yaml
_format_version: "3.0"
plugins:
  - name: ai-proxy-advanced
    config:
      targets:
        - route_type: llm/v1/responses
          auth:
            header_name: Authorization
            header_value: Bearer ${{ env "DECK_OPENAI_API_KEY" }}
      model:
        provider: openai
        name: gpt-4o
        options:
          max_tokens: 512
          temperature: 1.0

  - name: key-auth
    service: example-clean-service
    config:
      key_names:
        - apikey

consumers:
  - username: alex
    keyauth_credentials:
      - key: secret-key
```

## Benefits

1. **Simplified Integration**: No custom glue code required
2. **Enterprise Security**: Built-in Kong security capabilities
3. **Declarative Management**: Infrastructure as code with decK
4. **Scalable Architecture**: Enterprise-grade traffic handling
5. **Tool Standardization**: MCP provides consistent tool interface
