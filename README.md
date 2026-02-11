Change 1: Frontend Deployment

File: frontend-deployment.yaml
BEFORE:
yaml- name: REACT_APP_BACKEND_URL
  value: "backend:8080"
AFTER:
yaml- name: REACT_APP_BACKEND_URL
  value: "/api/tasks"
WHY:

React runs in the browser (your computer), not in the pod
"backend:8080" is internal cluster DNS - browsers can't access it
/api/tasks is a relative path - browser sends requests to same origin
With Ingress, requests to http://127.0.0.1/api/tasks get routed to backend service


Change 2: Ingress Configuration
File: minikube-ingress.yaml
FINAL VERSION:
yamlapiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: three-tier-ingress
  namespace: three-tier
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 8080
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 3000
WHY:

Ingress = Traffic Router - single entry point for all requests
Requests to /api/* → go to backend service
Requests to / → go to frontend service
We removed the rewrite-target annotation because it was incorrectly stripping /api


Change 3: Backend MongoDB Connection
File: backend-deployment.yaml
BEFORE:
yamlvalue: mongodb://mongodb-svc:27017/todo?directConnection=true
AFTER:
yamlvalue: mongodb://admin:password123@mongodb-svc:27017/todo?authSource=admin&directConnection=true
```

**WHY:**
- MongoDB requires **authentication** (username + password)
- Format: `mongodb://username:password@host:port/database?authSource=admin`
- `admin` and `password123` are from your secrets.yaml (base64 encoded)
- Without this, backend couldn't connect to MongoDB → `Unauthorized` error

---

## 🔄 **COMPLETE REQUEST FLOW**

Let me trace what happens when you add a task:

### **Step-by-Step Flow:**
```
1. USER ACTION
   └─> You type "Buy groceries" and click "ADD TASK"

2. BROWSER (React App)
   └─> taskServices.js makes POST request:
       fetch('/api/tasks', {task: 'Buy groceries'})
   └─> URL becomes: http://127.0.0.1/api/tasks

3. MINIKUBE TUNNEL
   └─> Forwards 127.0.0.1:80 → Minikube IP (192.168.49.2:80)

4. INGRESS CONTROLLER (Nginx)
   └─> Receives: GET http://192.168.49.2/api/tasks
   └─> Checks rules in minikube-ingress.yaml
   └─> Path matches: /api → route to backend service
   └─> Forwards to: http://backend:8080/api/tasks

5. BACKEND SERVICE
   └─> ClusterIP service (10.111.170.9:8080)
   └─> Routes to backend pod

6. BACKEND POD (Node.js)
   └─> index.js receives request at /api/tasks
   └─> Line 15: app.use("/api/tasks", tasks)
   └─> Calls tasks.js router
   └─> POST / handler (line 6-13)

7. MONGODB CONNECTION
   └─> Backend connects using:
       mongodb://admin:password123@mongodb-svc:27017/todo
   └─> mongodb-svc service → mongodb pod
   └─> Creates task document in 'tasks' collection

8. RESPONSE BACK
   └─> MongoDB confirms save
   └─> Backend returns task object: {_id: "...", task: "Buy groceries", completed: false}
   └─> Ingress routes response back
   └─> Browser receives data
   └─> React updates UI
   └─> You see "Buy groceries" in the list!
```

---

## 🏗️ **ARCHITECTURE DIAGRAM**
```
┌─────────────────────────────────────────────────────────────┐
│                    YOUR BROWSER                              │
│              http://127.0.0.1/                              │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        │ (minikube tunnel)
                        ▼
┌─────────────────────────────────────────────────────────────┐
│             MINIKUBE CLUSTER (192.168.49.2)                  │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         INGRESS CONTROLLER (Nginx)                   │   │
│  │  Rules:                                              │   │
│  │    /api/* → backend:8080                            │   │
│  │    /*     → frontend:3000                           │   │
│  └──────────────────┬──────────────┬────────────────────┘   │
│                     │              │                         │
│          ┌──────────▼──────┐   ┌──▼─────────────┐          │
│          │  Frontend Pod   │   │  Backend Pod    │          │
│          │  (React App)    │   │  (Node.js API)  │          │
│          │  Port: 3000     │   │  Port: 8080     │          │
│          │                 │   │                 │          │
│          │  Serves HTML/   │   │  /api/tasks    │          │
│          │  JS/CSS to      │   │  - GET         │          │
│          │  browser        │   │  - POST        │          │
│          │                 │   │  - PUT         │          │
│          └─────────────────┘   │  - DELETE      │          │
│                                 └────────┬────────┘          │
│                                          │                   │
│                                          │ MongoDB           │
│                                          │ Connection        │
│                                          ▼                   │
│                              ┌────────────────────┐         │
│                              │   MongoDB Pod      │         │
│                              │   Port: 27017      │         │
│                              │                    │         │
│                              │   Database: todo   │         │
│                              │   Collection:      │         │
│                              │     tasks          │         │
│                              └────────────────────┘         │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

---

## 🔑 **KEY CONCEPTS EXPLAINED**

### **1. Why Ingress Instead of NodePort?**

**NodePort Approach:**
```
Browser → Frontend NodePort (30007) → Frontend Pod
Browser → Backend NodePort (30080) → Backend Pod
```
❌ **Problems:**
- Two separate entry points
- CORS issues (different ports)
- Need to hardcode Minikube IP

**Ingress Approach:**
```
Browser → Ingress (80) → Routes to Frontend OR Backend
