# BoardgameListingWebApp

## Description 

**Board Game Database Full-Stack Web Application.**
This web application displays lists of board games and their reviews. While anyone can view the board game lists and reviews, they are required to log in to add/ edit the board games and their reviews. The 'users' have the authority to add board games to the list and add reviews, and the 'managers' have the authority to edit/ delete the reviews on top of the authorities of users.  

## Technologies

- Java
- Spring Boot
- Amazon Web Services(AWS) EC2
- Thymeleaf
- Thymeleaf Fragments
- HTML5
- CSS
- JavaScript
- Spring MVC
- JDBC
- H2 Database Engine (In-memory)
- JUnit test framework
- Spring Security
- Twitter Bootstrap
- Maven

## Features

- Full-Stack Application
- UI components created with Thymeleaf and styled with Twitter Bootstrap
- Authentication and authorization using Spring Security
  - Authentication by allowing the users to authenticate with a username and password
  - Authorization by granting different permissions based on the roles (non-members, users, and managers)
- Different roles (non-members, users, and managers) with varying levels of permissions
  - Non-members only can see the boardgame lists and reviews
  - Users can add board games and write reviews
  - Managers can edit and delete the reviews
- Deployed the application on AWS EC2
- JUnit test framework for unit testing
- Spring MVC best practices to segregate views, controllers, and database packages
- JDBC for database connectivity and interaction
- CRUD (Create, Read, Update, Delete) operations for managing data in the database
- Schema.sql file to customize the schema and input initial data
- Thymeleaf Fragments to reduce redundancy of repeating HTML elements (head, footer, navigation)



# DevSecOps CI/CD – Docker, Nexus, Kubernetes (Beginner → Intermediate Notes)

> **Purpose**
> Yeh document un sab cheezon ka **structured, detailed notes** hai jo humne abhi tak seekha hai — bilkul **zero se**, real commands, examples aur *why–what–how* ke saath.
> Target audience: **Beginner / Junior DevOps → Solid Foundation**

---

## 1️⃣ Big Picture – Hum kya build kar rahe hain?

Hum ek **modern DevSecOps flow** samajh rahe hain:

```
Code
 ↓
Maven Build (JAR)
 ↓
Nexus (Artifact Repository)
 ↓
Docker Image
 ↓
Docker Registry (Nexus)
 ↓
Kubernetes (Pods / Services / LoadBalancer)
```

Is flow mein har tool ka **specific role** hai — koi bhi tool extra nahi hai.

---

## 2️⃣ Git – Remote ka Concept

### 🔹 Git Remote kya hota hai?

> **Git remote sirf ek naam + URL ka mapping hota hai**
> Remote koi repo nahi hota, koi VM nahi hota.

Example:

```
origin → https://github.com/user/project.git
```

### 🔹 Existing remote ko kaise hataate hain?

```bash
git remote -v
git remote remove origin
git remote -v
```

✔️ Sirf mapping delete hoti hai, local code safe rehta hai.

---

## 3️⃣ Jenkins Pipeline – Variables ka Role

```groovy
APP_NAME = "boardgame"
VERSION = "1.0.${BUILD_NUMBER}"

NEXUS_MAVEN_REPO = "maven-releases"
NEXUS_DOCKER_REPO = "docker-repo"

NEXUS_URL = "http://nexus:8081"
DOCKER_IMAGE = "nexus:8083/${APP_NAME}:${VERSION}"
```

### 🧠 Meaning

* `APP_NAME` → Application ka naam
* `VERSION` → CI-friendly version (har build unique)
* `BUILD_NUMBER` → Jenkins auto increment
* `NEXUS_URL` → Nexus Web UI / API
* `8081` → Nexus UI + Maven
* `8083` → Docker Registry port

Final Docker image example:

```
nexus:8083/boardgame:1.0.12
```

---

## 4️⃣ Jenkins – Nexus mein JAR publish karna

