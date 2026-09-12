# Scenario 1

Use one heading for each problem.
Write what was wrong and how you fixed it.
Paste the config you changed (only the changed part).
Paste the commands you used.
Write every step you tried, even guesses.

English is better. Persian is OK.

# Problem 1: DNS

### What was wrong

When I wanted to install the required packages using:

```bash
sudo apt update
```

I got a **"Could not resolve"** error.

To determine whether the problem was related to the network connection or DNS, I tested connectivity by pinging Google's IP address:

```bash
ping -c 3 8.8.8.8
```

The ping was successful, which indicated that the network connection itself was working.

I then checked the DNS configuration:

```bash
cat /etc/resolv.conf
```

It showed:

```text
nameserver 127.0.0.1
```

This indicated that DNS queries were being sent to localhost.

I then checked the status of `systemd-resolved`:

```bash
systemctl status systemd-resolved
```

The output showed that `systemd-resolved` was not running.

I started the service:

```bash
systemctl start systemd-resolved
```

Then I checked its DNS configuration:

```bash
resolvectl status
```

It showed the following DNS servers:

```text
217.218.127.127
217.218.155.155
```

However, DNS resolution was still not working.

To find out where `systemd-resolved` was listening for DNS requests, I checked port 53:

```bash
ss -lunp | grep ':53'
```

The output showed that `systemd-resolved` was listening on:

```text
127.0.0.53:53
127.0.0.54:53
```

while `/etc/resolv.conf` was still pointing to:

```text
127.0.0.1
```

This was the main clue. The DNS configuration in `/etc/resolv.conf` was pointing to the wrong address.

I checked the actual configuration used by `systemd-resolved`:

```bash
cat /run/systemd/resolve/resolv.conf
```

It contained the correct DNS servers:

```text
nameserver 217.218.127.127
nameserver 217.218.155.155
```

Therefore, I fixed `/etc/resolv.conf` by creating a symbolic link to the correct configuration:

```bash
ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

Finally, I tested DNS resolution again:

```bash
getent hosts google.com
ping -c 3 google.com
```

Both tests were successful.

### Conclusion

The network connection was working correctly, but DNS resolution was broken. The problem was caused by an incorrect `/etc/resolv.conf` configuration. It pointed to `127.0.0.1`, while `systemd-resolved` was listening on `127.0.0.53` and `127.0.0.54`.

---

# Problem 2: Docker Compose

After fixing the DNS problem, I tried to use Docker Compose, but the Compose command was not available.

I checked:

```bash
docker compose version
```

Docker returned an unknown command error.

I then checked whether the `docker-compose` package was available:

```bash
apt-cache policy docker-compose
```

The output showed that version `1.29.2-6ubuntu1` was available.

I installed it using:

```bash
apt install -y docker-compose
```

Then I verified the installation:

```bash
docker-compose --version
```

The output confirmed:

```text
docker-compose version 1.29.2
```

Therefore, Docker Compose was installed successfully and I could continue with the deployment.

---

# Problem 3: Docker Compose / Application

First, I checked whether the Docker Compose configuration was valid:

```bash
docker-compose config
```

The configuration was valid, so I started the services:

```bash
docker-compose up -d
```

The following containers were started:

- nginx
- backend
- db

I checked their status:

```bash
docker-compose ps
```

All three containers were running.

However, when I tested the application:

```bash
curl -i http://localhost/graph
```

I received:

```text
HTTP/1.1 502 Bad Gateway
```

Since nginx was returning a `502 Bad Gateway`, I suspected that nginx was unable to communicate correctly with the backend.

Instead of changing the nginx configuration immediately, I checked the backend logs:

```bash
docker-compose logs --tail=50 backend
```

The important error was:

```text
could not translate host name "db" to address:
Name or service not known
```

This indicated that the backend could not resolve the hostname `db`.

---

# Problem 4: Docker Networking

To determine why the backend could not resolve `db`, I inspected the Docker networks of both containers.

First, I checked the backend:

```bash
docker inspect service-catalog_backend_1 \
  --format '{{json .NetworkSettings.Networks}}'
```

The backend was connected only to:

```text
nginx-backend-net
```

Then I checked the database container:

```bash
docker inspect service-catalog_db_1 \
  --format '{{json .NetworkSettings.Networks}}'
```

The database was connected only to:

```text
backend-db-net
```

This was the key clue.

The backend and the database were not connected to a common Docker network. Therefore, Docker's internal DNS could not resolve the hostname `db` from the backend container.

The network configuration was effectively:

```text
nginx
   |
nginx-backend-net
   |
backend

backend-db-net
   |
db
```

The backend needed to be connected to both networks.

I therefore modified `docker-compose.yml` and added `backend-db-net` to the backend service:

```yaml
backend:
  ...
  networks:
    - nginx-backend-net
    - backend-db-net
```

I then validated the updated configuration:

```bash
docker-compose config
```

The output confirmed that the backend was connected to both networks.

After recreating the containers, the backend was able to communicate with the database.

---

# Problem 5: Nginx / Backend Connection

After fixing the Docker network configuration, I tested the application again. However, I still received:

```text
HTTP/1.1 502 Bad Gateway
```

Therefore, I checked the nginx configuration:

```bash
cat ./nginx/nginx.conf
```

I found the following configuration:

```nginx
set $backend_upstream http://backend-api:8080;
```

However, the backend logs had already shown that Gunicorn was listening on:

```text
0.0.0.0:5000
```

Also, the Docker Compose service was named:

```text
backend
```

not:

```text
backend-api
```

Therefore, nginx was trying to connect to the wrong hostname and port:

```text
backend-api:8080    ❌
```

while the actual backend endpoint was:

```text
backend:5000        ✅
```

Before changing the nginx configuration, I verified that nginx could resolve the `backend` hostname through Docker's internal DNS:

```bash
docker-compose exec nginx getent hosts backend
```

It returned:

```text
172.19.0.2 backend backend
```

This confirmed that Docker DNS was working correctly and that nginx could resolve the backend service.

I therefore changed:

```nginx
set $backend_upstream http://backend-api:8080;
```

to:

```nginx
set $backend_upstream http://backend:5000;
```

I then recreated the nginx container:

```bash
docker-compose up -d --force-recreate nginx
```

---

# Final Verification

Finally, I tested the application again:

```bash
curl -i http://localhost/graph
```

This time, the request returned:

```text
HTTP/1.1 200 OK
Content-Type: application/json
```

The expected JSON response was also returned successfully.

Therefore, the complete request flow was working correctly:

```text
Client
  |
  v
Nginx :80
  |
  v
Backend :5000
  |
  v
PostgreSQL :5432
```

### Final Result

The `/graph` endpoint was successfully restored and returned:

```text
HTTP 200 OK
```

The main troubleshooting process followed this approach throughout:

```text
Observe the symptom
        ↓
Run a targeted test
        ↓
Analyze the error/output
        ↓
Identify the root cause
        ↓
Apply the smallest necessary fix
        ↓
Verify the result
```

This approach allowed me to identify and fix the issues at the OS/DNS, Docker Compose, Docker networking, backend, and nginx configuration levels.
