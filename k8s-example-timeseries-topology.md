# K8S Example: Timeseries Topology

[back](README#real-world-experiments)

- case
  - timeseries data from real world is published to a queue
  - a microservice consumes queue and writes into a timeseries DB
  - a microservice offers REST API for the data
  - an SPA shows the data via REST API
  - OAuth2 is configured to only allow owned data to show

## Architecture

1. `InfluxDB` time-series database - `stateful`
2. `Message Queue` handles incoming data and notifications - `stateful`
3. `Node.js Queue Consumer` consumes messages and writes to `InfluxDB` - `stateless`
4. `Node.js API Service` provides a REST API for reading `InfluxDB` - `stateless`
5. `Angular SPA` frontend SPA (static files via `Nginx`)
6. `OAuth2 Authentication` secures user access to the API - `stateless`

- deploy components as `pod`s
- managed by `deployment`s for `stateless` services and `StatefulSets` for `stateful`
  - Database & Queue - StatefulSet (InfluxDB, RabbitMQ)
  - Workers & API - Deployments (Queue Consumer, API)
  - Frontend & Authentication - Deployments (Angular, OAuth2 Proxy)
  - Ingress - Nginx to expose all services
- ensures scalability, security, reliability

### `InfluxDB`

- `PersistentVolume` for DB (persistent storage)
- `StatefulSet` for DB pods
- `Service` for internal communication

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: influxdb-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: influxdb
spec:
  serviceName: "influxdb"
  replicas: 1
  selector:
    matchLabels:
      app: influxdb
  template:
    metadata:
      labels:
        app: influxdb
    spec:
      containers:
        - name: influxdb
          image: influxdb:latest
          ports:
            - containerPort: 8086
          volumeMounts:
            - name: influxdb-storage
              mountPath: /var/lib/influxdb
  volumeClaimTemplates:
    - metadata:
        name: influxdb-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi

---
apiVersion: v1
kind: Service
metadata:
  name: influxdb
spec:
  selector:
    app: influxdb
  ports:
    - protocol: TCP
      port: 8086
      targetPort: 8086
  type: ClusterIP
```

### `RabbitMQ`

- `StatefulSet` for persistence
- `Service` for internal communication

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq
spec:
  serviceName: "rabbitmq"
  replicas: 1
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      containers:
        - name: rabbitmq
          image: rabbitmq:management
          ports:
            - containerPort: 5672 # AMQP protocol
            - containerPort: 15672 # Management UI
          env:
            - name: RABBITMQ_DEFAULT_USER
              value: "guest"
            - name: RABBITMQ_DEFAULT_PASS
              value: "guest"

---
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq
spec:
  selector:
    app: rabbitmq
  ports:
    - protocol: TCP
      port: 5672
      targetPort: 5672
    - protocol: TCP
      port: 15672
      targetPort: 15672
  type: ClusterIP
```

### `NodeJS` queue consumer worker

- stateless deployment
- consumes messages from `RabbitMQ`
- writes parsed data into `InfluxDB`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: queue-consumer
spec:
  replicas: 2
  selector:
    matchLabels:
      app: queue-consumer
  template:
    metadata:
      labels:
        app: queue-consumer
    spec:
      containers:
        - name: queue-consumer
          image: myrepo/queue-consumer:latest
          env:
            - name: RABBITMQ_URL
              value: "amqp://rabbitmq:5672"
            - name: INFLUXDB_URL
              value: "http://influxdb:8086"
```

### `NodeJS` API service

- stateless deployment
- exposes REST API for frontend
- uses OAuth2

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-service
  template:
    metadata:
      labels:
        app: api-service
    spec:
      containers:
        - name: api-service
          image: myrepo/api-service:latest
          ports:
            - containerPort: 3000
          env:
            - name: INFLUXDB_URL
              value: "http://influxdb:8086"
            - name: OAUTH2_PROVIDER_URL
              value: "https://auth.example.com"

---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api-service
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: ClusterIP
```

### `SPA` Angular frontend

- served via `Nginx`
- exposed via `LoadBalancer`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: myrepo/frontend:latest
          ports:
            - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```

### `OAuth2`

- external provider (e.g., Keycloak, Auth0, Google)
- uses an OAuth2 proxy to enforce authentication

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: oauth2-proxy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: oauth2-proxy
  template:
    metadata:
      labels:
        app: oauth2-proxy
    spec:
      containers:
        - name: oauth2-proxy
          image: quay.io/oauth2-proxy/oauth2-proxy
          args:
            - --provider=google
            - --client-id=<CLIENT_ID>
            - --client-secret=<CLIENT_SECRET>
            - --email-domain=example.com
            - --upstream=http://api-service:80
          ports:
            - containerPort: 4180

---
apiVersion: v1
kind: Service
metadata:
  name: oauth2-proxy
spec:
  selector:
    app: oauth2-proxy
  ports:
    - protocol: TCP
      port: 80
      targetPort: 4180
  type: ClusterIP
```

### Ingress by `Ingress` (Alternative 1)

- expose everything through a single Nginx Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /api/
            pathType: Prefix
            backend:
              service:
                name: oauth2-proxy
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

### Ingress by `Gateway` (Alternative 2)

- `Ingress` is becoming obsolete, use `Gateway` instead
- refactoring, use `Gateway` API to expose:
  - frontend at `/`
  - API service (secured via OAuth2 Proxy) at `/api/`
  - (optional) direct access route to `RabbitMQ` UI for debugging at `/rabbitmq/`
- result
  - user visits `myapp.example.com`
  - gateway forwards
    - `/` - frontend
    - `/api/` - OAuth2 Proxy - API Service
    - `/rabbitmq/` (optional) - Queue UI
  - improved security, flexibility, future scalability (e.g., adding gRPC or WebSockets)

#### Install Gateway API in Cluster

```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/latest/download/standard-install.yaml
```

#### Define Gateway for external access

- add Gateway as entry point
- create on port 80 handling incoming traffic
- use Nginx Gateway (could be Traefik, Istio etc.)

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: main-gateway
spec:
  gatewayClassName: nginx # Replace with your Gateway implementation (e.g., Istio, Traefik)
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: Same
```

#### Routing via `HTTPRoutes`

- create route per service

```yaml
# frontend
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: frontend-route
spec:
  parentRefs:
    - name: main-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: frontend
          port: 80

---
# API with OAuth2
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: api-route
spec:
  parentRefs:
    - name: main-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api/
      backendRefs:
        - name: oauth2-proxy
          port: 80

---
# Queue UI
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: rabbitmq-route
spec:
  parentRefs:
    - name: main-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /rabbitmq/
      backendRefs:
        - name: rabbitmq
          port: 15672
```

### Add TLS to Gateway

#### Add Cert-Manager (if not already installed)

- `Cert-Manager` automates SSL cert renewal

```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

#### Create a `ClusterIssuer` for `Let's Encrypt`

- use `Let's Encrypt` to generate valid SSL certificates
- replace `myapp.example.com` with real domain
- fyi apply with e.g. `kubectl apply -f cluster-issuer.yaml`

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com # Change to your real email
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx # Change if using another gateway (e.g., Traefik)
```

#### Request certificate

- create a `Certificate` resource that `Cert-Manager` will use to obtain a `Let's Encrypt` TLS cert
- following generates a TLS certificate stored in `myapp-tls-secret`
- fyi apply with e.g. `kubectl apply -f certificate.yaml`

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-tls
spec:
  secretName: myapp-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - myapp.example.com # Replace with your actual domain
```

#### Modify the `Gateway` to use `HTTPS`

- apply with e.g. `kubectl apply -f gateway.yaml`

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: main-gateway
spec:
  gatewayClassName: nginx
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: Same
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        certificateRefs:
          - name: myapp-tls-secret
      allowedRoutes:
        namespaces:
          from: Same
```

#### Redirect `HTTP` to `HTTPS`

- enforce HTTPS with a redirect rule using an `HTTPRoute`
- apply with e.g. `kubectl apply -f http-redirect.yaml`

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: http-redirect
spec:
  parentRefs:
    - name: main-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      filters:
        - type: RequestRedirect
          requestRedirect:
            scheme: https
```

### Integrate OAuth2 with Cert-Manager

- OAuth2 proxy secures API access
- Cert-Manager provisions TLS certificates for OAuth2 endpoints
- user must authenticate before accessing protected services

1. deploy an OAuth2 provider (Keycloak/Auth0/Google)
2. deploy OAuth2 Proxy (handles authentication)
3. configure Cert-Manager to secure OAuth2 proxy
4. update Gateway to enforce authentication

#### Deploy OAuth2 provider

- use any OAuth2 provider (Keycloak, Google, Auth0)
- e.g. deploy and expose Keycloak

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: keycloak
spec:
  replicas: 1
  selector:
    matchLabels:
      app: keycloak
  template:
    metadata:
      labels:
        app: keycloak
    spec:
      containers:
        - name: keycloak
          image: quay.io/keycloak/keycloak:latest
          args: ["start-dev"]
          env:
            - name: KEYCLOAK_ADMIN
              value: "admin"
            - name: KEYCLOAK_ADMIN_PASSWORD
              value: "admin"
          ports:
            - containerPort: 8080

---
apiVersion: v1
kind: Service
metadata:
  name: keycloak
spec:
  selector:
    app: keycloak
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

#### Deploy OAuth2 proxy

- OAuth2 proxy acts as a gatekeeper for the API (ensures user auth)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: oauth2-proxy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: oauth2-proxy
  template:
    metadata:
      labels:
        app: oauth2-proxy
    spec:
      containers:
        - name: oauth2-proxy
          image: quay.io/oauth2-proxy/oauth2-proxy
          args:
            - --provider=oidc
            - --oidc-issuer-url=https://keycloak.example.com/realms/myrealm
            - --client-id=my-client-id
            - --client-secret=my-client-secret
            - --cookie-secret=REPLACE_WITH_RANDOM_STRING
            - --upstream=http://api-service:80
          ports:
            - containerPort: 4180

---
apiVersion: v1
kind: Service
metadata:
  name: oauth2-proxy
spec:
  selector:
    app: oauth2-proxy
  ports:
    - protocol: TCP
      port: 80
      targetPort: 4180
```

#### Secure OAuth2 Proxy with Cert-Manager

- issue a TLS certificate for OAuth2 proxy
- apply with e.g. `kubectl apply -f oauth2-proxy-cert.yaml`

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: oauth2-proxy-cert
spec:
  secretName: oauth2-proxy-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - oauth.example.com # Change this to your domain
```

#### Update Gateway to enforce auth

- modify `Gateway` to ensure only authenticated users can access the API
- modify the `HTTPRoute` to enforce OAuth2 authentication

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: main-gateway
spec:
  gatewayClassName: nginx
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        certificateRefs:
          - name: myapp-tls-secret
      allowedRoutes:
        namespaces:
          from: Same

---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: api-auth-route
spec:
  parentRefs:
    - name: main-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api/
      backendRefs:
        - name: oauth2-proxy
          port: 80
```

### Restrict users to their own data

- introduce JWT claims OAuth2
  - user can access only own data
  - JWT tokens include user identity (sub, email, roles etc.)
  - API verifies JWT claims before responding

1. modify OAuth2 provider to include user claims
2. configure OAuth2 proxy to pass JWT to API
3. update API to extract and enforce user claims
4. test with different users

#### Include User Claims

- e.g. in Keycloak configure to include custom claims (e.g., user_id, email, role) in JWT
  - log in to Keycloak admin panel
  - select your realm, go to clients
  - select your API client (my-client-id)
  - go to client scopes - mappers - add mapper
  - choose "user attribute"
    - name: user_id
    - user attribute: sub
    - token claim name: user_id
    - claim type: string
    - include in ID/Access token: yes
  - repeat for email, roles
- JWT now has additional data

```json
{
  "sub": "123456789",
  "email": "user@example.com",
  "roles": ["user"]
}
```

#### Pass JWT to API

- modify OAuth2 proxy to pass JWT as `Authorization: Bearer` header
  - API will receive JWT token in every request: `Authorization: Bearer <JWT_TOKEN>`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: oauth2-proxy
spec:
  template:
    spec:
      containers:
        - name: oauth2-proxy
          image: quay.io/oauth2-proxy/oauth2-proxy
          args:
            - --provider=oidc
            - --oidc-issuer-url=https://keycloak.example.com/realms/myrealm
            - --client-id=my-client-id
            - --client-secret=my-client-secret
            - --cookie-secret=REPLACE_WITH_RANDOM_STRING
            - --upstream=http://api-service:80
            - --pass-authorization-header=true
            - --pass-access-token=true
```

#### Update API to Extract and Enforce User Claims

- modify API Service to validate JWTs and restrict access to user-owned data
- e.g. for express.js `npm install jsonwebtoken express-jwt`

```javascript
const express = require("express");
const jwt = require("jsonwebtoken");

const app = express();
const INFLUXDB_URL = process.env.INFLUXDB_URL;

// Middleware to verify JWT
const authenticateJWT = (req, res, next) => {
  const authHeader = req.headers.authorization;
  if (!authHeader) {
    return res.status(401).json({ message: "Unauthorized" });
  }

  const token = authHeader.split(" ")[1];
  jwt.verify(token, process.env.OAUTH2_PUBLIC_KEY, (err, user) => {
    if (err) return res.status(403).json({ message: "Forbidden" });

    req.user = user; // Attach user to request
    next();
  });
};

// Fetch user-specific data from InfluxDB
app.get("/api/data", authenticateJWT, async (req, res) => {
  const userId = req.user.sub; // Extract user ID from JWT claims

  // Query InfluxDB for only this user's data
  const query = `SELECT * FROM data WHERE user_id = '${userId}'`;
  try {
    const result = await fetch(
      `${INFLUXDB_URL}/query?db=mydb&q=${encodeURIComponent(query)}`
    );
    const jsonData = await result.json();
    res.json(jsonData);
  } catch (error) {
    res.status(500).json({ message: "Database error", error });
  }
});

app.listen(3000, () => console.log("API Service running on port 3000"));
```

#### Test with different users

- authenticate with Keycloak and get a JWT token
- each user should receive their own data
- call API with token

```sh
curl -X POST "https://keycloak.example.com/realms/myrealm/protocol/openid-connect/token" \
     -d "client_id=my-client-id" \
     -d "client_secret=my-client-secret" \
     -d "username=user@example.com" \
     -d "password=mypassword" \
     -d "grant_type=password"
```

```sh
curl -H "Authorization: Bearer <JWT_TOKEN>" https://myapp.example.com/api/data
```

### RBAC (role based access control) with JWT

- regular users can only access their own data
- admins can access all data
- API verifies user roles dynamically

1. modify OAuth2 provider to include roles in JWT
2. update OAuth2 proxy to forward roles
3. modify API to enforce RBAC
4. test API with different roles

#### Include Roles in JWT

- Keycloak role configuration
  - log in to Keycloak admin panel
  - select your realm - go to roles
  - create two roles
    - user - for regular users
    - admin - for users with full access
  - assign roles to users in users - role mappings
- Keycloak token claim mapping
  - go to client scopes - mappers - add mapper
  - choose "User Realm Role"
    - Name: roles
    - Token Claim Name: roles
    - Claim Type: Array
    - Include in ID & Access Token: yes
- JWT tokens will contain user roles

```json
{
  "sub": "123456789",
  "email": "user@example.com",
  "roles": ["user"]
}

{
  "sub": "987654321",
  "email": "admin@example.com",
  "roles": ["admin"]
}
```

#### Update OAuth2 Proxy to Forward Roles

- pass JWT roles to the API

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: oauth2-proxy
spec:
  template:
    spec:
      containers:
        - name: oauth2-proxy
          image: quay.io/oauth2-proxy/oauth2-proxy
          args:
            - --provider=oidc
            - --oidc-issuer-url=https://keycloak.example.com/realms/myrealm
            - --client-id=my-client-id
            - --client-secret=my-client-secret
            - --cookie-secret=REPLACE_WITH_RANDOM_STRING
            - --upstream=http://api-service:80
            - --pass-authorization-header=true
            - --pass-access-token=true
```

#### Enforce API RBAC

- e.g. `npm install jsonwebtoken express-jwt`

```javascript
const express = require("express");
const jwt = require("jsonwebtoken");

const app = express();
const INFLUXDB_URL = process.env.INFLUXDB_URL;

// Middleware to verify JWT and extract user info
const authenticateJWT = (req, res, next) => {
  const authHeader = req.headers.authorization;
  if (!authHeader) {
    return res.status(401).json({ message: "Unauthorized" });
  }

  const token = authHeader.split(" ")[1];
  jwt.verify(token, process.env.OAUTH2_PUBLIC_KEY, (err, user) => {
    if (err) return res.status(403).json({ message: "Forbidden" });

    req.user = user; // Attach user to request
    next();
  });
};

// Middleware to enforce roles
const authorizeRole = (role) => {
  return (req, res, next) => {
    if (!req.user || !req.user.roles.includes(role)) {
      return res
        .status(403)
        .json({ message: "Forbidden: Insufficient permissions" });
    }
    next();
  };
};

// Regular users can only fetch their own data
app.get("/api/data", authenticateJWT, async (req, res) => {
  const userId = req.user.sub;

  const query = `SELECT * FROM data WHERE user_id = '${userId}'`;
  try {
    const result = await fetch(
      `${INFLUXDB_URL}/query?db=mydb&q=${encodeURIComponent(query)}`
    );
    const jsonData = await result.json();
    res.json(jsonData);
  } catch (error) {
    res.status(500).json({ message: "Database error", error });
  }
});

// Admins can fetch all users' data
app.get(
  "/api/admin/data",
  authenticateJWT,
  authorizeRole("admin"),
  async (req, res) => {
    const query = `SELECT * FROM data`;
    try {
      const result = await fetch(
        `${INFLUXDB_URL}/query?db=mydb&q=${encodeURIComponent(query)}`
      );
      const jsonData = await result.json();
      res.json(jsonData);
    } catch (error) {
      res.status(500).json({ message: "Database error", error });
    }
  }
);

app.listen(3000, () => console.log("API Service running on port 3000"));
```

#### Test with different roles

- get regular user
- call api (should return own data)
- call admin api (should return 403)
- get admin user and try admin api (should return all data)

```sh
curl -X POST "https://keycloak.example.com/realms/myrealm/protocol/openid-connect/token" \
     -d "client_id=my-client-id" \
     -d "client_secret=my-client-secret" \
     -d "username=user@example.com" \
     -d "password=mypassword" \
     -d "grant_type=password"
```

```sh
curl -H "Authorization: Bearer <USER_JWT>" https://myapp.example.com/api/data
```

```sh
curl -H "Authorization: Bearer <USER_JWT>" https://myapp.example.com/api/admin/data
```

```sh
curl -X POST "https://keycloak.example.com/realms/myrealm/protocol/openid-connect/token" \
     -d "client_id=my-client-id" \
     -d "client_secret=my-client-secret" \
     -d "username=admin@example.com" \
     -d "password=adminpassword" \
     -d "grant_type=password"
```

### OAuth2 PKCE (Proof Key for Code Exchange) flow in the Angular frontend

- secure authentication without storing client secrets
- better security against authorization code interception
- seamless login flow for SPAs

1. configure OAuth2 provider for PKCE
2. install & configure Angular OAuth2
3. implement login flow in Angular
4. protect routes & handle tokens
5. test the authentication flow

#### Configure OAuth2 Provider for PKCE

- Keycloak
  - go to Keycloak admin panel - clients
  - select your frontend client (e.g., angular-client)
  - update these settings:
    - Access Type: public
    - Valid Redirect URIs: http://localhost:4200/\*
    - Web Origins: http://localhost:4200
    - OAuth 2.0 Flow: Authorization Code
    - Proof Key for Code Exchange (PKCE): Enabled
  - now Keycloak supports PKCE authentication for Angular

#### Add Angular OAuth2 Library

- install `npm install angular-oauth2-oidc`
- add `auth.config.ts` file

```typescript
import { AuthConfig } from "angular-oauth2-oidc";

export const authConfig: AuthConfig = {
  issuer: "https://keycloak.example.com/realms/myrealm",
  clientId: "angular-client",
  redirectUri: window.location.origin,
  responseType: "code",
  scope: "openid profile email",
  showDebugInformation: true,
  strictDiscoveryDocumentValidation: false,
  useSilentRefresh: true,
  oidc: true,
  requireHttps: true, // Set to false if using localhost
  disablePKCE: false, // Enable PKCE Flow
};
```

#### Implement Login Flow

- example for old app.module.ts
- and app.component.ts

```typescript
import { NgModule } from "@angular/core";
import { BrowserModule } from "@angular/platform-browser";
import { AppComponent } from "./app.component";
import { OAuthModule } from "angular-oauth2-oidc";

@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule,
    OAuthModule.forRoot(), // Import OAuth2 module
  ],
  providers: [],
  bootstrap: [AppComponent],
})
export class AppModule {}
```

```typescript
import { Component, OnInit } from "@angular/core";
import { OAuthService } from "angular-oauth2-oidc";
import { authConfig } from "./auth.config";

@Component({
  selector: "app-root",
  templateUrl: "./app.component.html",
  styleUrls: ["./app.component.css"],
})
export class AppComponent implements OnInit {
  constructor(private oauthService: OAuthService) {}

  ngOnInit(): void {
    this.configureAuth();
  }

  configureAuth() {
    this.oauthService.configure(authConfig);
    this.oauthService.loadDiscoveryDocumentAndTryLogin();
  }

  login() {
    this.oauthService.initLoginFlow();
  }

  logout() {
    this.oauthService.logOut();
  }

  get isLoggedIn(): boolean {
    return this.oauthService.hasValidAccessToken();
  }

  get userProfile(): any {
    return this.oauthService.getIdentityClaims();
  }
}
```

#### Protect Routes & Handle Tokens

- example for old `app-routing.module.ts`
- and for `auth.guard.ts`

```typescript
import { NgModule } from "@angular/core";
import { RouterModule, Routes } from "@angular/router";
import { AuthGuard } from "./auth.guard";
import { DashboardComponent } from "./dashboard/dashboard.component";
import { LoginComponent } from "./login/login.component";

const routes: Routes = [
  { path: "login", component: LoginComponent },
  {
    path: "dashboard",
    component: DashboardComponent,
    canActivate: [AuthGuard],
  },
  { path: "", redirectTo: "/dashboard", pathMatch: "full" },
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule],
})
export class AppRoutingModule {}
```

```typescript
import { Injectable } from "@angular/core";
import { CanActivate, Router } from "@angular/router";
import { OAuthService } from "angular-oauth2-oidc";

@Injectable({ providedIn: "root" })
export class AuthGuard implements CanActivate {
  constructor(private oauthService: OAuthService, private router: Router) {}

  canActivate(): boolean {
    if (!this.oauthService.hasValidAccessToken()) {
      this.router.navigate(["/login"]);
      return false;
    }
    return true;
  }
}
```

#### Test Auth Flow

- start Keycloak, API, Angular
- open `http://localhost:4200`
- click login - redirects to Keycloak
- authenticate with your credentials
- redirects back to Angular with an access token
- API calls now include `Authorization: Bearer <JWT_TOKEN>`

### Secrets Management

- secure sensitive information (client secrets, DB credentials, API keys) using Kubernetes Secrets
- integrate it with OAuth2, InfluxDB, Microservices

1. Use Kubernetes secrets for storing sensitive data
2. Mount secrets into deployments (OAuth2 Proxy, API, InfluxDB, etc.)
3. (Optional) Use external secret managers (e.g., HashiCorp Vault, AWS Secrets Manager)

#### Create Kubernetes Secrets

- add yaml
- convert to Base64
- apply for example with `kubectl apply -f app-secrets.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  OAUTH2_CLIENT_ID: bXktY2xpZW50LWlk # Base64 encoded
  OAUTH2_CLIENT_SECRET: c2VjcmV0LXZhbHVl # Base64 encoded
  INFLUXDB_USERNAME: aW5mbHV4dXNlcg== # Base64 encoded
  INFLUXDB_PASSWORD: c2VjcmV0cGFzc3dvcmQ= # Base64 encoded
  JWT_PUBLIC_KEY: LS0tLS1CRUdJTiBQUk... # Base64 encoded JWT Public Key
```

```sh
echo -n 'my-client-id' | base64
echo -n 'secret-value' | base64
```

### Mount Secrets into Deployments

- modify OAuth2 Proxy, API, and InfluxDB Deployments to use secrets

```yaml
# OAuth2 Proxy Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: oauth2-proxy
spec:
  template:
    spec:
      containers:
        - name: oauth2-proxy
          image: quay.io/oauth2-proxy/oauth2-proxy
          env:
            - name: OAUTH2_PROXY_CLIENT_ID
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: OAUTH2_CLIENT_ID
            - name: OAUTH2_PROXY_CLIENT_SECRET
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: OAUTH2_CLIENT_SECRET

---
# Node.js API Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  template:
    spec:
      containers:
        - name: api-service
          image: my-node-api
          env:
            - name: INFLUXDB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: INFLUXDB_USERNAME
            - name: INFLUXDB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: INFLUXDB_PASSWORD
            - name: JWT_PUBLIC_KEY
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: JWT_PUBLIC_KEY

---
#InfluxDB Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: influxdb
spec:
  template:
    spec:
      containers:
        - name: influxdb
          image: influxdb
          env:
            - name: INFLUXDB_ADMIN_USER
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: INFLUXDB_USERNAME
            - name: INFLUXDB_ADMIN_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: INFLUXDB_PASSWORD
```

#### Use External Secret Managers (Optional)

- more secure approach instead of Kubernetes Secrets e.e. HashiCorp Vault
- install Vault into Kubernetes
  - `helm repo add hashicorp https://helm.releases.hashicorp.com`
  - `helm install vault hashicorp/vault`
- store a secret in Vault
  - `vault kv put secret/api OAUTH2_CLIENT_ID="my-client-id"`
- deploy Vault Agent Injector to automatically inject secrets into Pods

### Add Helm Charts for Easy Deployment

- Helm packages Kubernetes yamls into charts making deployment easy and maintainable

1. install Helm
2. create a Helm chart
3. template secrets, ConfigMaps, and Deployments
4. deploy with Helm
5. upgrade, rollback, and manage releases

#### Install Helm

- `curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash`
- verify by `helm version`

#### Create a Helm Chart

- navigate to your project directory and create a Helm chart
  - `helm create myapp`
  - `cd myapp`
  - results in following structure

```bash
myapp/
  ├── charts/               # (For dependencies)
  ├── templates/            # (YAML templates for Kubernetes resources)
  │   ├── deployment.yaml   # (API, OAuth2 Proxy, etc.)
  │   ├── service.yaml      # (Service definitions)
  │   ├── ingress.yaml      # (Gateway/Ingress)
  │   ├── secret.yaml       # (Kubernetes Secrets)
  ├── values.yaml           # (Default values for templates)
  ├── Chart.yaml            # (Chart metadata)
```

#### Template Secrets, ConfigMaps, and Deployments

- `values.yaml` file for environment-specific values

```yaml
replicaCount: 1

image:
  api: my-node-api:latest
  oauth2Proxy: quay.io/oauth2-proxy/oauth2-proxy:latest
  influxdb: influxdb:latest

service:
  api:
    port: 3000
  influxdb:
    port: 8086

oauth2:
  clientID: "my-client-id"
  clientSecret: "my-client-secret"

influxdb:
  username: "influxuser"
  password: "influxpassword"

jwt:
  publicKey: "LS0tLS1CRUdJTiBQUk..."
```

- secret template `templates/secret.yaml` file dynamically inject secrets from `values.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  # {{ }} <-- should be formatted like this actually
  OAUTH2_CLIENT_ID: { { .Values.oauth2.clientID | b64enc } }
  OAUTH2_CLIENT_SECRET: { { .Values.oauth2.clientSecret | b64enc } }
  INFLUXDB_USERNAME: { { .Values.influxdb.username | b64enc } }
  INFLUXDB_PASSWORD: { { .Values.influxdb.password | b64enc } }
  JWT_PUBLIC_KEY: { { .Values.jwt.publicKey | b64enc } }
```

- deployment template `templates/deployment.yaml` for modifying

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  # {{ }} <-- should be formatted like this actually
  replicas: { { .Values.replicaCount } }
  template:
    spec:
      containers:
        - name: api-service
          image: "{{ .Values.image.api }}"
          env:
            - name: INFLUXDB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: INFLUXDB_USERNAME
            - name: INFLUXDB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: INFLUXDB_PASSWORD
            - name: JWT_PUBLIC_KEY
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: JWT_PUBLIC_KEY
```

- service template `templates/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: ClusterIP
  ports:
    # {{ }} <-- should be formatted like this actually
    - port: { { .Values.service.api.port } }
      targetPort: 3000
  selector:
    app: api-service
```

- Gateway/Ingress template `templates/ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Gateway
metadata:
  name: app-gateway
spec:
  gatewayClassName: nginx
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        certificateRefs:
          - name: tls-secret
      routes:
        - name: api-route
          match:
            path: /api
          backend:
            service:
              name: api-service
              port: 3000
```

#### Deploy with Helm

- `helm install myapp ./myapp`
- check with `kubectl get pods`

#### Upgrade, Rollback, and Manage Releases

- after changing `values.yaml` do `helm upgrade myapp ./myapp`
- rollback to previous with `helm rollback myapp 1`
- uninstall chart with `helm uninstall myapp`

### Multi-Environment Helm Values

- dev, staging, production deployments
- customize configurations per environment

1. create separate values.yaml file per env
2. deploy with Helm using `--values` flag

#### Create Environment-Specific Values Files

- `values-dev.yaml`
- `values-staging.yaml`
- `values-prod.yaml`

```yaml
replicaCount: 1

image:
  api: my-node-api:latest
  oauth2Proxy: quay.io/oauth2-proxy/oauth2-proxy:latest
  influxdb: influxdb:latest

service:
  api:
    port: 3000
  influxdb:
    port: 8086

oauth2:
  clientID: "dev-client-id"
  clientSecret: "dev-secret"

influxdb:
  username: "devuser"
  password: "devpassword"

jwt:
  publicKey: "LS0tLS1CRUdJTiBQUk..." # Development JWT public key
```

```yaml
replicaCount: 2

image:
  api: my-node-api:staging
  oauth2Proxy: quay.io/oauth2-proxy/oauth2-proxy:latest
  influxdb: influxdb:staging

service:
  api:
    port: 3000
  influxdb:
    port: 8086

oauth2:
  clientID: "staging-client-id"
  clientSecret: "staging-secret"

influxdb:
  username: "staginguser"
  password: "stagingpassword"

jwt:
  publicKey: "LS0tLS1CRUdJTiBQUk..." # Staging JWT public key
```

```yaml
replicaCount: 3

image:
  api: my-node-api:prod
  oauth2Proxy: quay.io/oauth2-proxy/oauth2-proxy:latest
  influxdb: influxdb:prod

service:
  api:
    port: 3000
  influxdb:
    port: 8086

oauth2:
  clientID: "prod-client-id"
  clientSecret: "prod-secret"

influxdb:
  username: "produser"
  password: "prodpassword"

jwt:
  publicKey: "LS0tLS1CRUdJTiBQUk..." # Production JWT public key
```

#### Deploy Using Helm with `--values` Flag

- `helm install myapp-dev ./myapp -f values-dev.yaml`
- `helm install myapp-staging ./myapp -f values-staging.yaml`
- `helm install myapp-prod ./myapp -f values-prod.yaml`
