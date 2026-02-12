Prerequisites

Minikube installed and running
kubectl installed
Docker Desktop running
Windows PowerShell (Administrator access for Helm installation)


Step 1: Install Helm
Install Helm on Windows
Open PowerShell as Administrator and run:
powershell# Download Helm
Invoke-WebRequest -Uri https://get.helm.sh/helm-v3.14.0-windows-amd64.zip -OutFile helm.zip

# Extract the archive
Expand-Archive -Path helm.zip -DestinationPath C:\helm

# Move helm.exe to system PATH
Move-Item -Path C:\helm\windows-amd64\helm.exe -Destination C:\Windows\System32\helm.exe -Force

# Clean up
Remove-Item helm.zip
Remove-Item -Recurse C:\helm
Verify Helm Installation
powershellhelm version
```

**Expected Output:**
```
version.BuildInfo{Version:"v3.14.0", GitCommit:"...", GitTreeState:"clean", GoVersion:"go1.21.5"}

Step 2: Create Helm Chart Structure
Navigate to Project Directory
powershellcd D:\Three-tier-Application-Deployment
Create Helm Chart
powershell# Create helm-chart directory
mkdir helm-chart
cd helm-chart

# Create chart structure using Helm
helm create three-tier-app

# Navigate to templates directory
cd three-tier-app\templates

# Remove default generated files
Remove-Item * -Recurse -Force

# Go back to chart root
cd ..
```

### Verify Directory Structure
```
Three-tier-Application-Deployment/
├── backend/
├── frontend/
├── k8s_manifests/
└── helm-chart/
    └── three-tier-app/
        ├── Chart.yaml
        ├── values.yaml
        ├── charts/
        └── templates/

Step 3: Create Chart Files
3.1 Edit Chart.yaml
Location: helm-chart/three-tier-app/Chart.yaml
Replace content with:
yamlapiVersion: v2
name: three-tier-app
description: A Helm chart for Three-Tier Todo Application
type: application
version: 1.0.0
appVersion: "1.0"

3.2 Edit values.yaml
Location: helm-chart/three-tier-app/values.yaml
Replace content with:
yamlnamespace: three-tier

mongodb:
  enabled: true
  image:
    repository: mongo
    tag: "4.4.6"
  auth:
    username: admin
    password: password123
    database: todo
  service:
    name: mongodb-svc
    port: 27017
  resources:
    requests:
      memory: "512Mi"
      cpu: "250m"
    limits:
      memory: "1Gi"
      cpu: "500m"

backend:
  enabled: true
  replicaCount: 1
  image:
    repository: ninad04/todo-backend
    tag: "1.0"
  service:
    name: backend
    port: 8080

frontend:
  enabled: true
  replicaCount: 1
  image:
    repository: ninad04/todo-frontend
    tag: "1.0"
  service:
    name: frontend
    port: 3000
  env:
    backendUrl: "/api/tasks"

ingress:
  enabled: true
  className: nginx

3.3 Create Template Files
Navigate to templates directory:
powershellcd helm-chart\three-tier-app\templates
Create the following files:
templates/namespace.yaml
yamlapiVersion: v1
kind: Namespace
metadata:
  name: {{ .Values.namespace }}
templates/mongodb-secret.yaml
yaml{{- if .Values.mongodb.enabled }}
apiVersion: v1
kind: Secret
metadata:
  name: mongo-sec
  namespace: {{ .Values.namespace }}
type: Opaque
data:
  username: {{ .Values.mongodb.auth.username | b64enc | quote }}
  password: {{ .Values.mongodb.auth.password | b64enc | quote }}
{{- end }}
templates/mongodb-deployment.yaml
yaml{{- if .Values.mongodb.enabled }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
  namespace: {{ .Values.namespace }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: "{{ .Values.mongodb.image.repository }}:{{ .Values.mongodb.image.tag }}"
        ports:
        - containerPort: {{ .Values.mongodb.service.port }}
        resources:
          {{- toYaml .Values.mongodb.resources | nindent 10 }}
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongo-sec
              key: username
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongo-sec
              key: password
{{- end }}
templates/mongodb-service.yaml
yaml{{- if .Values.mongodb.enabled }}
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.mongodb.service.name }}
  namespace: {{ .Values.namespace }}
