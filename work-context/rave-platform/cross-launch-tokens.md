# Wellness Proxy & Cross-Launch Tokens — RAVE Platform

## Overview

I contributed to the Wellness Proxy, a Python/FastAPI service that acts as an intermediary for cross-launch token handling and Nimble-specific route proxying within the RAVE platform. This service enables users who navigate to RAVE from other GLCP applications (cross-launch) to use their existing session tokens, and provides specialized routing for Nimble storage device wellness endpoints. I also handled HPA configuration, CVE fixes, and dependency management for this service.

---

## Architecture

### Where Wellness Proxy Sits

```
GLCP UI (Cross-Launch from Nimble Console)
    │
    │ Cross-launch token (different audience/issuer than RAVE standard)
    │
    ▼
┌──────────────────┐
│  Wellness Proxy   │ (Python / FastAPI)
│                    │
│  ├── Token Exchange│ ── Validate cross-launch token
│  ├── Route Mapping │ ── Map Nimble routes to RAVE endpoints
│  ├── Auth Header   │ ── Inject RAVE-compatible token
│  └── Proxy Forward │ ── Forward to wellnessProducer
└──────────────────┘
         │
         ▼
┌──────────────────┐
│ wellnessProducer  │ (Go)
│ (Standard auth)   │
└──────────────────┘
```

### Why a Separate Proxy

1. **Token format mismatch**: Cross-launch tokens from Nimble Console have different audience/issuer than what wellnessProducer expects. The proxy handles token exchange/validation without modifying the core Go service.

2. **Route translation**: Nimble-specific endpoints need to be mapped to RAVE's generic wellness API. The proxy translates Nimble device identifiers and route patterns.

3. **Separation of concerns**: Cross-launch logic is GLCP-platform-specific glue code. Keeping it in a separate Python service keeps wellnessProducer focused on core wellness logic.

4. **Language choice**: Python/FastAPI for rapid iteration on GLCP integration patterns — the GLCP platform team provides Python SDKs for token exchange.

---

## Cross-Launch Token Support

### What is Cross-Launch?

When a user is in the Nimble Console (a GLCP application) and clicks "View Wellness" for a device, they're "cross-launched" into the RAVE wellness UI. Their existing Nimble Console session token is passed along, but it has:

- **Different audience**: Nimble Console's audience, not RAVE's
- **Different issuer context**: May include Nimble-specific claims
- **Same user identity**: Same `sub` claim, same workspace

### Token Flow

```
1. User in Nimble Console (has Nimble session token)
2. Clicks "View Wellness for Device X"
3. GLCP redirects to RAVE with cross-launch token

4. Wellness Proxy:
   a. Receives request with cross-launch token
   b. Validates token signature (JWKS from GLCP)
   c. Extracts user identity and workspace
   d. Verifies the user has access to the target device
   e. Constructs a RAVE-compatible request context
   f. Forwards to wellnessProducer with proper auth

5. wellnessProducer:
   a. Receives proxied request with valid auth context
   b. Processes normally (standard auth middleware)
   c. Returns wellness data for the device
```

### Token Validation in Proxy

```python
from fastapi import FastAPI, Request, HTTPException
from jose import jwt, JWTError
import httpx

app = FastAPI()

GLCP_JWKS_URL = os.environ["GLCP_JWKS_URL"]
ALLOWED_AUDIENCES = os.environ["ALLOWED_AUDIENCES"].split(",")

async def validate_cross_launch_token(token: str) -> dict:
    """Validate cross-launch token from GLCP."""
    try:
        # Fetch JWKS (cached)
        jwks = await get_jwks(GLCP_JWKS_URL)
        
        # Decode and validate
        claims = jwt.decode(
            token,
            jwks,
            algorithms=["RS256"],
            audience=ALLOWED_AUDIENCES,
        )
        
        # Extract workspace and user identity
        workspace_id = claims.get("hpe_ccs_workspace_id")
        user_id = claims.get("sub")
        
        if not workspace_id or not user_id:
            raise HTTPException(401, "Missing required claims")
        
        return claims
        
    except JWTError as e:
        raise HTTPException(401, f"Token validation failed: {str(e)}")
```

