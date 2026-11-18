Alright. Let's prepare this as an **exam-ready practical guide for all 4 questions**, including:

1. Exact steps and commands to execute.
2. Files you need to create and their contents.
3. What output to expect and how to verify success.
4. Answers to every scenario-based/theoretical question.

I'll keep it practical. No unnecessary explanations or extra installations unless required.

# QUESTION 1: Docker Compose – Multi-Container Deployment with Nagios (25 Marks)

## Part A: Practical Execution

### Step 1: Create a project folder

Open PowerShell or VS Code terminal.

```bash
mkdir docker-nagios
cd docker-nagios
```

### Step 2: Create docker-compose.yml

Create a file named:

`docker-compose.yml`

Paste this:

```yaml
services:

  web:
    image: nginx:alpine
    container_name: web-app
    ports:
      - "8081:80"
    networks:
      - monitoring

  nagios:
    image: jasonrivers/nagios:latest
    container_name: nagios
    ports:
      - "8080:80"
    networks:
      - monitoring
    depends_on:
      - web

networks:
  monitoring:
    driver: bridge
```

**What are we doing?**

| Service | Image | Port |
|---|---|---|
| Web application | nginx:alpine | 8081 |
| Nagios | jasonrivers/nagios | 8080 |

Both containers communicate through the same Docker network.

Here:
- `8081:80` means host port 8081 maps to container port 80.
- `8080:80` means host port 8080 maps to container port 80.
- `depends_on` starts the web service before Nagios.

We are using Nginx as the sample web application from Docker Hub. If your examiner provides another image, replace `nginx:alpine` with that image.

### Step 3: Start both containers

```bash
docker compose up -d
```

This downloads the images if necessary and starts both containers.

### Step 4: Verify containers

```bash
docker compose ps
```

You should see:

- web-app – Running
- nagios – Running

### Step 5: Access the applications

Open your browser.

**Web application:**

http://localhost:8081

You should see the Nginx welcome page.

**Nagios:**

http://localhost:8080/nagios

Login:

```text
Username: nagiosadmin
Password: nagios
```

