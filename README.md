# Jenkins-CICD

Here's a detailed, step-by-step breakdown of how to create a **CI/CD pipeline for deploying an application to Kubernetes** using **Jenkins, SonarQube, Docker, and ArgoCD**.  

---  

## **Project Overview**  
### **Why do we create this project?**  
This project automates the process of building, testing, and deploying an application using CI/CD principles. Instead of manually building and deploying, the pipeline does it automatically whenever new code is pushed to the repository.  

### **What does this project do?**  
- It **builds** the application using **Maven**.  
- It **scans the code** for vulnerabilities using **SonarQube**.  
- It **packages** the application in a **Docker container** and pushes it to **DockerHub**.  
- It **deploys** the application to a **Kubernetes (K8s) cluster** using **ArgoCD**, which ensures continuous deployment.  

---

# **Step-by-Step Guide**  

## **1. Check the application locally**  
Before setting up the CI/CD pipeline, we should verify that the application works locally.  

### **Steps to run locally:**  
1. Clone the repository:  
   ```bash
   git clone <repo-url>
   cd <repo-name>
   ```
2. Install **Maven**:  
   ```bash
   sudo apt update
   sudo apt install maven -y
   mvn -version
   ```
3. Build and run a Docker image:  
   ```bash
   docker build -t my-app .
   docker run -d -p 8080:8080 my-app
   ```
4. Check the application:  
   Open a browser and visit:  
   ```
   http://localhost:8080
   ```
---
## **2. Set up a CI/CD Pipeline with Jenkins and SonarQube**  

### **Step 1: Launch an EC2 Instance (2GB RAM)**
1. Open AWS EC2 dashboard.  
2. Launch a **t2.small** instance with Ubuntu.  
3. Allow inbound traffic on ports **8080 (Jenkins), 9000 (SonarQube), 30007 (ArgoCD)**.  

### **Step 2: Install Jenkins**  
1. Update system packages:  
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```
2. Install Java (Required for Jenkins):  
   ```bash
   sudo apt install openjdk-17-jdk -y
   ```
3. Add Jenkins repository and install Jenkins:  
   ```bash
   wget -O /usr/share/keyrings/jenkins-keyring.asc \
   https://pkg.jenkins.io/debian/jenkins.io-2023.key  
   echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null  
   sudo apt update  
   sudo apt install jenkins -y  
   ```
4. Start and enable Jenkins:  
   ```bash
   sudo systemctl start jenkins
   sudo systemctl enable jenkins
   ```
5. Get the initial admin password:  
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
6. Open Jenkins in a browser:  
   ```
   http://<EC2-Public-IP>:8080
   ```
7. Install **Suggested Plugins**.  

### **Step 3: Configure Jenkins Pipeline**  
1. Go to **Jenkins Dashboard → New Item → Pipeline**  
2. Select **Pipeline Script from SCM**  
3. Choose **Git** and enter:  
   - Repository URL: `<repo-url>`  
   - Branch: `<branch-name>`  
   - Jenkinsfile Path: `Jenkinsfile`  

---

## **3. Install Docker and SonarQube on EC2**  

### **Step 1: Install Docker on EC2**
1. Install Docker:  
   ```bash
   sudo apt install docker.io -y
   ```
2. Start and enable Docker:  
   ```bash
   sudo systemctl start docker
   sudo systemctl enable docker
   ```
3. Grant Jenkins user permission to run Docker:  
   ```bash
   sudo usermod -aG docker jenkins
   sudo systemctl restart jenkins
   ```
4. Restart Docker:  
   ```bash
   sudo systemctl restart docker
   ```

### **Step 2: Install SonarQube**
1. Install dependencies:  
   ```bash
   sudo apt update && sudo apt install unzip -y
   ```
2. Create a SonarQube user:  
   ```bash
   sudo adduser sonarqube
   ```
3. Download SonarQube:  
   ```bash
   wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-10.4.1.88267.zip
   ```
4. Extract and configure:  
   ```bash
   unzip sonarqube-10.4.1.88267.zip
   sudo mv sonarqube-10.4.1.88267 /opt/sonarqube
   sudo chown -R sonarqube:sonarqube /opt/sonarqube
   sudo chmod -R 775 /opt/sonarqube
   ```
5. Start SonarQube:  
   ```bash
   cd /opt/sonarqube/bin/linux-x86-64
   ./sonar.sh start
   ```
6. Open SonarQube in a browser:  
   ```
   http://<EC2-Public-IP>:9000
   ```
7. Login (default credentials: `admin / admin`) and generate a **token**.

---

## **4. Configure Jenkins with SonarQube**  
1. In Jenkins, go to **Manage Jenkins → Manage Credentials**  
2. Add **Secret Text** (SonarQube Token).  
3. Install **SonarQube Scanner Plugin**.  
4. Update SonarQube URL in the **Jenkinsfile**:  
   ```yaml
   sonarqube {
      properties(['sonar.projectKey': 'my-app'])
   }
   ```
---

## **5. Install Minikube and ArgoCD**  
### **Step 1: Install Minikube on Local Machine**  
1. Install dependencies:  
   ```bash
   sudo apt install curl -y
   curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
   sudo install minikube-linux-amd64 /usr/local/bin/minikube
   ```
2. Start Minikube:  
   ```bash
   minikube start
   ```

### **Step 2: Install ArgoCD**
1. Install ArgoCD Operator:  
   ```bash
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```
2. Access ArgoCD UI:  
   ```bash
   kubectl port-forward svc/argocd-server -n argocd 30007:443
   ```
3. Open a browser:  
   ```
   https://localhost:30007
   ```
4. Login to ArgoCD (`admin` / default password from `kubectl get secrets`).

---

## **6. Deploy the Application using ArgoCD**  
### **Steps to Configure Deployment**  
1. Change **service type** from `ClusterIP` to `NodePort` in `deployment.yaml`.  
2. Push changes to the GitHub repository.  
3. In ArgoCD, **create a new application**, connect to GitHub, and enable auto-sync.  
4. The application will be deployed automatically in Kubernetes.  

---

## **7. Verify CI/CD Pipeline**
1. Click **Build** in Jenkins.  
2. Check if:  
   ✅ Docker image is pushed to **DockerHub**.  
   ✅ ArgoCD deployed the application.  
   ✅ The application is running in Kubernetes.  
3. Access the application via:  
   ```
   http://<Minikube-IP>:<NodePort>
   ```

---

## **Conclusion**  
By following this, you now have an **end-to-end automated CI/CD pipeline** using **Jenkins, SonarQube, Docker, and ArgoCD** that continuously deploys an application to **Kubernetes**! 🚀


