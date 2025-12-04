# Flask-MongoDB Kubernetes Deployment

## 📌 Problem Statement
Deploy a Python Flask application and a MongoDB database on a Kubernetes cluster with:
- Flask app supporting GET/POST endpoints  
- MongoDB with authentication  
- Persistent data storage  
- Autoscaling (HPA)  
- Resource limits & requests  
- Internal DNS communication  
- Minikube-compatible manifests

---
  
# 🧰 Tech Stack
Backend: Python Flask

Database: MongoDB (with authentication)

Containerization: Docker

Orchestration: Kubernetes (Deployments, StatefulSet, PVC, Services, HPA)

Environment Management: ConfigMaps, Secrets

Tools: Minikube, kubectl

---

# 🏗️ Architecture Diagram

<img width="956" height="724" alt="image" src="https://github.com/user-attachments/assets/cb7c24d3-37e1-4b72-b01c-a9adbcefc956" />

<img width="1388" height="753" alt="image" src="https://github.com/user-attachments/assets/8093f22d-7214-4414-9146-aa8e0b7b98bb" />

### Components
1. **Flask Application**: Web server with 2-5 replicas (autoscaled)
2. **MongoDB**: StatefulSet with persistent storage and authentication
3. **Services**: NodePort for Flask, ClusterIP for MongoDB
4. **HPA**: Scales Flask pods based on 70% CPU usage

### DNS Resolution in Kubernetes

Kubernetes provides internal DNS resolution through CoreDNS:

**How it works:**
- Each Service gets a DNS name: `<service-name>.<namespace>.svc.cluster.local`
- Pods can use short names within the same namespace: `mongodb-service`
- DNS queries are resolved by CoreDNS running in kube-system namespace

**In this project:**
- Flask connects to MongoDB using: `mongodb://admin:password123@mongodb-service:27017/`
- `mongodb-service` resolves to the MongoDB StatefulSet pods
- This is configured in `flask-configmap.yaml`

**DNS Resolution Flow:**
1. Flask pod makes request to `mongodb-service`
2. Pod's DNS resolver (at `/etc/resolv.conf`) forwards to CoreDNS
3. CoreDNS looks up Service name in its records
4. Returns ClusterIP of mongodb-service
5. Kubernetes networking routes traffic to MongoDB pod

### Resource Requests and Limits

**Purpose:**
- **Requests**: Guaranteed resources for scheduling. Kubernetes ensures node has these resources before placing pod.
- **Limits**: Maximum resources pod can use. Prevents resource exhaustion.

**Configuration (both Flask and MongoDB):**
```yaml
resources:
  requests:
    cpu: "200m"      # 0.2 CPU cores guaranteed
    memory: "250Mi"  # 250 MB guaranteed
  limits:
    cpu: "500m"      # Max 0.5 CPU cores
    memory: "500Mi"  # Max 500 MB
```

**Benefits:**
- Prevents pods from consuming all node resources
- Enables efficient bin-packing on nodes
- HPA uses requests as baseline for scaling decisions
- OOM killer uses limits to terminate excessive pods

---

## 🚀 Run Locally

### 1. Create Virtual Environment
```bash
python3 -m venv venv
source venv/bin/activate
```

### 2. Install Dependencies
```bash
pip install -r requirements.txt
```

### 3. Run MongoDB using Docker
```bash
docker run -d -p 27017:27017 --name mongodb mongo:latest
```

### 4. Set Environment Variable
```bash
export MONGODB_URI=mongodb://localhost:27017
```

### 5. Start Flask App
```bash
flask run
```

---

## 🐳 Docker Build & Push
```bash
docker build -t <your-dockerhub-username>/flask-app:latest .
docker push <your-dockerhub-username>/flask-app:latest
```

---

## ☸️ Kubernetes Deployment Steps

### 1. Start Minikube
```bash
minikube start --driver=docker
minikube addons enable metrics-server
```
### 2. Build Docker Image
```bash
eval $(minikube docker-env)
docker build -t flask-mongodb-app:v1 .
```