```groovy
stage('Publish JAR to Nexus') {
  steps {
    withCredentials([usernamePassword(
      credentialsId: 'nexus-creds',
      usernameVariable: 'NEXUS_USER',
      passwordVariable: 'NEXUS_PASS'
    )]) {
      sh '''
        mvn deploy \
        -Dnexus.username=$NEXUS_USER \
        -Dnexus.password=$NEXUS_PASS
      '''
    }
  }
}
```

### 🧠 Flow

* Jenkins credentials vault se secrets nikalta hai
* Env variables banata hai
* `mvn deploy` Nexus repo mein JAR upload karta hai

✔️ Secrets hardcode nahi hote

---

## 5️⃣ Docker – Multi-stage Build (Java App)

### 🔹 Build Stage (Heavy tools)

```dockerfile
FROM maven:3.9.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn -DskipTests dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests
```

* Maven + JDK
* JAR build hoti hai
* Dependencies cache hoti hain

---

### 🔹 Runtime Stage (Lightweight)

```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

✔️ Final image mein sirf:

* Java runtime
* app.jar

❌ Maven / source code nahi

---

## 6️⃣ Docker Networking – Container IP vs Public IP

### 🔹 docker inspect se kya milta hai?

```json
"IPAddress": "172.17.0.2"
```

* Yeh **container internal IP** hai
* Internet se accessible nahi

---

### 🔹 Port Mapping

```json
"HostPort": "8081"
```

Meaning:

```
VM_IP:8081 → Container:8081
```

---

### 🔥 Public IP kaha hoti hai?

> **Public IP container ki nahi hoti**
> **Public IP hamesha VM / Cloud resource ki hoti hai**

Check karne ke liye:

```bash
curl ifconfig.me
```

---

## 7️⃣ Kubernetes Service – LoadBalancer (Core Concept)

### 🔹 LoadBalancer Service kya karti hai?

> Cloud provider ka **real Load Balancer** banwa deti hai

Flow:

```
Public IP (Cloud LB)
 ↓
Cloud Load Balancer
 ↓
Worker Node IP : NodePort
 ↓
kube-proxy
 ↓
ClusterIP Service
 ↓
Pod
```

---

### 🔹 Load Balancing kaise hoti hai?

**Level 1 – Cloud LB (Nodes)**

* Round-Robin
* Least Connections
* Hash-based

**Level 2 – Kubernetes Service (Pods)**

* kube-proxy
* iptables / IPVS

---

## 8️⃣ Load Balancing Algorithms (Simple)

| Algorithm         | Logic                     |
| ----------------- | ------------------------- |
| Round-Robin       | Ek ke baad ek             |
| Least Connections | Jo kam busy               |
| Hash-based        | Same client → same server |

---

## 9️⃣ DevSecOps – YAML & Cluster Security

### 🔹 kubesec (YAML Scanner)

```bash
kubesec scan deployment.yaml
```

* Static YAML check
* Privileged containers ❌
* Resource limits ❌

---

### 🔹 kube-bench (CIS Benchmark)

* Cluster hardening
* kubelet config
* file permissions

---

### 🔹 Trivy (Cluster + Config Scan)

```bash
trivy k8s --report summary
```

HTML reports for audit possible.

---

## 🔟 Key Mental Models (Most Important)

* Container ka **public IP nahi hota**
* Service VM nahi hoti, logical hoti hai
* Load balancing **cloud + kube-proxy dono** karte hain
* Security build ke baad nahi, **design se start hoti hai**

---

## 🎤 Interview-ready Summary

> “I understand the full DevSecOps flow from Git and Jenkins CI, Maven and Nexus artifact management, Docker multi-stage builds, Docker networking, and Kubernetes services with cloud load balancing and security scanning.”

---

**End of Notes**

## How to Run

1. Clone the repository
2. Open the project in your IDE of choice
3. Run the application
4. To use initial user data, use the following credentials.
  - username: bugs    |     password: bunny (user role)
  - username: daffy   |     password: duck  (manager role)
5. You can also sign-up as a new user and customize your role to play with the application! 😊
