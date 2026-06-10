# AI-Powered Fitness Tracker Microservices Architecture

This document provides a comprehensive detailing of the **AI-Powered Fitness Tracker Microservices** project. The application utilizes a modern, distributed architecture combining a React frontend, Spring Boot backend microservices, Netflix Eureka service discovery, Spring Cloud Config, RabbitMQ asynchronous messaging, PostgreSQL/MongoDB database engines, Keycloak OAuth2 authentication (PKCE flow), and Google Gemini Generative AI.

---

## 🏛️ System Architecture

The following diagram illustrates the structural topology, network ports, databases, and communication flows of the system:

```mermaid
flowchart TD
    subgraph Frontend ["Client Layer (Port: 5173)"]
        React["React Frontend (MUI 6 & Redux)"]
    end

    subgraph Security ["Identity Provider (Port: 8181)"]
        Keycloak["Keycloak Server (fitness-oauth2 Realm)"]
    end

    subgraph Gateway ["Routing Layer (Port: 8080)"]
        APIGateway["Spring Cloud Gateway"]
        SyncFilter["KeycloakUserSyncFilter (Auto-Syncs Users)"]
    end

    subgraph Infrastructure ["Infrastructure Services"]
        ConfigServer["Spring Cloud Config Server (Port: 8888)"]
        Eureka["Eureka Discovery Server (Port: 8761)"]
        RabbitMQ["RabbitMQ Message Broker (Port: 5672)"]
    end

    subgraph Services ["Core Microservices"]
        UserService["User Service (Port: 8081)"]
        ActivityService["Activity Service (Port: 8082)"]
        AIService["AI Recommendation Service (Port: 8083)"]
    end

    subgraph Persistence ["Persistence Layer"]
        Postgres[("PostgreSQL\n(fitness_user_db)")]
        MongoActivity[("MongoDB\n(fitnessactivity)")]
        MongoRecommendation[("MongoDB\n(fitnessrecommendation)")]
    end

    subgraph External ["External Services"]
        Gemini["Google Gemini Pro API"]
    end

    %% Flow Connections
    React -.->|1. Authenticate (PKCE)| Keycloak
    React -->|2. REST Requests + Bearer Token| APIGateway
    APIGateway -->|3. Intercept & Validate| SyncFilter
    SyncFilter -->|4. Get /api/users/{id}/validate| UserService
    SyncFilter -->|5. Auto-Register (If New)| UserService
    
    APIGateway -->|Route /api/users/**| UserService
    APIGateway -->|Route /api/activities/**| ActivityService
    APIGateway -->|Route /api/recommendations/**| AIService

    %% Service Registrations & Configuration
    UserService -.->|Register| Eureka
    ActivityService -.->|Register| Eureka
    AIService -.->|Register| Eureka
    APIGateway -.->|Register| Eureka

    UserService -.->|Fetch Config| ConfigServer
    ActivityService -.->|Fetch Config| ConfigServer
    AIService -.->|Fetch Config| ConfigServer
    APIGateway -.->|Fetch Config| ConfigServer

    %% Microservice Communications & Storage
    UserService -->|JPA| Postgres
    ActivityService -->|WebClient Validate| UserService
    ActivityService -->|Spring Data Mongo| MongoActivity
    ActivityService -->|Publish Activity Event| RabbitMQ
    
    RabbitMQ -->|Consume Event| AIService
    AIService -->|Request Recommendation| Gemini
    AIService -->|Spring Data Mongo| MongoRecommendation
```

---

## 📂 Microservices Directory & Configuration Details

### 1. Eureka Server (`eureka`)
* **Purpose**: Netflix Eureka Server that provides dynamic service discovery and registration for all backend components.
* **Port**: `8761`
* **Bootstrap Config**: [application.yml](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/eureka/src/main/resources/application.yml)
* **Main Class**: [EurekaApplication.java](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/eureka/src/main/java/com/server/eureka/EurekaApplication.java) marked with `@EnableEurekaServer`.

### 2. Config Server (`configserver`)
* **Purpose**: Centralized external configuration server serving all configuration properties dynamically using the Spring Cloud Config Native file system profile.
* **Port**: `8888`
* **Bootstrap Config**: [application.yml](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/configserver/src/main/resources/application.yml)
* **Config Profiles Location**: Files are stored locally inside the resources directory [classpath:/config](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/configserver/src/main/resources/config).
* **Main Class**: [ConfigserverApplication.java](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/configserver/src/main/java/com/fitness/configserver/ConfigserverApplication.java) marked with `@EnableConfigServer`.

### 3. API Gateway (`gateway`)
* **Purpose**: Gateway routing agent that handles load-balanced routing, cross-origin requests, JWT authentication validation, and automatic profile registration/synchronization.
* **Port**: `8080`
* **Main Config (from Config Server)**: [api-gateway.yml](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/configserver/src/main/resources/config/api-gateway.yml)
* **Main Class**: [GatewayApplication.java](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/gateway/src/main/java/com/fitness/gateway/GatewayApplication.java)
* **Core Mechanisms**:
  * **JWT Validation**: Configured to check tokens against Keycloak using JWK set URI `http://localhost:8181/realms/fitness-oauth2/protocol/openid-connect/certs`.
  * **Auto User Provisioning**: Uses [KeycloakUserSyncFilter.java](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/gateway/src/main/java/com/fitness/gateway/KeycloakUserSyncFilter.java) to automatically check if the user from the incoming token exists in `userservice`. If not, it synchronously triggers user provisioning.
  * **CORS Configurations**: Allows all common verbs from `http://localhost:5173` passing headers `Authorization`, `Content-Type`, and `X-User-ID`.