spec:
  selector:
    app: mongodb
  ports:
  - name: mongodb
    port: {{ .Values.mongodb.service.port }}
    targetPort: {{ .Values.mongodb.service.port }}
  type: ClusterIP
{{- end }}
templates/backend-deployment.yaml
yaml{{- if .Values.backend.enabled }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: {{ .Values.namespace }}
  labels:
    role: backend
spec:
  replicas: {{ .Values.backend.replicaCount }}
  selector:
    matchLabels:
      role: backend
  template:
    metadata:
      labels:
        role: backend
    spec:
      containers:
      - name: backend
        image: "{{ .Values.backend.image.repository }}:{{ .Values.backend.image.tag }}"
        imagePullPolicy: Always
        env:
        - name: MONGO_CONN_STR
          value: "mongodb://{{ .Values.mongodb.auth.username }}:{{ .Values.mongodb.auth.password }}@{{ .Values.mongodb.service.name }}:{{ .Values.mongodb.service.port }}/{{ .Values.mongodb.auth.database }}?authSource=admin&directConnection=true"
        - name: MONGO_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongo-sec
              key: username
        - name: MONGO_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongo-sec
              key: password
        ports:
        - containerPort: {{ .Values.backend.service.port }}
        livenessProbe:
          httpGet:
            path: /ok
            port: {{ .Values.backend.service.port }}
          initialDelaySeconds: 2
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /ok
            port: {{ .Values.backend.service.port }}
          initialDelaySeconds: 5
          periodSeconds: 5
{{- end }}
templates/backend-service.yaml
yaml{{- if .Values.backend.enabled }}
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.backend.service.name }}
  namespace: {{ .Values.namespace }}
spec:
  ports:
  - port: {{ .Values.backend.service.port }}
    targetPort: {{ .Values.backend.service.port }}
  type: ClusterIP
  selector:
    role: backend
{{- end }}
templates/frontend-deployment.yaml
yaml{{- if .Values.frontend.enabled }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: {{ .Values.namespace }}
  labels:
    role: frontend
spec:
  replicas: {{ .Values.frontend.replicaCount }}
  selector:
    matchLabels:
      role: frontend
  template:
    metadata:
      labels:
        role: frontend
    spec:
      containers:
      - name: frontend
        image: "{{ .Values.frontend.image.repository }}:{{ .Values.frontend.image.tag }}"
        imagePullPolicy: Always
        env:
        - name: REACT_APP_BACKEND_URL
          value: {{ .Values.frontend.env.backendUrl | quote }}
        ports:
        - containerPort: {{ .Values.frontend.service.port }}
{{- end }}
templates/frontend-service.yaml
yaml{{- if .Values.frontend.enabled }}
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.frontend.service.name }}
  namespace: {{ .Values.namespace }}