### 3. Deploy to Kubernetes
```bash
cd k8s
kubectl apply -f mongodb-secret.yaml
kubectl apply -f mongodb-pv.yaml
kubectl apply -f mongodb-pvc.yaml
kubectl apply -f mongodb-statefulset.yaml
kubectl apply -f mongodb-service.yaml
kubectl wait --for=condition=ready pod -l app=mongodb --timeout=120s
kubectl apply -f flask-configmap.yaml
kubectl apply -f flask-deployment.yaml
kubectl apply -f flask-service.yaml
kubectl apply -f flask-hpa.yaml
```
### 4. Access Application
```bash
minikube service flask-service --url
# Or
kubectl port-forward service/flask-service 5000:5000
```

## Testing Scenarios

### 1. Basic Functionality Test
```bash
# Test root endpoint
curl http://$(minikube ip):30080/

# Insert data
curl -X POST -H "Content-Type: application/json" \
  -d '{"test":"data"}' http://$(minikube ip):30080/data

# Retrieve data
curl http://$(minikube ip):30080/data
```

**Expected:** All endpoints respond correctly, data persists in MongoDB

### 2. Database Persistence Test
```bash
# Insert data
curl -X POST -H "Content-Type: application/json" \
  -d '{"persistent":"test"}' http://$(minikube ip):30080/data

# Delete MongoDB pod
kubectl delete pod mongodb-0

# Wait for recreation
kubectl wait --for=condition=ready pod/mongodb-0 --timeout=120s

# Verify data still exists
curl http://$(minikube ip):30080/data
```

**Result:** ✅ Data persisted after pod deletion
**Observation:** PVC ensures data survives pod restarts

### 3. Autoscaling Test
```bash
# Deploy load generator
kubectl apply -f load-test.yaml

# Monitor HPA
kubectl get hpa --watch

# Monitor pods
kubectl get pods --watch

# Check CPU usage
kubectl top pods
```

**Results:**
- Initial state: 2 replicas, ~10% CPU usage
- Under load: CPU rose to ~85%
- Scaling triggered: HPA created 3rd replica at 2min mark
- Peak: 4 replicas at ~75% average CPU
- Cool down: After stopping load, scaled down to 2 replicas in ~5min

**Issues Encountered:**
1. **Metrics not available initially**: Metrics-server addon wasn't enabled
   - **Solution**: `minikube addons enable metrics-server`

2. **HPA showed "unknown" CPU**: Pods didn't have resource requests defined
   - **Solution**: Added resource requests to deployment

3. **Slow scaling**: Default behavior has delays
   - **Observation**: This is intentional to prevent flapping

### 4. DNS Resolution Test
```bash
# Exec into Flask pod
kubectl exec -it  -- /bin/sh

# Install ping
apk add --no-cache bind-tools

# Test DNS resolution
nslookup mongodb-service

# Test connection
nc -zv mongodb-service 27017
```

**Result:** ✅ DNS correctly resolves mongodb-service to ClusterIP

### 5. Authentication Test
```bash
# Try connecting without auth (should fail)
kubectl run -it --rm mongo-client --image=mongo:5.0 --restart=Never \
  -- mongo mongodb-service:27017

# Try with correct credentials (should succeed)
kubectl run -it --rm mongo-client --image=mongo:5.0 --restart=Never \
  -- mongo "mongodb://admin:password123@mongodb-service:27017"
```

**Result:** ✅ Authentication working correctly

## Monitoring Commands
```bash
# View all resources
kubectl get all

# Check HPA status
kubectl get hpa

# View pod resource usage
kubectl top pods

# View pod logs
kubectl logs -l app=flask-app --tail=50
kubectl logs mongodb-0

# Describe resources
kubectl describe pod 
kubectl describe hpa flask-hpa
```
## Cleanup
```bash
kubectl delete -f k8s/
minikube stop
minikube delete
```

## Troubleshooting

### Pods not starting
```bash
kubectl describe pod 
kubectl logs 
```

### MongoDB connection issues
```bash
kubectl logs -l app=flask-app | grep -i mongo
kubectl exec -it mongodb-0 -- mongo -u admin -p password123
```
### HPA not working
```bash
kubectl get hpa
kubectl top pods
kubectl describe hpa flask-hpa
```

## 🤝 Contribution Guidelines
1. Fork the repository  
2. Create a feature branch  
3. Commit changes  
4. Open a Pull Request  
5. Provide clear explanation of changes  

---

## 📜 License
This project is open-source under the MIT License.

---

## 🧑‍💻 Author
Ayush Singh
https://github.com/Ayush-silicon






