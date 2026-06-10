# 🏋️‍♂️ AI-Powered Fitness Tracker Microservices

An enterprise-grade, full-stack microservices application built and designed by **Saurabh Singh** using **Spring Boot**, **React**, **RabbitMQ**, **Keycloak OAuth2**, **PostgreSQL**, **MongoDB**, and **Google Gemini Generative AI**.

This system allows users to securely authenticate, track physical activities, and dynamically receive personalized AI-driven suggestions, safety recommendations, and workout analyses in real time.

---

## 🚀 Key Features

* **🛡️ Secure Gateway & Authentication**: Configured with Keycloak (OAuth2 with PKCE flow) at the API Gateway level to secure all backend microservice communications.
* **⚡ Automatic User Provisioning**: Features a custom Gateway WebFilter [KeycloakUserSyncFilter.java](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/gateway/src/main/java/com/fitness/gateway/KeycloakUserSyncFilter.java) that parses JWT claims and automatically registers new users inside the postgres database upon their first request.
* **🔄 Asynchronous Event-Driven Pipeline**: Decouples the activity logger and AI analytics engines. Logging an activity writes to MongoDB and publishes a message to RabbitMQ, triggering the AI engine in the background.
* **🤖 Gemini Pro Integrations**: The AI engine dynamically prompts Google Gemini with structural JSON schemas, parses the responses, and persists targeted training suggestions and safety guidelines.
* **🌐 Responsive React Client**: Developed with Material UI (v6), Redux Toolkit, and Vite to deliver a seamless, state-driven user experience.

---

## 🏛️ System Topology

For a deeper dive into the system flows, network configs, and senior-level observations, refer to the main [resource.md](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/resource.md) documentation.

```
[React App (5173)] ---> [API Gateway (8080)] ---> [Microservices (8081, 8082, 8083)]
                              |
                     [Keycloak Sync (8181)]
                              |
                    [RabbitMQ (5672)] ---> [Gemini AI Engine]
```

---

## 🛠️ Technology Stack

| Layer | Component | Details |
| :--- | :--- | :--- |
| **Frontend** | [fitness-app-frontend](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/fitness-app-frontend) | React 19, Vite, Material UI 6, Redux Toolkit, Axios, PKCE Auth |
| **Routing** | [gateway](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/gateway) | Spring Cloud Gateway, WebFlux Security, Keycloak Sync |
| **Discovery & Config** | [eureka](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/eureka), [configserver](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/configserver) | Netflix Eureka (8761), Spring Cloud Config Server (8888) |
| **Identity & Access** | Keycloak | Port 8181, Realm `fitness-oauth2`, OpenID JWT |
| **Microservices** | Java Core Services | [userservice](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/userservice) (JPA/Postgres), [activityservice](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/activityservice) (MongoDB/RabbitMQ), [aiservice](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/aiservice) (Gemini Pro/RabbitMQ) |
| **Databases** | PostgreSQL & MongoDB | Postgres (Port 5432), MongoDB (Port 27017) |

---

## 🚀 Getting Started

### 1. Launch Infrastructure Services
Spin up PostgreSQL, MongoDB, RabbitMQ, and Keycloak instantly using the root [docker-compose.yml](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/docker-compose.yml):
```bash
docker compose up -d
```

### 2. Configure Keycloak Realm & Client
* Admin console: `http://localhost:8181` (Credentials: `admin` / `admin`).
* Create a Realm named `fitness-oauth2`.
* Create a client named `oauth2-pkce-client` with client authentication off, standard flow on, redirect URI `http://localhost:5173/*`, and web origins `http://localhost:5173`.
* Create a user, go to credentials, set password (disable temporary toggle).

### 3. Run Microservices (Chronological Order)
Ensure Java 23/21 is installed. Open separate terminals to run each service:
```bash
# 1. Start Service Discovery
cd eureka && .\mvnw.cmd spring-boot:run

# 2. Start Central Config
cd ../configserver && .\mvnw.cmd spring-boot:run

# 3. Start Core Services
cd ../userservice && .\mvnw.cmd spring-boot:run
cd ../activityservice && .\mvnw.cmd spring-boot:run

# 4. Set Gemini env variables and Start AI Service
$env:GEMINI_API_URL="https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key="
$env:GEMINI_API_KEY="AIzaSyYourGeminiApiKeyHere"
cd ../aiservice && .\mvnw.cmd spring-boot:run

# 5. Start API Gateway
cd ../gateway && .\mvnw.cmd spring-boot:run
```

### 4. Run Frontend Client
```bash
cd fitness-app-frontend
npm install
npm run dev
```
Open `http://localhost:5173` to test the application.

---

## ✍️ Author & Maintainer

* **Saurabh Singh** - Lead Developer & Architect of this project.