### 4. User Service (`userservice`)
* **Purpose**: Manages registered user profile details, database-backed roles, and registration operations.
* **Port**: `8081`
* **Database**: PostgreSQL (`fitness_user_db` database on port `5432`)
* **Main Config (from Config Server)**: [user-service.yml](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/configserver/src/main/resources/config/user-service.yml)
* **Core Classes**:
  * Entity: [User.java](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/userservice/src/main/java/com/fitness/userservice/model/User.java)
  * Service: [UserService.java](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/userservice/src/main/java/com/fitness/userservice/service/UserService.java)
  * Controller: [UserController.java](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/userservice/src/main/java/com/fitness/userservice/controller/UserController.java)

### 5. Activity Service (`activityservice`)
* **Purpose**: Manages tracking and retrieval of users' physical activities (e.g. running, cycling, walking) and coordinates events with RabbitMQ.
* **Port**: `8082`
* **Database**: MongoDB (`fitnessactivity` database on port `27017`)
* **Main Config (from Config Server)**: [activity-service.yml](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/configserver/src/main/resources/config/activity-service.yml)
* **Core Classes**:
  * Document Model: [Activity.java](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/activityservice/src/main/java/com/fitness/activityservice/model/Activity.java)
  * Service: [ActivityService.java](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/activityservice/src/main/java/com/fitness/activityservice/service/ActivityService.java)
  * User Validation Client: [UserValidationService.java](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/activityservice/src/main/java/com/fitness/activityservice/service/UserValidationService.java) (calls `userservice` via WebClient)
  * Broker Configuration: [RabbitMqConfig.java](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/activityservice/src/main/java/com/fitness/activityservice/config/RabbitMqConfig.java) (defines exchange `fitness.exchange`, queue `activity.queue`, routing key `activity.tracking`)

### 6. AI Service (`aiservice`)
* **Purpose**: Listens to published activity events asynchronously via RabbitMQ, crafts dynamic prompt configurations, queries Google Gemini Pro API, parses the JSON responses, and provisions workout analysis and safety guidelines.
* **Port**: `8083`
* **Database**: MongoDB (`fitnessrecommendation` database on port `27017`)
* **Main Config (from Config Server)**: [ai-service.yml](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/configserver/src/main/resources/config/ai-service.yml)
* **Core Classes**:
  * Message Listener: [ActivityMessageListener.java](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/aiservice/src/main/java/com/fitness/aiservice/service/ActivityMessageListener.java)
  * prompt Orchestration: [ActivityAIService.java](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/aiservice/src/main/java/com/fitness/aiservice/service/ActivityAIService.java)
  * REST API Client: [GeminiService.java](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/aiservice/src/main/java/com/fitness/aiservice/service/GeminiService.java) (invokes Gemini generative AI using a structured payload)
  * Recommendation Entity: [Recommendation.java](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/aiservice/src/main/java/com/fitness/aiservice/model/Recommendation.java)

### 7. React Frontend (`fitness-app-frontend`)
* **Purpose**: A React Single-Page Application (SPA) utilizing Vite, Material UI (v6), Redux Toolkit, and standard PKCE-based security protocols.
* **Port**: `5173`
* **Core Configs & Pages**:
  * Auth Setup: [authConfig.js](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/fitness-app-frontend/src/authConfig.js)
  * Entry Point: [App.jsx](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/fitness-app-frontend/src/App.jsx)
  * Axios API definitions: [api.js](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/fitness-app-frontend/src/services/api.js) (injects token and `X-User-ID` into every request)
  * Screens:
    * [ActivityForm.jsx](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/fitness-app-frontend/src/components/ActivityForm.jsx): Submit physical activities.
    * [ActivityList.jsx](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/fitness-app-frontend/src/components/ActivityList.jsx): List recorded workouts.
    * [ActivityDetail.jsx](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/fitness-app-frontend/src/components/ActivityDetail.jsx): Detail workout, and display real-time parsed AI analysis, improvements, recommendations, and safety advice.

---

## ⚡ Integration Workflows

### User Lifecycle & Auto-Sync
```
[React SPA] login (Keycloak PKCE)
   |
   +---> [Axios Interceptor] (Attaches Token + X-User-ID)
            |
            v
   [Spring Cloud Gateway]
      |
      +---> [KeycloakUserSyncFilter]
               |
               +---> Call User Service: /api/users/{userId}/validate
                        |
                        +---> (If Not Exists) Call User Service: /api/users/register (auto-registers)
```

