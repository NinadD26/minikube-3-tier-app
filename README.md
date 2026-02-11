README.md
# Three-Tier Application on Minikube (MongoDB + Backend + Frontend)

This project deploys a **3-tier web application** on Kubernetes using **Minikube**.

## Stack
- **MongoDB** – Database tier  
- **Node.js Backend API** – Application tier  
- **React Frontend** – Presentation tier  
- **NGINX Ingress** – Traffic routing

---

## Architecture

Browser  
→ NGINX Ingress (`http://127.0.0.1`)  
→ Frontend Service  
→ Backend Service  
→ MongoDB Service  

---

## Prerequisites

- Docker  
- kubectl  
- Minikube  

Start Minikube:

```bash
minikube start
Verify:

bash
Copy code
kubectl get nodes
Step 1: Create Namespace
bash
Copy code
kubectl create namespace three-tier
Step 2: Deploy MongoDB
bash
Copy code
cd k8s_manifests/mongo

kubectl apply -f secrets.yaml -n three-tier
kubectl apply -f deploy.yaml -n three-tier
kubectl apply -f service.yaml -n three-tier
Verify:

bash
Copy code
kubectl get pods -n three-tier
kubectl get svc -n three-tier
Step 3: Deploy Backend
bash
Copy code
cd ../
kubectl apply -f backend-deployment.yaml -n three-tier
kubectl apply -f backend-service.yaml -n three-tier
Step 4: Deploy Frontend
bash
Copy code
kubectl apply -f frontend-deployment.yaml -n three-tier
kubectl apply -f frontend-service.yaml -n three-tier
Step 5: Enable Ingress in Minikube
bash
Copy code
minikube addons enable ingress
Verify:

bash
Copy code
kubectl get pods -n ingress-nginx
Step 6: Apply Ingress
bash
Copy code
kubectl apply -f minikube-ingress.yaml
Verify:

bash
Copy code
kubectl get ingress -n three-tier
Step 7: Start Tunnel
Keep this terminal open:

bash
Copy code
minikube tunnel
App is available at:

text
Copy code
http://127.0.0.1
Testing
Open browser:

text
Copy code
http://127.0.0.1
Add a task → refresh → task should persist.

Configuration Fixes
Frontend API Fix
frontend-deployment.yaml

yaml
Copy code
- name: REACT_APP_BACKEND_URL
  value: "/api/tasks"
Ingress Routing
minikube-ingress.yaml

yaml
Copy code
apiVersion: networking.k8s.io/v1
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
MongoDB Auth Fix
backend-deployment.yaml

yaml
Copy code
value: mongodb://admin:password123@mongodb-svc:27017/todo?authSource=admin&directConnection=true
Common Issues
Issue	Fix
No External IP	Run minikube tunnel
Cannot GET /	Check Ingress paths
Backend not connecting	Fix MongoDB auth URL

Final Verification

kubectl get pods -n three-tier
kubectl get svc -n three-tier
kubectl get ingress -n three-tier
All pods should be Running.

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

┌─────────────────────────────────────────────────────────────┐
│                    YOUR BROWSER                             │
│              http://127.0.0.1/                              │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        │ (minikube tunnel)
                        ▼
┌─────────────────────────────────────────────────────────────┐
│             MINIKUBE CLUSTER (192.168.49.2)                 │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         INGRESS CONTROLLER (Nginx)                   │   │
│  │  Rules:                                              │   │
│  │    /api/* → backend:8080                             │   │
│  │    /*     → frontend:3000                            │   │
│  └──────────────────┬──────────────┬────────────────────┘   │
│                     │              │                         │
│          ┌──────────▼──────┐   ┌──▼─────────────┐           │
│          │  Frontend Pod   │   │  Backend Pod    │           │
│          │  (React App)    │   │  (Node.js API)  │           │
│          │  Port: 3000     │   │  Port: 8080     │           │
│          │                 │   │  /api/tasks     │           │
│          │  Serves HTML/   │   │  - GET          │           │
│          │  JS/CSS to      │   │  - POST         │           │
│          │  browser        │   │  - PUT          │           │
│          │                 │   │  - DELETE       │           │
│          └─────────────────┘   └────────┬────────┘           │
│                                          │                    │
│                                          ▼                    │
│                              ┌────────────────────┐           │
│                              │   MongoDB Pod      │           │
│                              │   Port: 27017      │           │
│                              │   Database: todo   │           │
│                              │   Collection:tasks │           │
│                              └────────────────────┘           │
└───────────────────────────────────────────────────────────────┘
