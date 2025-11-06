# Scalability and Architecture Options for 100 Beamline Agents

## Scalability Comparison

### Direct Imports vs HTTP Separation (100 Agents)

| Metric | Direct Imports (100 agents) | HTTP Separation (100 agents) |
|--------|----------------------------|------------------------------|
| **Repo size** | 1 massive repo (10+ GB) | 101 small repos (<100 MB each) |
| **Build time** | 45 minutes | 2 minutes per agent |
| **Deploy one agent** | Restart all 100 | Restart just 1 |
| **Dependency conflicts** | Constant nightmare | None (isolated) |
| **Team conflicts** | Daily merge conflicts | Rare/never |
| **CI/CD time** | 3 hours per PR | 30 seconds per PR |
| **Memory usage** | 12 GB (all loaded) | 120 MB per agent (only what's needed) |
| **Downtime on update** | All 100 beamlines | Only 1 beamline |
| **Debug one issue** | Search through 100 agents | Look at 1 agent |
| **Onboard new beamline** | PR to monolith | Create new repo |

---

## Real-World Architecture Options for 100 Agents

### Option 1: HTTP with Service Discovery (Recommended)

```yaml
# Framework knows about agents dynamically
services:
  framework-api:
    environment:
      - AGENT_REGISTRY_URL=http://consul:8500
  
  # Service discovery (Consul, Eureka, etcd)
  consul:
    image: consul:latest
    ports:
      - "8500:8500"
    
# Each beamline agent registers itself
beamline-5.3.1-agent:
  environment:
    - REGISTRY_URL=http://consul:8500
    - AUTO_REGISTER=true
```

**Benefits:**
- ✅ Framework discovers agents automatically
- ✅ Agents can be anywhere (different data centers)
- ✅ Load balancing (multiple instances per beamline)
- ✅ Health checks (automatic failover)

**When to Use:**
- 30+ agents
- Need high availability
- Agents span multiple data centers
- Need automatic failover
- Registration becomes a bottleneck

---

### Option 2: HTTP with API Gateway

```
User Request
    ↓
API Gateway (Kong/Nginx)
    ↓
    ├─→ Beamline 1.0.1 Agent (10.0.1.100:8053)
    ├─→ Beamline 5.3.1 Agent (10.0.5.31:8053)  
    ├─→ Beamline 7.0.1 Agent (10.0.7.1:8053)
    └─→ ... (97 more)
```

**Benefits:**
- ✅ Central routing
- ✅ Authentication/authorization at gateway
- ✅ Rate limiting per beamline
- ✅ Monitoring/logging centralized

**Docker Compose Example:**

```yaml
version: '3.8'

services:
  # Framework API
  framework-api:
    build: .
    environment:
      # Framework only talks to gateway!
      - AGENT_GATEWAY_URL=http://api-gateway:8080
    networks:
      - agent-network

  # API Gateway (Kong)
  api-gateway:
    image: kong:3.4-alpine
    container_name: api-gateway
    environment:
      - KONG_DATABASE=off
      - KONG_DECLARATIVE_CONFIG=/usr/local/kong/kong.yml
    volumes:
      - ./kong.yml:/usr/local/kong/kong.yml:ro
    ports:
      - "8080:8000"      # Gateway proxy port
      - "8001:8001"      # Admin API
    networks:
      - agent-network
```

**Kong Configuration (kong.yml):**

```yaml
_format_version: "3.0"

services:
  # Beamline 5.3.1 Agent
  - name: beamline-531-service
    url: http://beamline-531-agent:8053
    routes:
      - name: beamline-531-route
        paths:
          - /agents/beamline-531
        strip_path: true
    plugins:
      - name: rate-limiting
        config:
          minute: 100
          policy: local
      - name: request-transformer
        config:
          add:
            headers:
              - X-Agent-Name:beamline_531

  # Add more beamline agents here...
```

**When to Use:**
- 20+ agents
- Need authentication (API keys, JWT, OAuth)
- Need rate limiting
- Need centralized monitoring
- Multiple environments (dev/staging/prod)
- Security requirements (WAF, DDoS protection)

---

### Option 3: Message Queue (for high scale)

```
Framework → RabbitMQ → [100 agent queues]
                ├─→ beamline-1.0.1-queue → Agent
                ├─→ beamline-5.3.1-queue → Agent
                └─→ ... (98 more queues)
```

**Docker Compose Example:**

```yaml
version: '3.8'

services:
  # Framework
  framework-api:
    build: .
    environment:
      - RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672
    depends_on:
      - rabbitmq

  # Message Queue
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"      # AMQP port
      - "15672:15672"    # Management UI
    environment:
      - RABBITMQ_DEFAULT_USER=guest
      - RABBITMQ_DEFAULT_PASS=guest

# Each agent listens to its own queue
beamline-531-agent:
  environment:
    - RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672
    - QUEUE_NAME=beamline-531-queue
```

**Benefits:**
- ✅ Fully asynchronous
- ✅ Automatic load balancing
- ✅ Message persistence (don't lose requests)
- ✅ Best for high-volume (1M+ requests/day)

**When to Use:**
- Very high volume (100k+ requests/day)
- Need asynchronous processing
- Need message persistence
- Need guaranteed delivery
- Complex workflow orchestration

---

## My Strong Recommendation for 100 Beamlines

### Start with HTTP + Simple Registry

**Framework compose file:**

```yaml
# docker-compose.yml (Framework repository)
version: '3.8'

name: alpha-framework

services:
  framework-api:
    build: .
    # Just framework
    
  agent-registry:
    build: ./registry
    # Simple registry service
    ports:
      - "8100:8100"
    
  postgres:
    image: postgres:15
    # Database for checkpointing
    
  # NO beamline agents here!
```

**Each beamline deploys separately:**

```yaml
# beamline-5.3.1/docker-compose.yml
version: '3.8'

name: beamline-531-agent

services:
  beamline-531:
    build: .
    environment:
      - FRAMEWORK_URL=http://framework-api:8000
      - REGISTRY_URL=http://agent-registry:8100
      - AUTO_REGISTER=true
    networks:
      - agent-network

networks:
  agent-network:
    external: true  # Connect to framework network
```

### Deploy Strategy

```bash
# 1. Central machine - Deploy framework
cd alpha-berkeley-framework/
docker compose --profile core up -d
# Result: framework-api, agent-registry, postgres running

# 2. Beamline 5.3.1 control computer
cd beamline-5.3.1-agent/
docker network create agent-network  # If not exists
docker compose up -d
# Result: beamline-531 agent running, registered with framework

# 3. Beamline 7.0.1 control computer  
cd beamline-7.0.1-agent/
docker compose up -d
# Result: beamline-701 agent running, registered with framework

# 4. Repeat for each beamline...
# ... each beamline deploys on their own machine
```

---

## Evolution Path

### Phase 1: Simple Registry (Start Here - Good for 5-20 agents)

```python
# src/framework/infrastructure/agent_registry.py
class AgentRegistry:
    def __init__(self):
        self._agents = {}  # Simple in-memory dict
    
    def register_agent(self, name, url, capabilities):
        self._agents[name] = AgentInfo(...)
```

**Pros:**
- ✅ Simple to implement
- ✅ Easy to understand
- ✅ No external dependencies

**Cons:**
- ❌ Lost on restart
- ❌ No health checks
- ❌ Single instance only

---

### Phase 2: Add Persistence (Good for 20-50 agents)

```python
# Add Redis/PostgreSQL persistence
class AgentRegistry:
    def __init__(self, redis_client):
        self.redis = redis_client
    
    def register_agent(self, name, url, capabilities):
        # Store in memory AND Redis
        self._agents[name] = agent_info
        self.redis.set(f"agent:{name}", json.dumps(agent_info))
```

**Pros:**
- ✅ Survives restarts
- ✅ Can have backup instances

---

### Phase 3: Add Health Checks (Good for 50-100 agents)

```python
# Add background health check thread
class AgentRegistry:
    async def start_health_monitoring(self):
        asyncio.create_task(self._health_check_loop())
    
    async def _health_check_loop(self):
        while True:
            for agent in self._agents:
                if not await self.check_health(agent):
                    agent.status = "unhealthy"
            await asyncio.sleep(10)
```

**Pros:**
- ✅ Automatic failure detection
- ✅ Automatic agent removal when down

---

### Phase 4: Migrate to Consul (Good for 100+ agents)

```yaml
# Replace custom registry with Consul
services:
  consul:
    image: hashicorp/consul:latest
    ports:
      - "8500:8500"
    command: agent -server -ui -bootstrap-expect=1
```

**Pros:**
- ✅ Industry-standard
- ✅ Highly available (cluster mode)
- ✅ Advanced features (health checks, DNS, KV store)
- ✅ Built-in monitoring

---

## Bottom Line

### With 100 beamline agents:

#### ❌ **Direct Imports = IMPOSSIBLE to manage**

**Problems:**
- Single monolith with 100 agents
- Every change affects everyone
- Hours to build, test, deploy
- Constant team conflicts
- All 100 beamlines go down together
- Dependency hell
- 10+ GB repository
- 3-hour CI/CD pipelines

#### ✅ **HTTP Separation = ESSENTIAL for survival**

**Benefits:**
- 100 independent repositories
- Changes isolated to one beamline
- Minutes to build, test, deploy
- Teams work independently
- Only affected beamline goes down
- No dependency conflicts
- Small repositories (<100 MB)
- 30-second CI/CD per agent

---

## Decision Matrix

| Number of Agents | Recommended Architecture |
|-----------------|--------------------------|
| **1-10 agents** | Simple HTTP + Manual registry (your current solution) |
| **10-30 agents** | HTTP + Simple registry with persistence (Redis) |
| **30-50 agents** | HTTP + Service discovery (Consul) OR API Gateway (Kong) |
| **50-100 agents** | Service Discovery (Consul) + API Gateway (Kong) |
| **100+ agents** | Service Discovery + API Gateway + Message Queue |

---

## Key Takeaway

**You MUST separate them.** 

Direct imports don't scale beyond ~5-10 agents maximum.

With 100 beamlines, HTTP (or gRPC/message queue) isn't just "better" - **it's the only viable option.**

Your current proposed solution (HTTP with simple registry) is:
- ✅ Perfect starting point for 5-20 agents
- ✅ Can evolve to Option 1 (Consul) when needed
- ✅ Can add Option 2 (API Gateway) features incrementally
- ✅ Much better than direct imports monolith