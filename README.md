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
minikube start
```

### 2. Apply MongoDB Resources
```bash
kubectl apply -f k8s/mongo-secret.yaml
kubectl apply -f k8s/mongo-pvc.yaml
kubectl apply -f k8s/mongo-statefulset.yaml
kubectl apply -f k8s/mongo-service.yaml
```

### 3. Deploy Flask App
```bash
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/flask-deployment.yaml
kubectl apply -f k8s/flask-service.yaml
```

### 4. Enable Autoscaling
```bash
kubectl apply -f k8s/hpa.yaml
```

### 5. Access Application
```bash
minikube service flask-service
```

---

## 🌐 DNS Resolution in Kubernetes
Kubernetes internal DNS resolves service names automatically.  
Flask connects to MongoDB using:

```
mongodb://<user>:<password>@mongo-service:27017/
```

`mongo-service` is resolved by CoreDNS to the MongoDB Pod IP.

---

## 📊 Resource Requests & Limits
Example configuration:
```yaml
resources:
  requests:
    cpu: "0.2"
    memory: "250Mi"
  limits:
    cpu: "0.5"
    memory: "500Mi"
```

---

## 🔄 Autoscaling (HPA Testing)
Simulate high load:
```bash
kubectl run loadgen --image=busybox -- sh -c "while true; do wget -q -O- http://flask-service:5000; done"
```

HPA scales replicas from **2 → 5** when CPU > 70%.

---

## 📸 Screenshots
(Add screenshots of your app, terminal, and Kubernetes dashboard)

---

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






