## Issue 1: nginx routing incorrect
- Nginx config file in conf.d/default.conf proxy the request to the api upstream at port 3001, which is wrong because api container running at port 3000
- no route for /status endpoint 
- Request to root / return download file instead of welcome page

#### <u>Diagnose</u>
- Hitting `https://localhost:8080/api/users`  returned 503 and explitcitly show in the logs:
```bash
nginx-1     | 2026/04/11 02:48:15 [error] 28#28: *1 connect() failed (111: Connection refused) while connecting to upstream, client: 192.168.65.1, server: , request: "GET /api/users HTTP/1.1", upstream: "http://172.18.0.4:3001/api/users", host: "localhost:8080"
```
Indicate that nginx forward traffic to api container port 3001.
But the api is listening on port 3000 as defined in the code:
```js
app.listen(3000, () => console.log("API running on 3000"));
```

#### <u>Fix</u>
- Correct the nginx config file to point to api port 3000:
```nginx
location /api/ {
        proxy_pass http://api:3000;
    }
```

- Add the /status location and forward to api upstream:
```nginx
    location /status {
        proxy_pass http://api:3000;
    }
```

- Instruct nginx to return plain text when hitting the root location:
```nginx
location = / {
        default_type text/plain;
        return 200 "Welcome to the platform\n";
    }
```


---
## Issue 2: database password hardcoded 
Postgres database password hardcoded in the index.js and docker-compose, exposing the risk of security when commiting to github
```js
...
const pool = new Pool({
  host: process.env.DB_HOST,
  user: "postgres",
  password: "postgres",
  database: "postgres",
  port: 5432,
});
```

#### <u>Fix:</u>
- Move the password to .env, avoid any hardcoding and add the .env to .gitignore

```js
...
const pool = new Pool({
  host: process.env.DB_HOST,
  user: process.env.POSTGRES_USER || "postgres",
  password: process.env.POSTGRES_PASSWORD || "postgres",
  database: process.env.POSTGRES_DB || "postgres",
  port: 5432,
});
```
and in docker-compose:
```docker
postgres:
    image: postgres:15
    env_file:
      - .env
```


---
## Issue 3: api Dockerfile running as root
- running docker image with root user is insecure
- security best practice is “least privilege”
- Change to `USER node` instead

```dockerfile
FROM node:20-alpine

WORKDIR /app
COPY package.json .
RUN npm install

COPY src ./src

USER node
CMD ["node", "src/index.js"]
```

---
## Issue 4: postgres init.sql script never be executed
The sql script in `postgres/init.sql` to tune the postgres `max_connections` parameter never be executed.


Diagnose by login to postgres container and check for the actual value is different from what being set in the script:

```bash
postgres=# SHOW max_connections;
 max_connections
-----------------
 100
(1 row)
```

Beside, this value is considered low, we need a reasonable number of connections in production for better performance.

#### <u>Fix: </u>
Mount the script the postgres container in docker-compose. 

```docker
postgres:
  image: postgres:15
  env_file:
    - .env
  volumes:
    - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
```

---
## Issue 5: no healthcheck for postgres and redis
- The compose file does not define `healthcheck` blocks for `postgres` or `redis`.
- `depends_on` only controls startup order, it does not guarantee that the services are ready to accept connections.
- As a result, the API can start and fail to connect to Postgres or Redis even though the containers are running.

#### <u>Fix:</u>
Add healthchecks to the Compose services and use a startup script or retry logic in the API if necessary.

```yaml
postgres:
  image: postgres:15
  env_file:
    - .env
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U postgres"]
    interval: 10s
    timeout: 5s
    retries: 5

redis:
  image: redis:7
  healthcheck:
    test: ["CMD-SHELL", "redis-cli ping"]
    interval: 10s
    timeout: 5s
    retries: 5
```

---
## Monitor and Alert:

| Metric | Alert condition |
|---|---|
| GET /status reponse | Reponse code != 200 | 
| Nginx error rate | 5xx code > 5% errors in 5m |
| API latency | 95th percentile > 300ms |
| DB connection count | > 80% of max connections |
| Redis memory | > 80% |
| Infra resources usage | > 80% usage
| Container health | any unhealthy container in 3 consecutive check |

Tools used for monitoring and logging:
- Prometheus and grafana for monitoring and alerting
- Centralised logging system ELK/Loki to store log of all components

---
## Prevent this in production
- Migrate to a more reliable container orchestrator like kubernetes
- Test in CI/CD pipeline. Healthcheck the service to identify issue early
- Move secret to external secret management system like Hashicorp vault or docker secrets
- Tuning the postgres `max_connections` parameter for production environment (~100)
- Persist postgres data in a volume so data will not be erased when container restart
- Add resource request/limit for each component 
- Add healthcheck to each service to ensure the readiness at startup