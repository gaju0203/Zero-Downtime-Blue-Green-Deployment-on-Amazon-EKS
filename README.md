## Kubernetes Blue-Green & Canary Deployment on AWS EKS 🚀

### 📌 Project Overview
This project demonstrates a **production-level Kubernetes deployment strategy** using **Blue-Green and Canary deployments** on **AWS EKS**.

The goal is to achieve:
- ✅ Zero downtime deployment
- ✅ Safe release of new versions
- ✅ Easy rollback
- ✅ Visual verification using different `index.html` pages

This project is implemented using:
- **AWS EKS**
- **Ubuntu EC2 (as DevOps jump server)**
- **Kubernetes Deployments, Services, ConfigMaps**
- **NGINX web server**

---

### Architecture Diagram (Conceptual)

User <br>
↓   <br>
AWS LoadBalancer Service      <br>
↓      <br>
Kubernetes Service      <br>
↓         <br>
Blue Pods (v1) ← Stable Version              <br>
Green Pods (v2) ← Canary Version                  <br>


Traffic is split based on the **number of running pods**.

![](./images/Artitecture.png)
---

### Tools & Technologies Used

| Tool | Purpose |
|---|---|
| AWS EC2 (Ubuntu) | DevOps jump server |
| AWS EKS | Managed Kubernetes cluster |
| kubectl | Kubernetes CLI |
| eksctl | EKS cluster creation |
| NGINX | Web server |
| ConfigMaps | Inject different index.html files |
| LoadBalancer Service | Public access |

---

### Project Structure
eks-blue-green-canary/         <br>
│                          <br>
├── blue-deployment.yaml            <br>
├── green-canary.yaml                  <br>
├── service.yaml                     <br>
├── blue-index.html                 <br>
├── green-index.html                <br>
└── README.md


---

### Step 1: Launch Ubuntu EC2 (Jump Server)

1. Launch **Ubuntu Server 22.04 LTS**
2. Instance type: `t3.medium`
3. Allow ports: `22`, `80`
4. Attach IAM role with `AdministratorAccess`
5. Connect using SSH

```bash
ssh -i eks-key.pem ubuntu@<EC2_PUBLIC_IP>
```
---
### Step 2: Install Required Tools on EC2
__Update system__
```bash
sudo apt update && sudo apt upgrade -y
```

__Install AWS CLI__
```bash
sudo apt install unzip curl -y
curl https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o awscliv2.zip
unzip awscliv2.zip
sudo ./aws/install
```

__Install kubectl__
```bash
curl -LO https://dl.k8s.io/release/v1.29.0/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

__Install eksctl__
```bash
curl -sLO https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz
tar -xzf eksctl_Linux_amd64.tar.gz
sudo mv eksctl /usr/local/bin/
```
---
### Step 3: Create AWS EKS Cluster
```
eksctl create cluster \
--name blue-green-cluster \
--region us-east-1 \
--nodegroup-name worker-nodes \
--node-type t3.medium \
--nodes 2
```
![](./images/Screenshot%20(232).png)

__Verify:__
```
kubectl get nodes
```
![](./images/Screenshot%20(233).png)

### Step 4: Create Blue Deployment (Stable Version)

```blue-deployment.yaml```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blue-app
spec:
  replicas: 4
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        volumeMounts:
        - name: blue-html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: blue-html
        configMap:
          name: blue-html
```
### Step 5: Create Green Canary Deployment (New Version)

```green-canary.yaml```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: green-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: nginx
        image: nginx:1.26
        ports:
        - containerPort: 80
        volumeMounts:
        - name: green-html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: green-html
        configMap:
          name: green-html
```
---
### Step 6: Create index.html Files
__Blue Version__
```
<!DOCTYPE html>
<html>
<head>
  <title>Blue Version</title>
</head>
<body style="background-color: lightblue;">
  <h1>🔵 BLUE VERSION</h1>
  <p>This is Blue (Stable) Deployment</p>
</body>
</html>
```
__Green Version__
```
<!DOCTYPE html>
<html>
<head>
  <title>Green Version</title>
</head>
<body style="background-color: lightgreen;">
  <h1>🟢 GREEN VERSION</h1>
  <p>This is Green (Canary) Deployment</p>
</body>
</html>
```
---
### Step 7: Create ConfigMaps
```
kubectl create configmap blue-html --from-file=index.html=blue-index.html
kubectl create configmap green-html --from-file=index.html=green-index.html
```
---
### Step 8: Create Kubernetes Service

```service.yaml```
```
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 80
  type: LoadBalancer
```
Apply all resources:
```
kubectl apply -f servivce.yml
```
```
kubectl get all
```
![](./images/Screenshot%20(235).png)
---
### Step 9: Access Application
```
kubectl get svc myapp-service
```
![](./images/Screenshot%20(236).png)

Open in browser:
```
http://<EXTERNAL-IP>
```

Refresh multiple times to see Blue and Green pages.

---
### Step 10: Canary Traffic Control
Initial Canary

Blue: 4 pods

Green: 1 pod

Increase Canary to 50%
```
kubectl scale deployment green-canary --replicas=3
kubectl scale deployment blue-app --replicas=3
```
![](./images/Screenshot%20(237).png)
Full Promotion
```
kubectl scale deployment green-canary --replicas=6
kubectl scale deployment blue-app --replicas=0
```
![](./images/Screenshot%20(238).png)
![](./images/Screenshot%20(241).png)
![](./images/Screenshot%20(242).png)

### Step 11: Rollback (Zero Downtime)
```
kubectl scale deployment green-canary --replicas=0
kubectl scale deployment blue-app --replicas=4
```
Cleanup
```
eksctl delete cluster --name blue-green-cluster --region ap-south-1
```

### Conclusion

This project simulates a real-world production deployment strategy used by DevOps teams.

It combines AWS, Kubernetes, and release engineering best practices, making it resume and interview ready.