spec:
  ports:
  - port: {{ .Values.frontend.service.port }}
    targetPort: {{ .Values.frontend.service.port }}
  type: ClusterIP
  selector:
    role: frontend
{{- end }}
templates/ingress.yaml
yaml{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: three-tier-ingress
  namespace: {{ .Values.namespace }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
  rules:
  - http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: {{ .Values.backend.service.name }}
            port:
              number: {{ .Values.backend.service.port }}
      - path: /
        pathType: Prefix
        backend:
          service:
            name: {{ .Values.frontend.service.name }}
            port:
              number: {{ .Values.frontend.service.port }}
{{- end }}

Step 4: Validate Helm Chart
powershell# Navigate to helm-chart directory
cd D:\Three-tier-Application-Deployment\helm-chart

# Lint the chart
helm lint three-tier-app
```

**Expected Output:**
```
==> Linting three-tier-app
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed

Step 5: Deploy Application
5.1 Start Minikube
powershellminikube start
5.2 Enable Ingress Addon
powershellminikube addons enable ingress
5.3 Install Helm Chart
powershell# Navigate to helm-chart directory
cd D:\Three-tier-Application-Deployment\helm-chart

# Install the chart
helm install my-todo-app three-tier-app
```

**Expected Output:**
```
NAME: my-todo-app
LAST DEPLOYED: Thu Feb 12 10:57:01 2026
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
5.4 Verify Deployment
powershell# Check pods (wait until all show 1/1 Running)
kubectl get pods -n three-tier --watch
Press Ctrl+C when all pods are running.
powershell# Check all resources
kubectl get all -n three-tier

# Check ingress
kubectl get ingress -n three-tier

# Check Helm release
helm list

Step 6: Access and Test Application
6.1 Start Minikube Tunnel
Open a NEW PowerShell window and run:
powershellminikube tunnel
Keep this window running - don't close it!
6.2 Test Backend API
powershellcurl.exe http://127.0.0.1/api/tasks
```

**Expected Output:**
```
[]
```

### 6.3 Access Frontend

Open your browser and navigate to:
```
http://127.0.0.1
6.4 Test Application Functionality

Add a Task:

Type "Deploy with Helm" in the input box
Click "ADD TASK"
Task should appear in the list


Complete a Task:

Click the checkbox next to the task
Task text should have strikethrough


Delete a Task:

Click the "Delete" button
Task should be removed


Verify Persistence:

Refresh the browser page
Tasks should still be there (stored in MongoDB)




Step 7: Manage Deployment
View Helm Release Information
powershell# List all releases
helm list

# Get release status
helm status my-todo-app

# Get values used in deployment
helm get values my-todo-app

# Get all values including defaults
helm get values my-todo-app --all

# View deployment history
helm history my-todo-app
Upgrade Deployment
To change configuration (e.g., scale frontend):
Edit values.yaml:
yamlfrontend:
  replicaCount: 2  # Changed from 1 to 2
Apply changes:
powershellhelm upgrade my-todo-app three-tier-app
Or use command line:
powershellhelm upgrade my-todo-app three-tier-app --set frontend.replicaCount=2
Rollback Deployment
powershell# Rollback to previous version
helm rollback my-todo-app

# Rollback to specific revision
helm rollback my-todo-app 1

Step 8: Cleanup
8.1 Delete Helm Release
powershell# Uninstall the application
helm uninstall my-todo-app
```

**Expected Output:**
```
release "my-todo-app" uninstalled
8.2 Verify Deletion
powershell# Check Helm releases
helm list

# Check namespace resources (should be empty or namespace deleted)
kubectl get all -n three-tier
8.3 Delete Namespace (Optional)
powershellkubectl delete namespace three-tier
8.4 Stop Minikube Tunnel
In the tunnel window, press Ctrl+C to stop the tunnel.
8.5 Stop Minikube (Optional)
powershellminikube stop
8.6 Delete Minikube Cluster (Complete Reset)
powershellminikube delete

Helm Commands Reference
Installation Commands
powershell# Install a chart
helm install <release-name> <chart-path>

# Install with custom values file
helm install <release-name> <chart-path> -f custom-values.yaml

# Install with inline value overrides
helm install <release-name> <chart-path> --set key=value

# Dry run (see what would be deployed)
helm install <release-name> <chart-path> --dry-run --debug
Management Commands
powershell# List releases
helm list
helm list -a  # Include uninstalled releases

# Get release status
helm status <release-name>

# Get release values
helm get values <release-name>
helm get values <release-name> --all

# Get release manifest
helm get manifest <release-name>

# View release history
helm history <release-name>
Update Commands
powershell# Upgrade release
helm upgrade <release-name> <chart-path>

# Upgrade with new values
helm upgrade <release-name> <chart-path> -f new-values.yaml

# Upgrade with inline values
helm upgrade <release-name> <chart-path> --set key=newvalue

# Rollback to previous version
helm rollback <release-name>

# Rollback to specific revision
helm rollback <release-name> <revision-number>
Deletion Commands
powershell# Uninstall release
helm uninstall <release-name>

# Uninstall and keep history
helm uninstall <release-name> --keep-history
Development Commands
powershell# Create new chart
helm create <chart-name>

# Lint chart
helm lint <chart-path>

# Template chart (see generated YAML)
helm template <chart-path>
helm template <release-name> <chart-path>

# Package chart
helm package <chart-path>

# Test chart installation
helm test <release-name>

Quick Deployment Reference
Complete Deployment Flow
powershell# 1. Start Minikube
minikube start

# 2. Enable Ingress
minikube addons enable ingress

# 3. Navigate to helm-chart directory
cd D:\Three-tier-Application-Deployment\helm-chart

# 4. Validate chart
helm lint three-tier-app

# 5. Install application
helm install my-todo-app three-tier-app

# 6. Check deployment
kubectl get pods -n three-tier --watch

# 7. Start tunnel (in new window)
minikube tunnel

# 8. Access application
start http://127.0.0.1
Complete Cleanup Flow
powershell# 1. Uninstall application
helm uninstall my-todo-app

# 2. Delete namespace
kubectl delete namespace three-tier

# 3. Stop tunnel (Ctrl+C in tunnel window)

# 4. Stop Minikube
minikube stop

Troubleshooting
Chart Validation Errors
powershell# Check chart syntax
helm lint three-tier-app

# See generated YAML
helm template three-tier-app

# Debug installation
helm install my-todo-app three-tier-app --dry-run --debug
Deployment Issues
powershell# Check pod status
kubectl get pods -n three-tier

# Check pod logs
kubectl logs <pod-name> -n three-tier

# Describe pod
kubectl describe pod <pod-name> -n three-tier

# Check events
kubectl get events -n three-tier --sort-by='.lastTimestamp'
Application Not Accessible
powershell# Verify tunnel is running
# (Check if minikube tunnel window is open)

# Check ingress
kubectl get ingress -n three-tier

# Test backend directly
kubectl port-forward -n three-tier svc/backend 8080:8080
curl.exe http://localhost:8080/api/tasks