### Activity Logging & AI Inference Pipeline
```
[React SPA] POST /api/activities
   |
   v
[Gateway]
   |
   v
[Activity Service]
   |
   +---> 1. Save Activity in MongoDB
   +---> 2. Publish Activity Event to RabbitMQ (fitness.exchange -> activity.tracking)
            |
            v
[RabbitMQ Broker]
            |
            v
[AI Service] (Listens on activity.queue)
   |
   +---> 1. Build prompt requesting structured JSON from Google Gemini
   +---> 2. Call Gemini API (POST contents payload)
   +---> 3. Parse JSON Response fields (analysis, improvements, suggestions, safety)
   +---> 4. Save parsed Recommendation in MongoDB
```

---

## 🔍 Senior-Level Technical Findings & Anomalies

A code-level audit of the current repository highlights a few inconsistencies, bugs, and design recommendations:

### 1. The Activity Detail Rendering Discrepancy (UI Defect)
In [ActivityDetail.jsx](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/fitness-app-frontend/src/components/ActivityDetail.jsx), the frontend fetches data using:
```js
export const getActivityDetail = (id) => api.get(`/recommendations/activity/${id}`);
```
This endpoint routes to `RecommendationService` in the `aiservice` which returns the [Recommendation.java](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/aiservice/src/main/java/com/fitness/aiservice/model/Recommendation.java) model.
However, [ActivityDetail.jsx](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/fitness-app-frontend/src/components/ActivityDetail.jsx) tries to render:
```jsx
<Typography>Type: {activity.type}</Typography>
<Typography>Duration: {activity.duration} minutes</Typography>
<Typography>Calories Burned: {activity.caloriesBurned}</Typography>
```
Because the `Recommendation` entity does not contain `duration`, `caloriesBurned`, or a `type` property (it defines `activityType` instead of `type`), **these fields will render as blank/undefined in the user interface**.
* **Suggested Fix**: Update `ActivityDetail.jsx` to fetch both the activity from `activityservice` (`GET /api/activities/{id}`) and the recommendation from `aiservice` (`GET /api/recommendations/activity/{id}`), then combine the states locally for a complete UI.

### 2. Prop Destructuring Typo in Activity Form (React bug)
* **Issue**: In [App.jsx](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/fitness-app-frontend/src/App.jsx#L13), the prop is passed as:
  `onActivitiesAdded = {() => window.location.reload()}`
* But in [ActivityForm.jsx](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/fitness-app-frontend/src/components/ActivityForm.jsx#L6-L17), the prop is destructured and invoked as:
  `const ActivityForm = ({ onActivityAdded }) => { ... onActivityAdded(); ... }`
* **Impact**: Submitting a new activity will throw a `TypeError: onActivityAdded is not a function` and fail to refresh the activity feed.
* **Suggested Fix**: Align both names to `onActivityAdded`.

### 3. Synchronous WebClient Operations in Reactive Pipeline
* **Issue**: Inside the `activityservice` component [UserValidationService.java](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/activityservice/src/main/java/com/fitness/activityservice/service/UserValidationService.java#L23), the code performs a `.block()` call to validate the user.
* **Impact**: Blocking calls within Spring WebFlux or WebClient workflows bypasses the asynchronous advantages of reactive programming and risks thread starvation.
* **Suggested Fix**: Leverage non-blocking reactive chains if possible or use Spring MVC's standard RestTemplate/FeignClient if blockings are required.

### 4. Hardcoded Database and Broker Credentials
* **Issue**: Standard credentials (e.g. `admin@123` for Postgres in [user-service.yml](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/configserver/src/main/resources/config/user-service.yml)) are hardcoded in the configuration server's YAML files.
* **Suggested Fix**: Migrate these configurations to use Spring property placeholders combined with environment variables (e.g. `${SPRING_DATASOURCE_PASSWORD}`) to keep secrets out of source control.

---

## 🚀 Deployment & Operational Startup Sequence

To launch the microservices application successfully, run the components in the following chronological sequence:

1. **Start Infrastructure Services**:
   * **PostgreSQL**: Port `5432` (Ensure database `fitness_user_db` exists)
   * **MongoDB**: Port `27017` (Used by activity and AI services)
   * **RabbitMQ**: Port `5672` (Used for message queue integration)
   * **Keycloak**: Port `8181` (Create realm `fitness-oauth2` and configure client `oauth2-pkce-client`)

2. **Boot up Core System Services**:
   * Start **Eureka Server** (`eureka` on port `8761`)
   * Start **Config Server** (`configserver` on port `8888`)

3. **Start Core Microservices**:
   * Start **User Service** (`userservice` on port `8081`)
   * Start **Activity Service** (`activityservice` on port `8082`)
   * Configure environment variables `GEMINI_API_URL` and `GEMINI_API_KEY`, then start **AI Service** (`aiservice` on port `8083`)

4. **Start Routing Service**:
   * Start **API Gateway** (`gateway` on port `8080`)

5. **Start Client Frontend**:
   * Run `npm install` and `npm run dev` in [fitness-app-frontend](file:///c:/Users/kk/OneDrive%20-%20BENNETT%20UNIVERSITY/Desktop/Projects/fitness-app-microservices-main/fitness-app-frontend) to spin up the React application on `http://localhost:5173`.