These are the default credentials for the selected image. [Docker Hub](https://hub.docker.com/r/jasonrivers/nagios/?utm_source=chatgpt.com)

### Step 6: Verify container logs

```bash
docker compose logs
```

For individual services:

```bash
docker compose logs web
docker compose logs nagios
```

### Step 7: Configure Nagios to monitor the web application

This is useful if your examiner expects actual monitoring rather than simply running both containers.

Enter the Nagios container:

```bash
docker exec -it nagios bash
```

Open the configuration:

```bash
vi /opt/nagios/etc/objects/localhost.cfg
```

Add these definitions:

```text
define host {
    use             linux-server
    host_name       nginx-app
    alias           Nginx Web Application
    address         web
}

define service {
    use                     generic-service
    host_name               nginx-app
    service_description     HTTP Service
    check_command           check_http
}
```

Save the file and exit.

Restart Nagios:

```bash
exit
docker restart nagios
```

Now Nagios can monitor the HTTP service of the web container using its Docker service name, `web`.

---

## Part B: Scenario-Based Questions and Answers

### 1. Temporarily stop all running services without removing containers.

**Answer:**

```bash
docker compose stop
```

Explanation: Stops containers but preserves them, their networks and configuration.

### 2. Start the same containers again without recreating them.

**Answer:**

```bash
docker compose start
```

Explanation: Restarts previously stopped containers.

### 3. Stop and remove containers and the default network.

**Answer:**

```bash
docker compose down
```

Explanation: Stops and removes containers and the Compose-created network.

### 4. A new service has been added to docker-compose.yml. Which command applies the updated configuration?

**Answer:**

```bash
docker compose up -d
```

If an image needs rebuilding:

```bash
docker compose up -d --build
```

Explanation: Creates or updates services according to the Compose configuration.

### 5. Web application runs, but Nagios reports that the application is unavailable. What would you check?

**Answer:**

I would check:

1. **Nagios Host Configuration:** Verify that the hostname and IP address are correct.
2. **Service Configuration:** Verify that the correct monitoring command, such as `check_http`, is configured.
3. **Docker Networking:** Ensure both containers are connected to the same Docker network.
4. **Container Address:** Use the Docker service name instead of localhost.
5. **Port Mapping:** Verify that the application is listening on the correct internal container port.
6. **Container Status:** Execute `docker ps` to verify that both containers are running.
7. **Logs:** Execute `docker compose logs` to identify errors.

---

# QUESTION 2: Jenkins Pipeline – Build Automation and Deployment (25 Marks)

This is probably the most important practical because it combines GitHub, Maven, Jenkins, Tomcat and Docker Hub.

## Part A: Practical Execution

### Step 1: Fork the GitHub repository

Open:

[OLTRS GitHub Repository](https://github.com/archanareddyse/OLTRS.git?utm_source=chatgpt.com)

1. Login to GitHub.
2. Click **Fork**.
3. Select your GitHub account.
4. Create the fork.

Your repository should look something like:

```text
https://github.com/YOUR_USERNAME/OLTRS
```

### Step 2: Configure Jenkins

Open Jenkins:

```text
http://localhost:8080
```

Install these plugins through:

Manage Jenkins → Plugins → Available plugins.

Required plugins:

| Plugin | Purpose |
|---|---|
| Git | Clone GitHub repository |
| Maven Integration | Maven integration |
| Pipeline | Pipeline execution |
| Pipeline Stage View | Display pipeline stages |
| Docker Pipeline | Docker integration |
| Email Extension | Email notifications |

Restart Jenkins if required.

### Step 3: Configure Maven and JDK

Go to:

Manage Jenkins → Tools.

Configure:

**JDK**
- Name: `JDK`
- JAVA_HOME: Your installed JDK path.

**Maven**
- Name: `Maven`
- Install automatically or provide its installation path.

Save.

Make sure Java and Maven are also accessible to the Jenkins agent running the pipeline.

### Step 4: Configure Docker Hub credentials

Go to:

Manage Jenkins → Credentials → System → Global credentials → Add Credentials.

Select:

```text
Kind: Username with password

Username: Your Docker Hub username
Password: Docker Hub access token

ID: dockerhub-creds
```

Use a Docker Hub access token instead of your account password.

### Step 5: Create a Dockerfile

Inside your cloned OLTRS project, create:

`Dockerfile`

Use:

```dockerfile
FROM tomcat:9.0-jdk8

COPY target/*.war /usr/local/tomcat/webapps/ROOT.war

EXPOSE 8080

CMD ["catalina.sh", "run"]
```

This Dockerfile:
- Uses Tomcat as the base image.
- Copies the generated WAR file.
- Deploys the application to Tomcat.
- Exposes port 8080.

**Important:** This assumes your Maven project generates a WAR file compatible with Tomcat 9 and Java 8. If its Java version or packaging differs, adjust the base image accordingly.

### Step 6: Create a Jenkins Pipeline

In Jenkins:

1. Click New Item.
2. Enter `OLTRS-Pipeline`.
3. Select Pipeline.
4. Click OK.

Under Pipeline, select:

```text
Definition: Pipeline script
```

Paste this Scripted Pipeline.

Replace `YOUR_USERNAME` and the email address with your details.

```groovy
node {

    try {

        stage('Checkout') {

            git branch: 'main',
                url: 'https://github.com/YOUR_USERNAME/OLTRS.git'

        }

        stage('Build') {

            bat 'mvn clean test package'

        }

        stage('Docker Build and Tomcat Deployment') {

            bat 'docker build -t YOUR_USERNAME/oltrs:latest .'

            bat 'powershell -Command "docker rm -f oltrs-app; exit 0"'

            bat 'docker run -d --name oltrs-app -p 8081:8080 YOUR_USERNAME/oltrs:latest'

        }

        stage('Docker Push') {

            withCredentials([usernamePassword(
                credentialsId: 'dockerhub-creds',
                usernameVariable: 'DOCKER_USER',
                passwordVariable: 'DOCKER_PASS'
            )]) {

                bat 'echo %DOCKER_PASS%| docker login -u %DOCKER_USER% --password-stdin'

                bat 'docker push YOUR_USERNAME/oltrs:latest'

                bat 'docker logout'

            }

        }

    }

    catch (err) {

        currentBuild.result = 'FAILURE'

        throw err

    }

    finally {

        if (currentBuild.currentResult == 'FAILURE') {

            mail(
                to: 'YOUR_EMAIL@gmail.com',
                subject: 'OLTRS Pipeline Failed',
                body: 'Jenkins pipeline failed. Check Jenkins console output.'
            )

        }

    }

}
```

**Note:** This version uses Windows `bat` commands because you're working on Windows. Your Jenkins agent must have Maven and Docker installed and accessible. The email step requires a configured SMTP server and Mailer support.

### Step 7: Configure GitHub Webhook

Open your forked GitHub repository.

Go to:

Settings → Webhooks → Add webhook.

Enter:

```text
Payload URL:
http://YOUR_PUBLIC_IP:8080/github-webhook/

Content type:
application/json
```

For local testing, use a publicly reachable Jenkins URL or a tunnelling service. `localhost` will not work as a webhook destination for GitHub.

Select:

```text
Just the push event
```

Click Add webhook.

### Step 8: Enable webhook triggering in Jenkins

Open:

Jenkins → OLTRS-Pipeline → Configure.

Under Build Triggers, select:

```text
GitHub hook trigger for GITScm polling
```

Save.

### Step 9: Execute the Pipeline

Click:

```text
Build Now
```

Jenkins should execute these stages:

| Stage | Operation |
|---|---|
| Checkout | Clone GitHub repository |
| Build | Maven build and testing |
| Docker Build and Tomcat Deployment | Create Docker image and deploy application |
| Docker Push | Push image to Docker Hub |

### Step 10: Verify the deployment

Open:

```text
http://localhost:8081
```

Your application should be accessible.

Verify Docker:

```bash
docker images
```

Verify running container:

```bash
docker ps
```

Verify Docker Hub:

Open your Docker Hub account and check the repositories.

You should find:

```text
YOUR_USERNAME/oltrs
```

### Step 11: Configure Pipeline Stage View

Install the Pipeline Stage View plugin if you haven't already.

Open your Jenkins pipeline.

You should see a stage-wise view:

```text
Checkout          SUCCESS
Build             SUCCESS
Docker Build      SUCCESS
Docker Push       SUCCESS
```

---

## Part B: Scenario-Based Jenkins Questions

### 1. Developer pushes a new commit, but Jenkins does not start the pipeline. What configurations should be checked?

**Answer:**

I would check:

1. GitHub Webhook URL is correct.
2. Jenkins server is accessible from GitHub.
3. Webhook content type is `application/json`.
4. Push event is enabled.
5. GitHub webhook trigger is enabled in Jenkins.
6. Correct GitHub repository URL is configured.
7. Jenkins Git plugin is installed.
8. Check GitHub webhook delivery logs for errors.

### 2. Maven build fails. How do you ensure Docker Build and Push stages are skipped?

**Answer:**

Use a `try-catch` mechanism in Scripted Pipeline.

```groovy
try {

    stage('Build') {
        bat 'mvn clean package'
    }

    stage('Docker Build') {
        bat 'docker build -t myapp .'
    }

    stage('Docker Push') {
        bat 'docker push myapp'
    }

}
catch (err) {

    currentBuild.result = 'FAILURE'

    throw err

}
```

Explanation:

If the Maven build fails, Jenkins throws an exception. Execution moves to the catch block, and subsequent stages are skipped.

### 3. Docker Hub push fails due to authentication. How should credentials be configured securely?

**Answer:**

1. Generate a Docker Hub access token.
2. Open Jenkins Credentials.
3. Add Username and Password credentials.
4. Save the credentials with an ID.
5. Use `withCredentials()` in Jenkinsfile.

Example:

```groovy
withCredentials([usernamePassword(
    credentialsId: 'dockerhub-creds',
    usernameVariable: 'USER',
    passwordVariable: 'PASS'
)]) {

    bat 'echo %PASS%| docker login -u %USER% --password-stdin'

    bat 'docker push username/myapp:latest'

}
```

Never hardcode Docker Hub passwords in Jenkinsfiles.

### 4. Pipeline completes Checkout and Build but fails during Deployment. How do you identify the error?

**Answer:**

1. Open Jenkins Dashboard.
2. Select the pipeline job.
3. Open the failed build number.
4. Click Console Output.
5. Identify the stage where execution failed.
6. Check Docker logs using:

```bash
docker logs oltrs-app
```

7. Verify Tomcat port mapping and Docker image configuration.

### 5. Send an email only when the pipeline fails. How?

**Answer:**

Use the `finally` block and check the build result.

```groovy
finally {

    if (currentBuild.currentResult == 'FAILURE') {

        mail(
            to: 'student@gmail.com',
            subject: 'Build Failed',
            body: 'Jenkins pipeline execution failed.'
        )

    }

}
```

Explanation:

- If the build fails, an email is sent.
- If the build succeeds, no email is sent.

---

# QUESTION 3: Containerization and Deployment using Minikube (25 Marks)

This question is relatively straightforward. You need to deploy your Docker Hub image into Kubernetes.

## Part A: Practical Execution

### Step 1: Start Minikube

Open PowerShell.

```bash
minikube start --driver=docker
```

Wait until the cluster starts.

### Step 2: Verify Minikube

```bash
minikube status
```

Expected:

```text
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

### Step 3: Verify Kubernetes nodes

```bash
kubectl get nodes
```

Expected:

```text
NAME       STATUS
minikube   Ready
```

### Step 4: Create a Deployment YAML file

Create:

`deployment.yaml`

Paste:

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:
  name: oltrs-deployment

spec:
  replicas: 2

  selector:
    matchLabels:
      app: oltrs

  template:
    metadata:
      labels:
        app: oltrs

    spec:
      containers:
        - name: oltrs-container

          image: YOUR_USERNAME/oltrs:latest

          ports:
            - containerPort: 8080
```

Replace `YOUR_USERNAME` with your Docker Hub username.

### Step 5: Apply the Deployment

```bash
kubectl apply -f deployment.yaml
```

### Step 6: Verify Deployment

```bash
kubectl get deployments
```

### Step 7: Check Pods

```bash
kubectl get pods
```

You should see two Pods in Running state.

### Step 8: Create Kubernetes Service

Create:

`service.yaml`

Paste:

```yaml
apiVersion: v1

kind: Service

metadata:
  name: oltrs-service

spec:
  type: NodePort

  selector:
    app: oltrs

  ports:
    - port: 8080

      targetPort: 8080

      nodePort: 30080
```

Important:

- `port`: Service port.
- `targetPort`: Application's container port.
- `nodePort`: External port.

NodePort must be between 30000 and 32767 by default.

### Step 9: Apply Service

```bash
kubectl apply -f service.yaml
```

### Step 10: Verify Service

```bash
kubectl get services
```

You should see something similar to:

```text
NAME            TYPE       PORT(S)
oltrs-service   NodePort   8080:30080/TCP
```

### Step 11: Scale Deployment to Four Replicas

Execute:

```bash
kubectl scale deployment oltrs-deployment --replicas=4
```

This changes the number of Pods from two to four.

### Step 12: Verify Four Replicas

```bash
kubectl get deployments
```

Expected:

```text
NAME              READY   UP-TO-DATE   AVAILABLE
oltrs-deployment  4/4     4            4
```

Check Pods:

```bash
kubectl get pods -o wide
```

All four should be Running and Ready.

### Step 13: Verify NodePort Service

```bash
kubectl describe service oltrs-service
```

Check:

- Type: NodePort
- Port: 8080
- TargetPort: 8080
- NodePort: 30080
- Endpoints: Pod IP addresses.

### Step 14: Access the Application

Execute:

```bash
minikube service oltrs-service --url
```

It will return a URL.

Example:

```text
http://127.0.0.1:xxxxx
```

Open that URL in your browser.

If Minikube is running with Docker Desktop on Windows, keep the terminal running if the command creates a tunnel.

This is the documented Minikube method for accessing NodePort applications. [minikube](https://minikube.sigs.k8s.io/docs/handbook/accessing/?utm_source=chatgpt.com)

---

## Part B: Scenario-Based Kubernetes Questions

### 1. Application needs increased traffic. Deployment has two replicas and administrator wants four. Which command?

**Answer:**

```bash
kubectl scale deployment oltrs-deployment --replicas=4
```

Verify:

```bash
kubectl get deployments
kubectl get pods
```

Explanation: Scaling increases the number of Pod replicas managed by the Deployment.

### 2. Pods are running, but users cannot access the application outside the cluster. What should be checked?

**Answer:**

Check or create a Kubernetes Service.

For external access, use NodePort:

```yaml
spec:
  type: NodePort
```

Verify the following:

- Service type.
- NodePort configuration.
- TargetPort.
- Pod labels and Service selectors.
- Minikube networking.

### 3. Deployment is configured for four replicas, but only two Pods are Running. Which commands identify the reason?

**Answer:**

Execute:

```bash
kubectl get pods
```

```bash
kubectl describe deployment oltrs-deployment
```

```bash
kubectl describe pods
```

```bash
kubectl get events
```

```bash
kubectl logs POD_NAME
```

These commands help identify image pull failures, scheduling errors, application crashes and resource problems.

### 4. Service ENDPOINTS shows `<none>`. What could cause this?

**Answer:**

Possible reasons:

1. Service selector does not match Pod labels.
2. Pods are not running.
3. Incorrect labels are configured.
4. Pods are not Ready.

Check:

```bash
kubectl get pods --show-labels
```

```bash
kubectl describe service oltrs-service
```

Compare the Service selector with the Pod labels.

Correct them so that both match.

For example:

Service:

```yaml
selector:
  app: oltrs
```

Pod:

```yaml
labels:
  app: oltrs
```

### 5. Application is no longer required. How do you delete Deployment and Service?

**Answer:**

```bash
kubectl delete deployment oltrs-deployment
```

```bash
kubectl delete service oltrs-service
```

Verify:

```bash
kubectl get deployments
kubectl get services
kubectl get pods
```

---

# QUESTION 4: Deployment in AWS Environment using Amazon EC2 (25 Marks)

This is the final question. You need to launch a virtual machine, install the required software and deploy your web application.

## Part A: Practical Execution

### Step 1: Open AWS Console

Open:

[AWS Management Console](https://console.aws.amazon.com/?utm_source=chatgpt.com)

Login to your AWS account.

Search for:

```text
EC2
```

Open EC2 Dashboard.

### Step 2: Launch an EC2 Instance

Click:

```text
Launch Instance
```

Configure:

| Setting | Value |
|---|---|
| Name | OLTRS-Server |
| AMI | Ubuntu Server |
| Instance Type | t2.micro or eligible free-tier option |
| Key Pair | Create new key pair |
| Network | Default VPC |
| Auto-assign Public IP | Enable |
| Storage | Default |

Click Launch Instance.

Wait until the instance state becomes:

```text
Running
```

And status checks pass.

### Step 3: Configure Security Group

Open your instance → Security → Security Groups.

Edit inbound rules.

Add:

| Type | Port | Source |
|---|---|---|
| SSH | 22 | My IP |
| Custom TCP | 8080 | 0.0.0.0/0 |

Port 22 is for SSH.

Port 8080 is for Tomcat.

If you map the application to port 80 instead, allow HTTP port 80.

**Important:** Never expose SSH to everyone unless specifically required for a temporary lab demonstration.

### Step 4: Connect to EC2

Select your instance.

Click:

```text
Connect → EC2 Instance Connect
```

Alternatively, use SSH from your terminal:

```bash
ssh -i "your-key.pem" ubuntu@YOUR_PUBLIC_IP
```

### Step 5: Update Ubuntu

Execute:

```bash
sudo apt update
```

### Step 6: Install Git, Java, Maven and Docker

```bash
sudo apt install git maven openjdk-17-jdk docker.io -y
```

Start Docker:

```bash
sudo systemctl start docker
```

Enable Docker:

```bash
sudo systemctl enable docker
```

Verify:

```bash
java -version
mvn -version
docker --version
```

**Note:** If your OLTRS project requires Java 8, install the compatible Java version instead of Java 17.

### Step 7: Clone the GitHub Repository

```bash
git clone https://github.com/YOUR_USERNAME/OLTRS.git
```

Move into the project:

```bash
cd OLTRS
```

### Step 8: Build the Application

```bash
mvn clean package
```

This generates the WAR file inside:

```text
target/
```

Verify:

```bash
ls target/
```

### Step 9: Create Dockerfile

Inside the project directory:

```bash
nano Dockerfile
```

Paste:

```dockerfile
FROM tomcat:9.0-jdk8

COPY target/*.war /usr/local/tomcat/webapps/ROOT.war

EXPOSE 8080

CMD ["catalina.sh", "run"]
```

Save the file.

### Step 10: Build Docker Image

```bash
sudo docker build -t oltrs-app .
```

Verify:

```bash
sudo docker images
```

### Step 11: Run the Application

```bash
sudo docker run -d \
--name oltrs-container \
-p 8080:8080 \
oltrs-app
```

### Step 12: Verify the Container

```bash
sudo docker ps
```

Check logs:

```bash
sudo docker logs oltrs-container
```

Wait for Tomcat to start successfully.

### Step 13: Access the Application

Get the EC2 Public IPv4 address from AWS Console.

Open your browser:

```text
http://YOUR_PUBLIC_IP:8080
```

Your OLTRS application should appear.

**Important:** If your application does not open, check:
- EC2 Security Group inbound rules.
- Docker container status.
- Tomcat logs.
- Whether the application is listening on port 8080.

---

## Part B: Scenario-Based Amazon EC2 Questions

### 1. Developer cannot connect to EC2 using SSH. What should be checked?

**Answer:**

Check:

1. EC2 instance state should be Running.
2. Public IPv4 address should be available.
3. Security Group should allow inbound SSH on port 22.
4. SSH source IP should be correct.
5. Correct key pair should be used.
6. Key file permissions should be correct.

For Linux:

```bash
chmod 400 your-key.pem
```

Connect:

```bash
ssh -i your-key.pem ubuntu@PUBLIC_IP
```

### 2. Application runs on port 8080, but users cannot access it through the public IP. What should be verified?

**Answer:**

Check:

1. Security Group allows inbound TCP port 8080.
2. Application listens on port 8080.
3. Docker port mapping is correct.
4. Application is running.
5. Ubuntu firewall is not blocking traffic.
6. Public IP address is correct.

Example Docker mapping:

```bash
-p 8080:8080
```

### 3. EC2 has no Public IPv4 address, but application must be accessed through the internet. What is required?

**Answer:**

Configure public internet connectivity using:

1. A public subnet.
2. An Internet Gateway attached to the VPC.
3. A route table containing a route to the Internet Gateway.
4. A Public IPv4 address or Elastic IP associated with the instance.
5. Appropriate Security Group rules.

An Elastic IP provides a static public IPv4 address.

### 4. Application data must remain available after stopping and starting EC2. Which storage option should be considered?

**Answer:**

Use Amazon EBS (Elastic Block Store).

Explanation:

- EBS provides persistent block storage.
- Data can remain available after stopping and starting an EC2 instance.
- Attach an EBS volume to the instance.
- Store application data on that volume.

For containerized applications, persistent Docker volumes can also be configured on EBS-backed storage.

### 5. Organization wants HTTP access but SSH restricted to administrators. How should Security Group inbound rules be configured?

**Answer:**

Configure:

| Type | Port | Source |
|---|---|---|
| HTTP | 80 | 0.0.0.0/0 |
| SSH | 22 | Administrator's public IP |

Explanation:

- HTTP allows users to access the application.
- SSH allows only authorized administrators to connect.
- Restricting SSH to a specific IP improves security.

---


aws, google, github: N!khil18
nikhilsai.reddy24@kmit.edu.in : N!khil18
docker: N!khil1812




# SOFTWARE ENGINEERING LAB INTERNAL II
# ALL UNIQUE PRACTICAL QUESTIONS AND ANSWERS

---

# 1. DOCKER AND DOCKER COMPOSE

## Q1. Deploy a web application and MySQL database using Docker Compose.

### Step 1: Create project folder

```bash
mkdir docker-project
cd docker-project
```

### Step 2: Create docker-compose.yml

```yaml
services:

  web:
    image: YOUR_USERNAME/YOUR_WEB_IMAGE:latest
    container_name: web-app
    ports:
      - "8080:8080"
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app-network

  db:
    image: mysql:8
    container_name: mysql-db
    environment:
      MYSQL_ROOT_PASSWORD: root123
      MYSQL_DATABASE: mydb
      MYSQL_USER: admin
      MYSQL_PASSWORD: admin123
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-proot123"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  mysql-data:
```

Replace the web image and port with those required by the application.

### Step 3: Start containers

```bash
docker compose up -d
```

### Step 4: Verify containers

```bash
docker ps
```

### Step 5: Check logs

```bash
docker compose logs
```

### Step 6: Access application

```text
http://localhost:8080
```

### Step 7: Verify MySQL

```bash
docker exec -it mysql-db mysql -u admin -p
```

Enter:

```text
admin123
```

Execute:

```sql
SHOW DATABASES;
```

---

## Q2. Deploy WordPress and MySQL using Docker Compose.

Create `docker-compose.yml`.

```yaml
services:

  wordpress:
    image: wordpress:latest
    container_name: wordpress
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: mysql
      WORDPRESS_DB_USER: admin
      WORDPRESS_DB_PASSWORD: admin123
      WORDPRESS_DB_NAME: wordpress_db
    depends_on:
      mysql:
        condition: service_healthy
    networks:
      - wordpress-network

  mysql:
    image: mysql:8
    container_name: mysql
    environment:
      MYSQL_ROOT_PASSWORD: root123
      MYSQL_DATABASE: wordpress_db
      MYSQL_USER: admin
      MYSQL_PASSWORD: admin123
    volumes:
      - mysql-data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-proot123"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - wordpress-network

networks:
  wordpress-network:
    driver: bridge

volumes:
  mysql-data:
```

### Execute

```bash
docker compose up -d
```

Verify:

```bash
docker ps
```

Access:

```text
http://localhost:8080
```

Complete WordPress installation through the browser.

### Verify persistence

```bash
docker compose down
docker compose up -d
```

Your WordPress data remains because MySQL uses a named volume.

---

## Q3. Deploy Flask API and Nginx using Docker Compose.

Create `docker-compose.yml`.

```yaml
services:

  flask:
    image: YOUR_FLASK_IMAGE
    container_name: flask-app
    ports:
      - "5000:5000"
    networks:
      - food-network

  nginx:
    image: nginx:latest
    container_name: nginx-server
    ports:
      - "80:80"
    depends_on:
      - flask
    networks:
      - food-network

networks:
  food-network:
    driver: bridge
```

Execute:

```bash
docker compose up -d
```

Verify:

```bash
docker ps
```

Access:

```text
http://localhost:5000
http://localhost
```

---

## Q4. Deploy Java Maven Web Application and MySQL using Docker.

### Step 1: Build WAR

```bash
mvn clean package
```

### Step 2: Create Dockerfile

```dockerfile
FROM tomcat:9.0-jdk8

COPY target/*.war /usr/local/tomcat/webapps/ROOT.war

EXPOSE 8080

CMD ["catalina.sh", "run"]
```

### Step 3: Build image

```bash
docker build -t java-web-app .
```

### Step 4: Run MySQL

```bash
docker run -d \
--name mysql-db \
-e MYSQL_ROOT_PASSWORD=root123 \
-e MYSQL_DATABASE=mydb \
-p 3306:3306 \
mysql:8
```

### Step 5: Create network

```bash
docker network create app-network
```

Connect MySQL:

```bash
docker network connect app-network mysql-db
```

### Step 6: Run application

```bash
docker run -d \
--name web-app \
--network app-network \
-p 8080:8080 \
java-web-app
```

### Step 7: Verify

```bash
docker ps
```

Access:

```text
http://localhost:8080
```

---

## Q5. Docker Compose commands – Scenario-based questions

**1. Temporarily stop all services without removing containers.**

```bash
docker compose stop
```

**2. Start previously stopped containers without recreating them.**

```bash
docker compose start
```

**3. Stop and remove containers and default network.**

```bash
docker compose down
```

**4. Apply updated docker-compose.yml configuration.**

```bash
docker compose up -d
```

**5. Stop only the backend service.**

```bash
docker compose stop backend
```

**6. Restart only the backend service.**

```bash
docker compose restart backend
```

**7. Rebuild images and start services.**

```bash
docker compose up -d --build
```

**8. View all service logs.**

```bash
docker compose logs
```

**9. View logs of a particular service.**

```bash
docker compose logs backend
```

**10. Remove containers, network and volumes.**

```bash
docker compose down -v
```

---

# 2. JENKINS – FREESTYLE PROJECT

## Q1. Create a Jenkins Freestyle Project to build a Java Maven application.

### Step 1: Open Jenkins

```text
http://localhost:8080
```

### Step 2: Configure JDK

Go to:

```text
Manage Jenkins → Tools → JDK installations
```

Enter:

```text
Name: JDK
JAVA_HOME: Your installed JDK path
```

### Step 3: Configure Maven

Go to:

```text
Manage Jenkins → Tools → Maven installations
```

Enter:

```text
Name: Maven
Install automatically: Enable
```

Save.

### Step 4: Create Freestyle Project

1. Click New Item.
2. Enter project name: `Library-Build`.
3. Select Freestyle Project.
4. Click OK.

### Step 5: Configure GitHub Repository

Under Source Code Management:

Select Git.

Enter:

```text
Repository URL:
https://github.com/YOUR_USERNAME/PROJECT.git

Branch:
*/main
```

### Step 6: Configure Build Trigger

Select:

```text
GitHub hook trigger for GITScm polling
```

Alternatively, select:

```text
Poll SCM
```

### Step 7: Configure Build Environment

Select:

```text
Delete workspace before build starts
```

Optional.

### Step 8: Configure Build

Under Build Steps:

Select:

```text
Invoke top-level Maven targets
```

Enter:

```text
Goals:
clean install test package
```

### Step 9: Archive WAR File

Under Post-build Actions:

Select:

```text
Archive the artifacts
```

Enter:

```text
target/*.war
```

### Step 10: Execute

Click:

```text
Build Now
```

Check:

```text
Console Output
```

Verify:

```text
Finished: SUCCESS
```

WAR file will be available under:

```text
target/
```

---

## Q2. Jenkins Freestyle – Automatically detect GitHub changes.

### Step 1

Open Jenkins project → Configure.

### Step 2

Under Source Code Management:

Select Git and enter the repository URL.

### Step 3

Under Build Triggers:

Select:

```text
GitHub hook trigger for GITScm polling
```

### Step 4

Open GitHub repository.

Go to:

```text
Settings → Webhooks → Add webhook
```

Enter:

```text
Payload URL:
http://YOUR_PUBLIC_IP:8080/github-webhook/

Content type:
application/json
```

Select:

```text
Just the push event
```

Click Add webhook.

### Step 5

Push a new commit to GitHub.

Jenkins automatically starts the build when the webhook is successfully delivered.

---

## Q3. Jenkins – Build only a specific GitHub branch.

Under Source Code Management → Git → Branch Specifier:

```text
*/main
```

To build another branch:

```text
*/development
```

---

## Q4. Jenkins – Scheduled build every day at 6 PM.

Open:

```text
Jenkins → Project → Configure → Build Triggers
```

Select:

```text
Build periodically
```

Enter:

```text
0 18 * * *
```

Explanation:

| Field | Meaning |
|---|---|
| 0 | Minute |
| 18 | Hour |
| * | Every day |
| * | Every month |
| * | Every day of week |

The job executes every day at 6 PM according to the Jenkins controller's timezone.

---

# 3. JENKINS – SCRIPTED PIPELINE

## Q1. Create a Jenkins Scripted Pipeline for Maven Build and Testing.

Create a Pipeline project.

Select:

```text
Definition: Pipeline script
```

Paste:

```groovy
node {

    stage('Checkout') {

        git branch: 'main',
            url: 'https://github.com/YOUR_USERNAME/PROJECT.git'

    }

    stage('Clean') {

        bat 'mvn clean'

    }

    stage('Build') {

        bat 'mvn compile'

    }

    stage('Test') {

        bat 'mvn test'

    }

    stage('Package') {

        bat 'mvn package'

    }

}
```

Click Build Now.

Verify all stages.

---

## Q2. Jenkins Scripted Pipeline using configured Maven installation.

Configure Maven in:

```text
Manage Jenkins → Tools
```

Name:

```text
Maven
```

Pipeline:

```groovy
node {

    def mvn

    stage('Configure Maven') {

        mvn = tool name: 'Maven', type: 'hudson.tasks.Maven$MavenInstallation'

    }

    stage('Checkout') {

        git branch: 'main',
            url: 'https://github.com/YOUR_USERNAME/PROJECT.git'

    }

    stage('Build') {

        bat "\"${mvn}\\bin\\mvn.cmd\" clean install"

    }

    stage('Test') {

        bat "\"${mvn}\\bin\\mvn.cmd\" test"

    }

    stage('Package') {

        bat "\"${mvn}\\bin\\mvn.cmd\" package"

    }

}
```

---

## Q3. Jenkins Scripted Pipeline with separate Checkout, Build and Test stages.

```groovy
node {

    stage('Checkout') {

        git 'https://github.com/YOUR_USERNAME/PROJECT.git'

    }

    stage('Build') {

        bat 'mvn clean install'

    }

    stage('Test') {

        bat 'mvn test'

    }

}
```

---

## Q4. Jenkins Pipeline – Generate WAR file.

```groovy
node {

    stage('Checkout') {

        git 'https://github.com/YOUR_USERNAME/PROJECT.git'

    }

    stage('Clean') {

        bat 'mvn clean'

    }

    stage('Install') {

        bat 'mvn install'

    }

    stage('Test') {

        bat 'mvn test'

    }

    stage('Package') {

        bat 'mvn package'

    }

    stage('Archive') {

        archiveArtifacts artifacts: 'target/*.war'

    }

}
```

---

## Q5. Jenkins Scripted Pipeline – Build, Docker image and Docker Hub push.

### Step 1: Configure Docker Hub Credentials

Go to:

```text
Manage Jenkins → Credentials → System → Global credentials
```

Add:

```text
Kind: Username with password

Username: Docker Hub username
Password: Docker Hub access token

ID: dockerhub-creds
```

### Step 2: Create Dockerfile

```dockerfile
FROM tomcat:9.0-jdk8

COPY target/*.war /usr/local/tomcat/webapps/ROOT.war

EXPOSE 8080

CMD ["catalina.sh", "run"]
```

### Step 3: Jenkinsfile

```groovy
node {

    stage('Checkout') {

        git branch: 'main',
            url: 'https://github.com/YOUR_USERNAME/PROJECT.git'

    }

    stage('Build') {

        bat 'mvn clean test package'

    }

    stage('Docker Build') {

        bat 'docker build -t YOUR_DOCKER_USERNAME/myapp:latest .'

    }

    stage('Docker Push') {

        withCredentials([usernamePassword(
            credentialsId: 'dockerhub-creds',
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASS'
        )]) {

            bat 'echo %DOCKER_PASS%| docker login -u %DOCKER_USER% --password-stdin'

            bat 'docker push YOUR_DOCKER_USERNAME/myapp:latest'

            bat 'docker logout'

        }

    }

}
```

---

## Q6. Jenkins – GitHub Webhook Configuration.

1. Open GitHub repository.
2. Click Settings.
3. Click Webhooks.
4. Click Add webhook.

Enter:

```text
Payload URL:
http://YOUR_PUBLIC_IP:8080/github-webhook/

Content type:
application/json
```

Select:

```text
Just the push event
```

Click Add webhook.

In Jenkins:

```text
Project → Configure → Build Triggers
```

Select:

```text
GitHub hook trigger for GITScm polling
```

Save.

---

# 4. JENKINS EMAIL NOTIFICATIONS

## Q1. Configure Jenkins Email Extension Plugin.

### Step 1: Install Plugin

Go to:

```text
Manage Jenkins → Plugins
```

Install:

```text
Email Extension Plugin
Mailer Plugin
```

### Step 2: Generate Gmail App Password

1. Open Google Account.
2. Click Manage your Google Account.
3. Open Security.
4. Click 2-Step Verification.
5. Scroll down.
6. Select App Passwords.
7. Enter application name: Jenkins.
8. Click Create.
9. Copy the generated password.

### Step 3: Add Jenkins Credentials

Go to:

```text
Manage Jenkins → Credentials → Global credentials → Add Credentials
```

Enter:

```text
Kind: Username with password

Username: Your Gmail address
Password: Gmail App Password

ID: gmail-jenkins
```

Click Create.

### Step 4: Configure Extended E-mail Notification

Go to:

```text
Manage Jenkins → System
```

Under Extended E-mail Notification:

| Setting | Value |
|---|---|
| SMTP Server | smtp.gmail.com |
| SMTP Port | 465 |
| Credentials | gmail-jenkins |
| Use SSL | Enabled |
| Use TLS | Disabled |
| Use OAuth 2.0 | Disabled |
| Default user e-mail suffix | @gmail.com |
| Default Content Type | Plain Text |

Under Default Recipients:

```text
YOUR_EMAIL@gmail.com
```

### Step 5: Test Configuration

Select:

```text
Test configuration by sending test e-mail
```

Enter your email and test.

---

## Q2. Send Email Only When Build Fails.

```groovy
node {

    try {

        stage('Checkout') {

            git 'https://github.com/YOUR_USERNAME/PROJECT.git'

        }

        stage('Build') {

            bat 'mvn clean package'

        }

        stage('Test') {

            bat 'mvn test'

        }

    }

    catch (err) {

        emailext(
            to: 'YOUR_EMAIL@gmail.com',
            subject: 'Jenkins Build Failed',
            body: "Build failed. Check console output: ${BUILD_URL}console"
        )

        currentBuild.result = 'FAILURE'

        throw err

    }

}
```

---

## Q3. Send Email for Both SUCCESS and FAILURE.

```groovy
node {

    try {

        stage('Checkout') {

            git 'https://github.com/YOUR_USERNAME/PROJECT.git'

        }

        stage('Build') {

            bat 'mvn clean package'

        }

        stage('Test') {

            bat 'mvn test'

        }

    }

    catch (err) {

        currentBuild.result = 'FAILURE'

        throw err

    }

    finally {

        if (currentBuild.currentResult == 'SUCCESS') {

            emailext(
                to: 'YOUR_EMAIL@gmail.com',
                subject: 'SUCCESS: Build Completed',
                body: "Application built successfully. Build: ${BUILD_URL}"
            )

        }

        else if (currentBuild.currentResult == 'FAILURE') {

            emailext(
                to: 'YOUR_EMAIL@gmail.com',
                subject: 'FAILURE: Build Failed',
                body: "Application build failed. Check: ${BUILD_URL}console"
            )

        }

    }

}
```

---

## Q4. Intentionally introduce a build failure and send an email.

```groovy
node {

    try {

        stage('Checkout') {

            git 'https://github.com/YOUR_USERNAME/PROJECT.git'

        }

        stage('Build') {

            bat 'invalid-maven-command'

        }

    }

    catch (err) {

        currentBuild.result = 'FAILURE'

        emailext(
            to: 'YOUR_EMAIL@gmail.com',
            subject: 'Test Failure Alert',
            body: 'Jenkins detected a build failure successfully.'
        )

        throw err

    }

}
```

---

# 5. KUBERNETES AND MINIKUBE

## Q1. Deploy an application using Minikube.

### Step 1: Start Minikube

```bash
minikube start --driver=docker
```

### Step 2: Verify cluster

```bash
minikube status
```

### Step 3: Check nodes

```bash
kubectl get nodes
```

### Step 4: Create deployment.yaml

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:
  name: myapp

spec:
  replicas: 2

  selector:
    matchLabels:
      app: myapp

  template:
    metadata:
      labels:
        app: myapp

    spec:
      containers:
        - name: myapp

          image: YOUR_USERNAME/myapp:latest

          ports:
            - containerPort: 8080
```

### Step 5: Apply deployment

```bash
kubectl apply -f deployment.yaml
```

### Step 6: Verify deployment

```bash
kubectl get deployments
```

### Step 7: Verify Pods

```bash
kubectl get pods
```

---

## Q2. Scale Deployment to Four or Five Replicas.

For four replicas:

```bash
kubectl scale deployment myapp --replicas=4
```

For five replicas:

```bash
kubectl scale deployment myapp --replicas=5
```

Verify:

```bash
kubectl get deployments
kubectl get pods
```

---

## Q3. Create a Kubernetes NodePort Service.

Create `service.yaml`.

```yaml
apiVersion: v1

kind: Service

metadata:
  name: myapp-service

spec:
  type: NodePort

  selector:
    app: myapp

  ports:
    - port: 8080

      targetPort: 8080

      nodePort: 30080
```

### Apply Service

```bash
kubectl apply -f service.yaml
```

### Verify

```bash
kubectl get services
```

### Access application

```bash
minikube service myapp-service --url
```

Open the returned URL in your browser.

---

## Q4. Automatically recreate a deleted Pod.

First, check Pods:

```bash
kubectl get pods
```

Delete one Pod:

```bash
kubectl delete pod POD_NAME
```

Check again:

```bash
kubectl get pods
```

Kubernetes automatically creates a replacement Pod because the Deployment maintains the desired replica count.

---

## Q5. Perform Kubernetes Rolling Update.

Update the application image:

```bash
kubectl set image deployment/myapp myapp=YOUR_USERNAME/myapp:v2
```

Check rollout:

```bash
kubectl rollout status deployment/myapp
```

Verify:

```bash
kubectl get pods
```

---

## Q6. Roll back to previous Deployment version.

```bash
kubectl rollout undo deployment/myapp
```

Verify:

```bash
kubectl rollout status deployment/myapp
```

Check history:

```bash
kubectl rollout history deployment/myapp
```

---

## Q7. Troubleshoot Pods that are Pending or repeatedly restarting.

### Step 1

```bash
kubectl get pods
```

### Step 2

```bash
kubectl describe pod POD_NAME
```

### Step 3

```bash
kubectl logs POD_NAME
```

For previous crashed container logs:

```bash
kubectl logs POD_NAME --previous
```

### Step 4

```bash
kubectl get events
```

### Common reasons

| Problem | Reason |
|---|---|
| ImagePullBackOff | Incorrect image name or authentication |
| CrashLoopBackOff | Application repeatedly crashes |
| Pending | Insufficient resources or scheduling issue |
| CreateContainerConfigError | Incorrect configuration |
| ErrImagePull | Image cannot be downloaded |

---

## Q8. Delete Deployment and Service.

```bash
kubectl delete deployment myapp
```

```bash
kubectl delete service myapp-service
```

Verify:

```bash
kubectl get deployments
kubectl get services
kubectl get pods
```

---

# 6. NAGIOS MONITORING

## Q1. Deploy Nagios using Docker.

### Step 1: Pull image

```bash
docker pull jasonrivers/nagios:latest
```

### Step 2: Run container

```bash
docker run -d \
--name nagios \
-p 8081:80 \
jasonrivers/nagios:latest
```

### Step 3: Verify container

```bash
docker ps
```

### Step 4: Access Nagios

Open:

```text
http://localhost:8081/nagios
```

Default credentials:

```text
Username: nagiosadmin
Password: nagios
```

### Step 5: View logs

```bash
docker logs nagios
```

---

## Q2. Configure Nagios to monitor a web application.

Enter Nagios container:

```bash
docker exec -it nagios bash
```

Open:

```bash
vi /opt/nagios/etc/objects/localhost.cfg
```

Add:

```text
define host {
    use             linux-server
    host_name       web-server
    alias           Web Application Server
    address         web
}

define service {
    use                     generic-service
    host_name               web-server
    service_description     HTTP Service
    check_command           check_http
}
```

Here `web` is the Docker service/container hostname, provided both containers can resolve each other on a shared network.

Restart:

```bash
exit
docker restart nagios
```

Open Nagios dashboard and check host/service status.

---

# 7. AWS EC2 DEPLOYMENT

## Q1. Launch an Ubuntu EC2 instance.

### Step 1: Open AWS

Open AWS Management Console.

Search:

```text
EC2
```

Click Launch Instance.

### Step 2: Configure Instance

| Setting | Value |
|---|---|
| Name | My-Application |
| AMI | Ubuntu Server |
| Instance type | Free-tier eligible type |
| Key Pair | Create new key |
| Network | Default VPC |
| Public IP | Enable |
| Storage | Default |

Click Launch Instance.

### Step 3: Configure Security Group

Add inbound rules:

| Type | Port | Source |
|---|---|---|
| SSH | 22 | My IP |
| HTTP | 80 | 0.0.0.0/0 |
| Custom TCP | 8080 | 0.0.0.0/0 |

Use only the application port actually required.

### Step 4: Connect to EC2

Select instance → Connect → EC2 Instance Connect.

Alternatively:

```bash
ssh -i "your-key.pem" ubuntu@PUBLIC_IP
```

### Step 5: Update Ubuntu

```bash
sudo apt update
```

---

## Q2. Deploy index.html using Docker and Nginx.

### Step 1: Install Docker

```bash
sudo apt install docker.io -y
```

### Step 2: Start Docker

```bash
sudo systemctl start docker
```

```bash
sudo systemctl enable docker
```

### Step 3: Create project folder

```bash
mkdir website
cd website
```

### Step 4: Create index.html

```bash
nano index.html
```

Paste:

```html
<html>
<head>
<title>My Website</title>
</head>

<body>
<h1>Welcome to My Website</h1>
<p>Successfully deployed using Docker and Nginx.</p>
</body>

</html>
```

### Step 5: Create Dockerfile

```bash
nano Dockerfile
```

Paste:

```dockerfile
FROM nginx:latest

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80
```

### Step 6: Build Docker Image

```bash
sudo docker build -t mywebsite .
```

### Step 7: Run Container

```bash
sudo docker run -d \
--name website \
-p 80:80 \
mywebsite
```

### Step 8: Verify

```bash
sudo docker ps
```

### Step 9: Access

```text
http://YOUR_EC2_PUBLIC_IP
```

---

## Q3. Clone GitHub repository and deploy using Docker.

### Step 1: Install Git

```bash
sudo apt install git -y
```

### Step 2: Clone repository

```bash
git clone https://github.com/YOUR_USERNAME/PROJECT.git
```

### Step 3: Enter directory

```bash
cd PROJECT
```

### Step 4: Create Dockerfile

For static website:

```dockerfile
FROM nginx:latest

COPY . /usr/share/nginx/html/

EXPOSE 80
```

### Step 5: Build image

```bash
sudo docker build -t mywebsite .
```

### Step 6: Run container

```bash
sudo docker run -d \
--name mywebsite \
-p 80:80 \
mywebsite
```

### Step 7: Verify

```bash
sudo docker ps
```

Open:

```text
http://PUBLIC_IP
```

---

## Q4. Deploy Maven Web Application on AWS EC2 using Docker.

### Step 1: Install required software

```bash
sudo apt update
```

```bash
sudo apt install git maven openjdk-17-jdk docker.io -y
```

### Step 2: Clone repository

```bash
git clone https://github.com/YOUR_USERNAME/PROJECT.git
```

```bash
cd PROJECT
```

### Step 3: Build WAR

```bash
mvn clean package
```

### Step 4: Create Dockerfile

```dockerfile
FROM tomcat:9.0-jdk8

COPY target/*.war /usr/local/tomcat/webapps/ROOT.war

EXPOSE 8080

CMD ["catalina.sh", "run"]
```

### Step 5: Build Docker image

```bash
sudo docker build -t java-app .
```

### Step 6: Run container

```bash
sudo docker run -d \
--name java-app \
-p 8080:8080 \
java-app
```

### Step 7: Verify

```bash
sudo docker ps
```

### Step 8: Access

```text
http://PUBLIC_IP:8080
```

For port 9090, use:

```bash
sudo docker run -d \
--name java-app \
-p 9090:8080 \
java-app
```

Access:

```text
http://PUBLIC_IP:9090
```

---

## Q5. Troubleshoot EC2 SSH connection failure.

### Check:

1. EC2 instance is Running.
2. Public IP exists.
3. Security Group allows port 22.
4. Correct `.pem` file is used.
5. Correct username is used.

For Ubuntu:

```bash
ssh -i "your-key.pem" ubuntu@PUBLIC_IP
```

For Linux key permissions:

```bash
chmod 400 your-key.pem
```

---

## Q6. Application is running but cannot be accessed through public IP.

Check:

1. Security Group inbound rules.
2. Correct application port.
3. Docker port mapping.
4. Container status.
5. Application logs.
6. Ubuntu firewall.
7. Public IPv4 address.

Check container:

```bash
sudo docker ps
```

Check logs:

```bash
sudo docker logs CONTAINER_NAME
```

Check port:

```bash
sudo ufw status
```

If UFW is active and blocking the application:

```bash
sudo ufw allow 8080/tcp
```

---

## Q7. Stop Docker container and terminate EC2 instance.

Stop container:

```bash
sudo docker stop CONTAINER_NAME
```

Remove container:

```bash
sudo docker rm CONTAINER_NAME
```

Remove image:

```bash
sudo docker rmi IMAGE_NAME
```

Terminate EC2:

1. Open AWS Console.
2. Go to EC2.
3. Select instance.
4. Click Instance State.
5. Select Terminate Instance.
6. Confirm termination.

---

# 8. ALL UNIQUE SCENARIO-BASED / THEORETICAL QUESTIONS AND ANSWERS

## A. Docker and Docker Compose

### 1. What happens if the MySQL container stops while the application is running?

The application loses database connectivity. Database-dependent operations such as login, registration and data retrieval may fail until MySQL becomes available again.

### 2. How can database information be preserved after deleting a container?

Use Docker named volumes.

Example:

```yaml
volumes:
  mysql-data:
```

Mount it:

```yaml
volumes:
  - mysql-data:/var/lib/mysql
```

### 3. Why should application and database run in separate containers?

- Independent deployment.
- Easy maintenance.
- Better isolation.
- Independent scaling.
- Easier troubleshooting.
- Different software environments.

### 4. How can unauthorized access to database information be prevented?

- Use strong passwords.
- Store credentials securely.
- Restrict database ports.
- Use private Docker networks.
- Apply database user permissions.
- Enable encryption.

### 5. How do you troubleshoot database connection failure?

Check:

1. MySQL container status.
2. Database credentials.
3. Database name.
4. Hostname.
5. Port configuration.
6. Docker network.
7. Container logs.

Commands:

```bash
docker ps
docker logs mysql-db
docker network ls
```

### 6. Why is a custom Docker network required?

It allows containers to communicate using container or service names and provides network isolation.

### 7. How do you deploy five department websites?

Create five application services with separate configurations and host port mappings, using Docker Compose. Use separate databases or database schemas as required.

### 8. How do you prevent port conflicts?

Map different host ports to the required container ports.

Example:

```yaml
ports:
  - "8081:80"
```

```yaml
ports:
  - "8082:80"
```

---

## B. Jenkins and Maven

### 9. What happens if compilation errors occur?

Maven reports compilation errors and returns a non-zero exit status. Jenkins marks the build as FAILURE and skips subsequent stages.

### 10. How does Jenkins detect GitHub changes?

Using:
- GitHub Webhooks.
- Poll SCM.
- Scheduled builds.

### 11. Difference between WAR generation and deployment on Tomcat?

**WAR generation:** Creates a packaged Java web application.

**Deployment:** Places the WAR file in Tomcat so that users can access the application.

### 12. Why is Maven useful?

- Manages dependencies.
- Automates compilation.
- Executes tests.
- Packages applications.
- Provides a standard build lifecycle.

### 13. How do you troubleshoot an incorrect pom.xml?

Execute:

```bash
mvn validate
```

Then:

```bash
mvn clean package
```

Check:
- XML syntax.
- Dependencies.
- Plugin versions.
- Java version.
- Repository configuration.

### 14. How does Jenkins identify compilation errors?

Maven returns a failure status when compilation fails. Jenkins detects the non-zero exit code and marks the build as FAILURE.

### 15. What happens if compilation succeeds but tests fail?

The build fails during the test phase. Jenkins reports failed test cases and skips later stages if normal failure propagation is used.

### 16. Why use Maven instead of manually downloading JAR files?

Maven automatically downloads and manages dependencies, resolves dependency versions and simplifies project builds.

### 17. How can Jenkins build only a specific branch?

Configure the Git branch specifier:

```text
*/main
```

### 18. Difference between Continuous Integration and Continuous Deployment?

**Continuous Integration (CI):** Developers frequently integrate code changes, followed by automated builds and tests.

**Continuous Deployment (CD):** Successfully tested changes are automatically deployed to the target environment.

### 19. How do you troubleshoot Jenkins build failure?

1. Open Jenkins Dashboard.
2. Select the failed job.
3. Open the failed build.
4. Click Console Output.
5. Identify the error.
6. Correct the source code or configuration.
7. Execute the pipeline again.

### 20. How do you ensure Docker stages are skipped when Maven fails?

Use Scripted Pipeline exception handling. If Maven throws an error, execution moves to `catch`, and subsequent Docker stages are not executed.

### 21. How do you securely configure Docker Hub credentials?

Use Jenkins Credentials and `withCredentials()`.

Never hardcode passwords in Jenkinsfiles.

### 22. Why does a GitHub webhook fail to trigger Jenkins?

Possible reasons:

- Incorrect webhook URL.
- Jenkins is not publicly accessible.
- GitHub webhook delivery failed.
- Incorrect repository URL.
- GitHub hook trigger is disabled.
- Required Jenkins plugin is missing.

### 23. How do you identify a failed Jenkins stage?

Open:

```text
Jenkins → Job → Build Number → Console Output
```

Check the Pipeline Stage View to identify the failed stage.

### 24. How do you send an email only when a build fails?

Use:

```groovy
catch (err) {

    emailext(
        to: 'YOUR_EMAIL@gmail.com',
        subject: 'Build Failed',
        body: 'Jenkins build failed.'
    )

    throw err

}
```

### 25. How do you send emails for both successful and failed builds?

Use:

```groovy
finally {

    if (currentBuild.currentResult == 'SUCCESS') {

        emailext(
            to: 'YOUR_EMAIL@gmail.com',
            subject: 'Build Successful',
            body: 'Build completed successfully.'
        )

    }

    else if (currentBuild.currentResult == 'FAILURE') {

        emailext(
            to: 'YOUR_EMAIL@gmail.com',
            subject: 'Build Failed',
            body: 'Build execution failed.'
        )

    }

}
```

---

## C. Nagios Monitoring

### 26. How can Nagios identify whether an application server is running or down?

Nagios uses host checks such as ping and service checks to verify server availability.

### 27. Difference between WARNING and CRITICAL alerts?

**WARNING:** A monitored resource has crossed a warning threshold but is not yet in a critical state.

**CRITICAL:** The resource or service has reached a critical condition or become unavailable.

### 28. What happens if CPU usage reaches 100%?

The server may become slow, application response times increase and requests may fail. Nagios can generate alerts when configured CPU thresholds are exceeded.

### 29. How can Nagios detect application failure even when the server is running?

Configure HTTP service checks. The server may respond to ping while the web application itself is unavailable.

### 30. How can an administrator identify whether downtime is caused by Tomcat, MySQL or the server?

Configure separate monitoring checks:

- Host monitoring for server availability.
- HTTP monitoring for Tomcat/application.
- Database monitoring for MySQL.

Compare their statuses and logs.

### 31. How does Nagios identify high CPU utilization?

It executes CPU monitoring plugins and compares the result with configured warning and critical thresholds.

### 32. What happens if the Nagios monitoring server fails?

Monitoring and alert generation stop temporarily. The monitored application may continue running, but its status is not being monitored.

### 33. How do you identify whether downtime is caused by networking?

Check connectivity, ping, DNS resolution, firewall rules and network service checks.

---

## D. Kubernetes

### 34. What happens if one application Pod crashes?

Kubernetes detects the failed Pod and creates a replacement to maintain the desired replica count.

### 35. How does Kubernetes maintain the desired number of replicas?

A Deployment uses a ReplicaSet to maintain the specified number of Pods. If a Pod fails, a replacement is created.

### 36. Difference between scaling up and scaling down?

**Scaling up:** Increasing the number of application replicas.

**Scaling down:** Decreasing the number of application replicas.

### 37. How does increasing replicas improve performance?

It distributes incoming requests across multiple application instances, reducing the load on individual Pods.

### 38. How does Kubernetes distribute incoming requests?

A Kubernetes Service provides a stable network endpoint and distributes traffic across eligible backend Pods.

### 39. Difference between manual scaling and automatic scaling?

**Manual scaling:** Administrator changes the replica count using commands or YAML.

**Automatic scaling:** Kubernetes Horizontal Pod Autoscaler adjusts replica count based on configured metrics.

### 40. Why is Kubernetes Service required?

It provides a stable IP address and DNS name to access application Pods and distributes requests among them.

### 41. How do you recover from a failed application update?

Execute:

```bash
kubectl rollout undo deployment/myapp
```

### 42. Why are only two out of four Pods running?

Possible reasons:

- Insufficient CPU or memory.
- Incorrect Docker image.
- ImagePullBackOff.
- Application crashes.
- Scheduling failures.
- Configuration errors.

Troubleshoot:

```bash
kubectl get pods
kubectl describe pods
kubectl get events
kubectl logs POD_NAME
```

### 43. Why does Service ENDPOINTS show `<none>`?

Possible reasons:

- Service selector does not match Pod labels.
- Pods are not Ready.
- Pods are not running.

Check:

```bash
kubectl get pods --show-labels
kubectl describe service myapp-service
```

### 44. How do you expose Pods outside the cluster?

Create a Kubernetes Service using NodePort.

```yaml
type: NodePort
```

### 45. What is a Rolling Update?

A Rolling Update gradually replaces old application Pods with new Pods without stopping the entire application at once.

### 46. What is Rollback?

Rollback restores a previous working version of an application Deployment.

Command:

```bash
kubectl rollout undo deployment/myapp
```

---

## E. AWS EC2

### 47. Why can't users access an application through the EC2 Public IP?

Possible reasons:

- Incorrect Security Group rules.
- Application not running.
- Incorrect port mapping.
- Firewall blocking traffic.
- No Public IPv4 address.
- Incorrect application configuration.

### 48. Why should HTTP and application ports be configured in Security Groups?

Security Groups act as virtual firewalls. They control which incoming traffic can reach the EC2 instance.

### 49. What happens if Tomcat stops while EC2 is running?

The EC2 instance remains running, but applications hosted on Tomcat become unavailable.

### 50. How can application availability be improved if an EC2 instance fails?

Use:

- Multiple EC2 instances.
- Elastic Load Balancing.
- Auto Scaling Groups.
- Health checks.
- Backups.

### 51. Advantages of AWS over a local server?

- Scalability.
- Remote accessibility.
- High availability options.
- Flexible computing resources.
- Pay-as-you-go pricing.
- Managed cloud services.

### 52. Why restrict SSH access?

Restricting SSH to administrator IP addresses reduces unauthorized access and improves instance security.

### 53. What if EC2 has no Public IPv4 address?

Configure internet connectivity using:

- Public subnet.
- Internet Gateway.
- Appropriate route table.
- Public IPv4 address or Elastic IP.
- Security Group rules.

### 54. Which storage option preserves data after stopping and starting EC2?

Amazon EBS (Elastic Block Store).

EBS provides persistent block storage independently of the running state of an EC2 instance.

### 55. How should Security Groups be configured for HTTP and restricted SSH?

| Type | Port | Source |
|---|---|---|
| HTTP | 80 | 0.0.0.0/0 |
| SSH | 22 | Administrator's public IP |

---

# 9. IMPORTANT COMMANDS – QUICK REVISION

| Technology | Command |
|---|---|
| Docker | `docker ps` |
| Docker | `docker images` |
| Docker | `docker logs CONTAINER_NAME` |
| Docker | `docker stop CONTAINER_NAME` |
| Docker | `docker rm CONTAINER_NAME` |
| Docker | `docker build -t app .` |
| Docker | `docker run -d -p 8080:8080 app` |
| Docker Compose | `docker compose up -d` |
| Docker Compose | `docker compose stop` |
| Docker Compose | `docker compose start` |
| Docker Compose | `docker compose down` |
| Docker Compose | `docker compose logs` |
| Maven | `mvn clean` |
| Maven | `mvn compile` |
| Maven | `mvn test` |
| Maven | `mvn install` |
| Maven | `mvn package` |
| Kubernetes | `minikube start --driver=docker` |
| Kubernetes | `kubectl get nodes` |
| Kubernetes | `kubectl get pods` |
| Kubernetes | `kubectl get deployments` |
| Kubernetes | `kubectl get services` |
| Kubernetes | `kubectl apply -f deployment.yaml` |
| Kubernetes | `kubectl apply -f service.yaml` |
| Kubernetes | `kubectl scale deployment myapp --replicas=4` |
| Kubernetes | `kubectl describe pod POD_NAME` |
| Kubernetes | `kubectl logs POD_NAME` |
| Kubernetes | `kubectl rollout undo deployment/myapp` |
| Kubernetes | `kubectl delete deployment myapp` |
| Kubernetes | `kubectl delete service myapp-service` |
| Minikube | `minikube service myapp-service --url` |
| AWS | `sudo apt update` |
| AWS | `sudo apt install git maven docker.io -y` |
| AWS | `sudo docker build -t app .` |
| AWS | `sudo docker ps` |
| AWS | `sudo docker logs CONTAINER_NAME` |