### Nimble Route Mapping

```python
NIMBLE_ROUTE_MAP = {
    "/nimble/v1/devices/{device_id}/wellness": "/v3/wellness?deviceId={device_id}",
    "/nimble/v1/devices/{device_id}/cases": "/v3/wellness?deviceId={device_id}&alertType=manual",
    "/nimble/v1/devices/{device_id}/alerts": "/v3/wellness?deviceId={device_id}&severity=high",
}

@app.get("/nimble/v1/devices/{device_id}/wellness")
async def get_nimble_device_wellness(
    device_id: str,
    request: Request,
    claims: dict = Depends(validate_cross_launch_token),
):
    """Proxy Nimble wellness request to wellnessProducer."""
    
    # Verify device belongs to user's workspace
    if not await verify_device_access(device_id, claims["hpe_ccs_workspace_id"]):
        raise HTTPException(403, "Device not accessible in this workspace")
    
    # Forward to wellnessProducer
    async with httpx.AsyncClient() as client:
        response = await client.get(
            f"{WELLNESS_PRODUCER_URL}/v3/wellness",
            params={"deviceId": device_id},
            headers={
                "Authorization": request.headers.get("Authorization"),
                "X-Workspace-ID": claims["hpe_ccs_workspace_id"],
                "X-Request-ID": request.headers.get("X-Request-ID", str(uuid4())),
            },
        )
    
    return Response(
        content=response.content,
        status_code=response.status_code,
        headers=dict(response.headers),
    )
```

---

## HPA Configuration

### Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: wellness-proxy
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: wellness-proxy
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 25
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
```

**Configuration rationale**:
- **minReplicas: 2**: Always at least 2 pods for availability (survives single pod failure)
- **maxReplicas: 10**: Caps resource usage, sufficient for observed peak traffic
- **CPU target 70%**: Leaves headroom for request spikes
- **Scale-down stabilization 300s**: Prevents flapping during variable traffic (5-minute cooldown)
- **Scale-up fast (30s)**: Quick scale-up during traffic bursts
- **Scale-down conservative (25%/min)**: Gradual scale-down to avoid over-correction

---

## CVE Fixes & Dependency Management

### Security Vulnerability Process

As part of my responsibilities, I handled CVE fixes for the Wellness Proxy's Python dependencies:

```python
# requirements.txt — before CVE fix
fastapi==0.95.0
uvicorn==0.21.1
httpx==0.23.3
python-jose==3.3.0
cryptography==38.0.4  # CVE-2023-XXXXX

# requirements.txt — after CVE fix
fastapi==0.104.1
uvicorn==0.24.0
httpx==0.25.2
python-jose==3.3.0
cryptography==41.0.7  # Patched version
```

### CVE Fix Process

1. **Detection**: Automated dependency scanning in CI/CD (GitHub Actions) flags vulnerable packages
2. **Assessment**: Evaluate CVE severity (CVSS score), exploitability in our context
3. **Update**: Bump dependency version, run tests
4. **Test**: Full test suite + integration tests with Nimble route proxying
5. **Deploy**: Through standard GitOps pipeline (ArgoCD)

### Docker Image Security

```dockerfile
FROM python:3.11-slim-bookworm

# Security: Non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# Security: Run as non-root
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
    CMD curl -f http://localhost:8000/health || exit 1

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Security practices**:
- **Slim base image**: Minimal attack surface (`python:3.11-slim-bookworm`)
- **Non-root user**: Application runs as `appuser`, not root
- **No cache**: `--no-cache-dir` reduces image size and removes pip cache
- **Health check**: Kubernetes liveness probe via health endpoint

---

## Integration with RAVE Ecosystem

### Communication with WellnessProducer

```
Wellness Proxy ──HTTP──▶ wellnessProducer (via Istio service mesh)
```

- **Protocol**: HTTP/2 via Istio sidecar (mTLS)
- **Auth propagation**: Cross-launch token validated by proxy, forwarded to wellnessProducer
- **Request tracing**: X-Request-ID header propagated for distributed tracing

