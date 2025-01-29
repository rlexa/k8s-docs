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

# TODO

- next: Need Helm charts for easy deployment?
- next: Want OAuth2 login for Angular SPA (PKCE Flow)?
- next: add secrets management?
- question: so NodeJS api does not have to be started in HTTP itself, endpoint proxy makes HTTPS instead?
- question: Keycloak vs. others?
