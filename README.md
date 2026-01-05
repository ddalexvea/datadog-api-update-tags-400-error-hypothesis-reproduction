# OpenAPI vs Form-Data Validation Bug

## Problem

A mismatch between **OpenAPI schema validation** and **Flask's request handling** causes requests to fail when:

- The **OpenAPI schema** defines parameters as `in: query` (query string)
- The **frontend** sends parameters as `multipart/form-data` (request body)
- **Flask's `request.values`** would accept either, but OpenAPI validation rejects the request **before** Flask sees it

## Real-World Example (Datadog)

This reproduces an issue found in Infrastructure Map vs Host List:

| UI Component | How it sends `host_alias` | Result |
|--------------|---------------------------|--------|
| **Host List** | Query string (`?host_alias=xxx`) | ✅ Works |
| **Infrastructure Map** | Form-data body | ❌ 400 Error |

Error from production:
```json
{"status":"400","title":"Missing Parameter","detail":"missing parameter \"host_alias\" in \"query\""}
```

---

## Quick Start (Minikube)

### 1. Create the directory

```bash
mkdir -p openapi-formdata-bug && cd openapi-formdata-bug
```

### 2. Create requirements.txt

```bash
cat > requirements.txt << 'EOF'
flask==3.0.0
connexion[flask,swagger-ui,uvicorn]==3.0.5
EOF
```

### 3. Create app.py

```bash
cat > app.py << 'EOF'
from flask import Flask, request, jsonify
import connexion

app = connexion.FlaskApp(__name__, specification_dir='./')
app.add_api('openapi.yaml', validate_responses=True)
flask_app = app.app

@flask_app.route('/direct/update_tags', methods=['POST'])
def direct_update_tags():
    host_alias = request.values.get('host_alias')
    user_tags = request.values.get('user_tags', '')
    if not host_alias:
        return jsonify({"error": "Missing host_alias parameter"}), 400
    return jsonify({
        "message": "Success (direct Flask - no OpenAPI)",
        "source": "request.values (query OR form-data)",
        "host_alias": host_alias,
        "user_tags": user_tags
    })

def update_tags(host_alias, user_tags=None):
    return {
        "message": "Success (OpenAPI validated)",
        "source": "query params only (per OpenAPI spec)",
        "host_alias": host_alias,
        "user_tags": user_tags or ""
    }

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
```

### 4. Create openapi.yaml

```bash
cat > openapi.yaml << 'EOF'
openapi: 3.0.0
info:
  title: OpenAPI Form-Data Bug Reproduction
  version: 1.0.0
paths:
  /source/update_tags:
    post:
      operationId: app.update_tags
      summary: Update host tags (OpenAPI validated)
      parameters:
        - name: host_alias
          in: query
          required: true
          schema:
            type: string
        - name: user_tags
          in: query
          required: false
          schema:
            type: string
      requestBody:
        required: false
        content:
          application/json:
            schema:
              type: object
      responses:
        '200':
          description: Success
        '400':
          description: Bad Request
EOF
```

### 5. Create Dockerfile

```bash
cat > Dockerfile << 'EOF'
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py openapi.yaml ./
EXPOSE 5000
CMD ["python", "app.py"]
EOF
```

### 6. Create deployment.yaml

```bash
cat > deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openapi-bug-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: openapi-bug-demo
  template:
    metadata:
      labels:
        app: openapi-bug-demo
    spec:
      containers:
      - name: app
        image: openapi-bug-demo:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: openapi-bug-demo
spec:
  type: NodePort
  selector:
    app: openapi-bug-demo
  ports:
  - port: 5000
    targetPort: 5000
    nodePort: 30500
EOF
```

### 7. Create test.sh

```bash
cat > test.sh << 'EOF'
#!/bin/bash
BASE_URL="${1:-http://localhost:5000}"
echo "=== TEST 1: OpenAPI + Query Params (✅ Works) ==="
curl -s -X POST "$BASE_URL/source/update_tags?host_alias=minikube&user_tags=env:test" -H "Content-Type: application/json" -d '{}'
echo -e "\n\n=== TEST 2: OpenAPI + Form-Data (❌ 400 Error) ==="
curl -s -X POST "$BASE_URL/source/update_tags" --form "host_alias=minikube" --form "user_tags=env:test"
echo -e "\n\n=== TEST 3: Direct Flask + Query Params (✅ Works) ==="
curl -s -X POST "$BASE_URL/direct/update_tags?host_alias=minikube&user_tags=env:test"
echo -e "\n\n=== TEST 4: Direct Flask + Form-Data (✅ Works) ==="
curl -s -X POST "$BASE_URL/direct/update_tags" --form "host_alias=minikube" --form "user_tags=env:test"
echo -e "\n"
EOF
chmod +x test.sh
```

### 8. Build & Deploy to Minikube