### Monitoring

```python
from prometheus_client import Counter, Histogram

cross_launch_requests = Counter(
    "wellness_proxy_cross_launch_total",
    "Total cross-launch requests",
    ["source_app", "status"],
)

proxy_latency = Histogram(
    "wellness_proxy_latency_seconds",
    "Proxy request latency",
    ["route", "method"],
)
```

Metrics exposed at `/metrics` endpoint, scraped by Prometheus.

---

## Testing

### Unit Tests

```python
def test_cross_launch_token_validation():
    """Valid cross-launch token should be accepted."""
    token = generate_test_jwt(
        sub="user-123",
        aud="nimble-console",
        workspace_id="ws-456",
    )
    claims = validate_cross_launch_token(token)
    assert claims["sub"] == "user-123"
    assert claims["hpe_ccs_workspace_id"] == "ws-456"

def test_cross_launch_token_wrong_audience():
    """Token with wrong audience should be rejected."""
    token = generate_test_jwt(aud="wrong-audience")
    with pytest.raises(HTTPException) as exc:
        validate_cross_launch_token(token)
    assert exc.value.status_code == 401

def test_nimble_route_mapping():
    """Nimble device wellness route should map to v3 endpoint."""
    response = client.get(
        "/nimble/v1/devices/dev-123/wellness",
        headers={"Authorization": f"Bearer {valid_token}"},
    )
    assert response.status_code == 200

def test_device_access_control():
    """User should only access devices in their workspace."""
    token = generate_test_jwt(workspace_id="workspace-A")
    response = client.get(
        "/nimble/v1/devices/device-in-workspace-B/wellness",
        headers={"Authorization": f"Bearer {token}"},
    )
    assert response.status_code == 403
```

### Integration Tests

- Cross-launch flow with mock GLCP token provider
- Route mapping with mock wellnessProducer
- HPA behavior under load (k6 load testing)

---

## Deployment

### Helm Chart

```yaml
# values.yaml
image:
  repository: hpe-docker-registry/rave/wellness-proxy
  tag: "1.4.2"

replicaCount: 2

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

env:
  - name: WELLNESS_PRODUCER_URL
    value: "http://wellness-producer.rave.svc.cluster.local:8080"
  - name: GLCP_JWKS_URL
    valueFrom:
      configMapKeyRef:
        name: rave-auth-config
        key: glcp-jwks-url
```

### ArgoCD Sync

Wellness Proxy follows the same GitOps pipeline as all RAVE services:
1. Code change → PR → Review → Merge
2. CI builds Docker image, pushes to registry
3. Helm values updated with new image tag
4. ArgoCD detects change, syncs to cluster
5. Rolling update with zero downtime

---

## [KEY_POINTS]

- Wellness Proxy bridges cross-launch tokens from Nimble Console to RAVE's auth model
- Python/FastAPI chosen for rapid GLCP SDK integration (platform team provides Python SDKs)
- HPA configured with conservative scale-down (300s stabilization) and fast scale-up (30s)
- CVE fixes handled through automated scanning → assessment → update → test → deploy pipeline
- Device access control: proxy verifies device belongs to user's workspace before forwarding

## [COMMON_MISTAKES]

- Thinking the proxy is just a reverse proxy — it does token validation and route translation
- Assuming cross-launch tokens are the same as standard RAVE tokens — different audience/issuer
- Overlooking the device access check — proxy must verify workspace-device relationship
- Confusing HPA scale-down stabilization with scale-up — they have very different windows

## [FOLLOW_UP]

- "Why Python instead of Go for this service?" → GLCP platform SDKs are Python-first, rapid iteration for integration logic
- "What happens if wellnessProducer is down?" → Proxy returns 502, client can retry, monitoring alerts
- "How do you handle token expiry during a cross-launch session?" → Token refresh via GLCP SDK, proxy re-validates
- "What CVEs have you fixed?" → Cryptography library CVEs, FastAPI security patches, httpx transport issues