```bash
eval $(minikube docker-env)
docker build -t openapi-bug-demo:latest .
kubectl apply -f deployment.yaml
kubectl wait --for=condition=ready pod -l app=openapi-bug-demo --timeout=60s
```

### 9. Test

```bash
kubectl port-forward svc/openapi-bug-demo 5000:5000 &
sleep 3
./test.sh http://localhost:5000
```

### 10. Cleanup

```bash
kubectl delete -f deployment.yaml
```

---

## Files

### `app.py`

```python
"""
Reproduction of OpenAPI vs Flask form-data bug.

Flask accepts both query params and form-data via request.values,
but OpenAPI validation only allows what's defined in the spec (query params).
"""
from flask import Flask, request, jsonify
import connexion

# Create Connexion app (Flask + OpenAPI validation)
app = connexion.FlaskApp(__name__, specification_dir='./')
app.add_api('openapi.yaml', validate_responses=True)

# Get the underlying Flask app for direct routes (no OpenAPI validation)
flask_app = app.app


@flask_app.route('/direct/update_tags', methods=['POST'])
def direct_update_tags():
    """
    Direct Flask endpoint - NO OpenAPI validation.
    Uses request.values which accepts BOTH query params AND form-data.
    """
    host_alias = request.values.get('host_alias')
    user_tags = request.values.get('user_tags', '')
    
    if not host_alias:
        return jsonify({"error": "Missing host_alias parameter"}), 400
    
    return jsonify({
        "message": "Success (direct Flask - no OpenAPI)",
        "source": "request.values (query OR form-data)",
        "host_alias": host_alias,
        "user_tags": user_tags
    })


# OpenAPI-validated endpoint is defined in openapi.yaml and handled below
def update_tags(host_alias, user_tags=None):
    """
    OpenAPI-validated endpoint handler.
    Parameters come from OpenAPI validation layer.
    """
    return {
        "message": "Success (OpenAPI validated)",
        "source": "query params only (per OpenAPI spec)",
        "host_alias": host_alias,
        "user_tags": user_tags or ""
    }


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

### `openapi.yaml`

```yaml
openapi: 3.0.0
info:
  title: OpenAPI Form-Data Bug Reproduction
  description: |
    Demonstrates the mismatch between OpenAPI validation and Flask's request.values.
    
    The /source/update_tags endpoint requires host_alias as a QUERY parameter.
    Sending it in form-data body will fail OpenAPI validation, even though
    Flask's request.values would accept it.
  version: 1.0.0

paths:
  /source/update_tags:
    post:
      operationId: app.update_tags
      summary: Update host tags (OpenAPI validated)
      description: |
        This endpoint requires host_alias as a QUERY parameter.
        Form-data in the body will be rejected by OpenAPI validation.
      parameters:
        - name: host_alias
          in: query
          required: true
          description: The host alias to update tags for
          schema:
            type: string
        - name: user_tags
          in: query
          required: false
          description: Comma-separated list of tags
          schema:
            type: string
      requestBody:
        required: false
        content:
          application/json:
            schema:
              type: object
              properties:
                _authentication_token:
                  type: string
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                type: object
                properties:
                  message:
                    type: string
                  host_alias:
                    type: string
                  user_tags:
                    type: string
        '400':
          description: Bad Request - Missing required parameter
          content:
            application/json:
              schema:
                type: object
                properties:
                  status:
                    type: integer
                  title:
                    type: string
                  detail:
                    type: string
```

---

### `requirements.txt`

```
flask==3.0.0
connexion[flask,swagger-ui,uvicorn]==3.0.5
```

---

### `Dockerfile`

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py openapi.yaml ./

EXPOSE 5000

CMD ["python", "app.py"]
```

---

### `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openapi-bug-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: openapi-bug-demo
  template:
    metadata:
      labels:
        app: openapi-bug-demo
    spec:
      containers:
      - name: app
        image: openapi-bug-demo:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: openapi-bug-demo
spec:
  type: NodePort
  selector:
    app: openapi-bug-demo
  ports:
  - port: 5000
    targetPort: 5000
    nodePort: 30500
```

---

### `test.sh`

```bash
#!/bin/bash

BASE_URL="${1:-http://localhost:5000}"

echo "=========================================="
echo "OpenAPI vs Form-Data Bug Reproduction"
echo "=========================================="
echo ""
echo "Testing against: $BASE_URL"
echo ""

echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "TEST 1: OpenAPI endpoint + Query Params (✅ Works)"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
curl -s -X POST "$BASE_URL/source/update_tags?host_alias=minikube&user_tags=env:test" \
  -H "Content-Type: application/json" -d '{}'
echo -e "\n"

echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "TEST 2: OpenAPI endpoint + Form-Data (❌ 400 Error)"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
curl -s -X POST "$BASE_URL/source/update_tags" \
  --form "host_alias=minikube" \
  --form "user_tags=env:test"
echo -e "\n"

echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "TEST 3: Direct Flask + Query Params (✅ Works)"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
curl -s -X POST "$BASE_URL/direct/update_tags?host_alias=minikube&user_tags=env:test"
echo -e "\n"

echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "TEST 4: Direct Flask + Form-Data (✅ Works)"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
curl -s -X POST "$BASE_URL/direct/update_tags" \
  --form "host_alias=minikube" \
  --form "user_tags=env:test"
echo -e "\n"

echo "=========================================="
echo "SUMMARY"
echo "=========================================="
echo ""
echo "┌─────────────────────┬──────────────┬─────────────┐"
echo "│ Endpoint            │ Query Params │ Form-Data   │"
echo "├─────────────────────┼──────────────┼─────────────┤"
echo "│ /source/update_tags │ ✅ Works     │ ❌ Rejected │"
echo "│ (OpenAPI validated) │              │             │"
echo "├─────────────────────┼──────────────┼─────────────┤"
echo "│ /direct/update_tags │ ✅ Works     │ ✅ Works    │"
echo "│ (Flask only)        │              │             │"
echo "└─────────────────────┴──────────────┴─────────────┘"
```

---

## Test Results

### TEST 1: OpenAPI endpoint + Query Params ✅

```bash
curl -X POST 'http://localhost:5000/source/update_tags?host_alias=minikube&user_tags=env:test' \
  -H "Content-Type: application/json" -d '{}'
```

**Result:** ✅ Success
```json
{
  "host_alias": "minikube",
  "message": "Success (OpenAPI validated)",
  "source": "query params only (per OpenAPI spec)",
  "user_tags": "env:test"
}
```

### TEST 2: OpenAPI endpoint + Form-Data ❌

```bash
curl -X POST 'http://localhost:5000/source/update_tags' \
  --form "host_alias=minikube" \
  --form "user_tags=env:test"
```

**Result:** ❌ 400 Error
```json
{"type": "about:blank", "title": "Bad Request", "detail": "Missing query parameter 'host_alias'", "status": 400}
```

### TEST 3: Direct Flask + Query Params ✅

```bash
curl -X POST 'http://localhost:5000/direct/update_tags?host_alias=minikube&user_tags=env:test'
```

**Result:** ✅ Success
```json
{
  "host_alias": "minikube",
  "message": "Success (direct Flask - no OpenAPI)",
  "source": "request.values (query OR form-data)",
  "user_tags": "env:test"
}
```

### TEST 4: Direct Flask + Form-Data ✅

```bash
curl -X POST 'http://localhost:5000/direct/update_tags' \
  --form "host_alias=minikube" \
  --form "user_tags=env:test"
```

**Result:** ✅ Success
```json
{
  "host_alias": "minikube",
  "message": "Success (direct Flask - no OpenAPI)",
  "source": "request.values (query OR form-data)",
  "user_tags": "env:test"
}
```

---

## Summary

```
┌─────────────────────┬──────────────┬─────────────┐
│ Endpoint            │ Query Params │ Form-Data   │
├─────────────────────┼──────────────┼─────────────┤
│ /source/update_tags │ ✅ Works     │ ❌ Rejected │
│ (OpenAPI validated) │              │             │
├─────────────────────┼──────────────┼─────────────┤
│ /direct/update_tags │ ✅ Works     │ ✅ Works    │
│ (Flask only)        │              │             │
└─────────────────────┴──────────────┴─────────────┘
```

---

## Root Cause

```
Frontend sends:
  POST /source/update_tags
  Content-Type: multipart/form-data
  Body: { host_alias: "minikube", user_tags: "env:test" }
         │
         ▼
┌─────────────────────────────────────┐
│     OpenAPI Validation Layer        │  ❌ REJECTED!
│                                     │
│  Schema says: host_alias in: query  │
│  Request has: host_alias in: body   │
└─────────────────────────────────────┘
         │
         ✖ Request never reaches Flask
         ▼
┌─────────────────────────────────────┐
│     Flask Endpoint                  │  Would accept it!
│                                     │
│  request.values.get("host_alias")   │
│  (accepts query OR form-data)       │
└─────────────────────────────────────┘
```

---

## Fix Options

### Option 1: Fix Frontend (Recommended)

Change the frontend to send `host_alias` as a query parameter:

```typescript
// Before (broken - Infrastructure Map)
HttpPOST('/source/update_tags', { type: 'formdata' })({ host_alias, user_tags });

// After (working - like Host List)
HttpPOST(`/source/update_tags?host_alias=${host_alias}&user_tags=${user_tags}`)({});
```

### Option 2: Update OpenAPI Schema

Change the schema to accept parameters in the request body:

```yaml
# Before (current)
parameters:
  - name: host_alias
    in: query  # ❌ Only accepts query params

# After (accepts form-data)
requestBody:
  content:
    multipart/form-data:
      schema:
        properties:
          host_alias:
            type: string
          user_tags:
            type: string
```
