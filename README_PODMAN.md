# Running DayTrader10 on PostgreSQL with Podman and load testing it with JMeter

This guide runs PostgreSQL and the DayTrader10 Open Liberty application as two Podman containers, then drives load against the application from the host with Apache JMeter.

```
 host                                  podman machine (VM)
+-----------------+      :9080      +------------------------------------+
| JMeter (CLI)    | --------------> | daytrader-app  (Open Liberty)      |
+-----------------+                 |        |  JDBC, network daytrader-net
                                    |        v                           |
                    :5432 (optional)| daytrader-pg   (PostgreSQL 16)     |
                                    +------------------------------------+
```

All commands below were run and verified on macOS (arm64) with Podman 6.1, Java 21, Maven 3.x and JMeter 5.6.3.

## 1. Prerequisites

| Tool | Notes |
|------|-------|
| Podman | `brew install podman`, then `podman machine init` once |
| JDK 17+ and Maven | only needed on the host to build the WAR |
| Apache JMeter 5.6.x | `brew install jmeter` |
| JMeter WebSocket sampler plugin | `jmeter-websocket-samplers` jar (see section 7) |

Start the Podman VM:

```bash
podman machine start        # "already running" is fine
podman info > /dev/null     # sanity check
```

The default VM (7 CPUs / 8 GiB) was sufficient for the load test below.

## 2. Build the application

```bash
mvn clean package -DskipTests
```

The container image needs these two build outputs:

* `target/io.openliberty.sample.daytrader10.war`
* `target/liberty/wlp/usr/shared/resources/PostgresLibs/postgresql-42.7.3.jar`  (copied there by the `copy-postgresql-dependency` execution in `pom.xml`)

## 3. Build the application image

[`Dockerfile.postgresql`](Dockerfile.postgresql) uses `icr.io/appcafe/open-liberty:kernel-slim-java17-openj9-ubi`. It copies in:

* [`server.xml_postgresql`](src/main/liberty/config/server.xml_postgresql) as `/config/server.xml`
* the WAR
* the PostgreSQL JDBC driver

It then runs `features.sh` to install the Liberty features listed in `server.xml`, and `configure.sh` to warm up the class cache.

```bash
podman build -f Dockerfile.postgresql -t daytrader10-pg .
```

`server.env` is intentionally **not** copied into the image. It would override the `MAX_USERS` / `MAX_QUOTES` values you pass with `-e` at run time.

## 4. Start PostgreSQL

```bash
podman network create daytrader-net

podman run -d --name daytrader-pg --network daytrader-net -p 5432:5432 \
  -e POSTGRES_DB=tradedb \
  -e POSTGRES_USER=db_username \
  -e POSTGRES_PASSWORD=db_password \
  postgres:16 \
  -c max_prepared_transactions=100 \
  -c max_connections=300

podman exec daytrader-pg pg_isready -U db_username -d tradedb
```

* `max_prepared_transactions=100` is **required**. In the default EJB3 runtime mode the app uses XA (two-phase commit) transactions. With PostgreSQL's default of `0` they fail with *"prepared transactions are disabled"*.
* `max_connections=300` leaves room for the app's 100-connection pool plus admin sessions.
* `-p 5432:5432` is optional. It only lets you run `psql` from the host.

## 5. Start the application

```bash
podman run -d --name daytrader-app --network daytrader-net -p 9080:9080 \
  -e tradeDbHost=daytrader-pg \
  -e tradeDbPort=5432 \
  -e tradeDbName=tradedb \
  -e dbUser=db_username \
  -e dbPass=db_password \
  -e MAX_QUOTES=10000 \
  -e MAX_USERS=15000 \
  localhost/daytrader10-pg
```

| Variable | Purpose | Default in `server.xml_postgresql` |
|----------|---------|-----------------------------------|
| `tradeDbHost` | PostgreSQL host. Use the container name on the shared network | `localhost` |
| `tradeDbPort` | PostgreSQL port | `5432` |
| `tradeDbName` | Database name | `tradedb` |
| `dbUser` / `dbPass` | Database credentials | `db_username` / `db_password` |
| `MAX_QUOTES` / `MAX_USERS` | Number of stocks and users created by `buildDB` | `1000` / `500` (from `daytrader.properties`) |
| `RUNTIME_MODE` | 0 = EJB3 (default), 1 = Direct JDBC, 2 = Session to Direct | `0` |
| `ORDER_PROCESSING_MODE` | Order processing mode | `0` (sync) |

Wait for startup (about 2 seconds after the container starts) and check the settings the app picked up:

```bash
podman logs daytrader-app 2>&1 | grep -E "MAX_|ready to run"
# MAX_USERS = 15000 users
# MAX_QUOTES = 10000 quotes
# CWWKF0011I: The defaultServer server is ready to run a smarter planet.
```

If `MAX_USERS` shows `500`, the environment variables were not applied. Check that you rebuilt the image after removing `server.env`.

## 6. Create and populate the database

The tables are not created automatically. Initialise them through the application, in this order:

```bash
# 6.1 Create the tables (uses src/main/webapp/dbscripts/postgresql/Table.ddl)
curl -s -o /dev/null -w "%{http_code}\n" "http://localhost:9080/daytrader/config?action=buildDBTables"   # 200

# 6.2 Restart the app so it picks up the new schema
podman restart daytrader-app

# 6.3 Populate quotes, users, accounts and holdings
curl -s -o /dev/null -m 600 "http://localhost:9080/daytrader/config?action=buildDB"
```

With 15000 users and 10000 quotes, step 6.3 takes roughly 2 to 3 minutes. The HTTP request returns only when it is finished. Verify the row counts:

```bash
podman exec daytrader-pg psql -U db_username tradedb -tc \
  "select (select count(*) from accountprofileejb), (select count(*) from quoteejb), (select count(*) from holdingejb)"
#  15000 | 10000 | ~75000
```

Do not call `?action=resetTrade` with *delete all*. It wipes the data.

## 7. Run JMeter from the host

### 7.1 One-time JMeter setup

The test plan uses a WebSocket sampler, which is not bundled with JMeter:

```bash
curl -L -o /opt/homebrew/Cellar/jmeter/5.6.3/libexec/lib/ext/wss.jar \
  https://repo1.maven.org/maven2/net/luminis/jmeter/jmeter-websocket-samplers/1.3.2/jmeter-websocket-samplers-1.3.2.jar
```

### 7.2 Run the test

```bash
mkdir -p /tmp/jm
jmeter -n -t jmeter_files/daytrader10.jmx \
  -JHOST=localhost -JPORT=9080 -JDURATION=60 \
  -Jjmeter.save.saveservice.response_data=false \
  -Jjmeter.save.saveservice.samplerData=false \
  -l /tmp/jm/pod.jtl -j /tmp/jm/pod.log
```

The two `-Jjmeter.save.saveservice.*=false` options keep response bodies out of the results file. Without them a 90 second run produced a 505 MB `.jtl` and exhausted memory.

Test plan parameters (`-J<name>=<value>`):

| Parameter | Default | Meaning |
|-----------|---------|---------|
| `HOST`, `PORT` | `localhost`, `9080` | Target host and port. Use `localhost` / `9080` for the container |
| `DURATION` | 180 | Test length in seconds |
| `THREADS` | 50 | Concurrent virtual users |
| `RAMP` | 0 | Ramp-up time |
| `MAXTHINKTIME` | 0 | Maximum think time between requests |
| `STOCKS` | 9999 | Highest stock index used (`s:0` to `s:STOCKS`) |
| `BOTUID` / `TOPUID` | 0 / 14999 | User id range (`uid:BOTUID` to `uid:TOPUID`) |

**The user and stock ranges must exist in the database.** The defaults assume 10000 quotes and 15000 users, which is why section 5 sets `MAX_QUOTES=10000` and `MAX_USERS=15000`. If you populate a smaller database, lower `-JSTOCKS`, `-JBOTUID` and `-JTOPUID` to match. Otherwise many requests fail. A run with the defaults against a 500-user database gave about 33% errors.

### 7.3 Check the results

JMeter prints periodic `summary` lines to the console. To summarise the `.jtl` afterwards:

```bash
awk -F, 'NR>1{n++; if($8!="true")e++} END{print n" samples, "e+0" errors"}' /tmp/jm/pod.jtl
```

Verified result (50 threads, defaults, 15000 users / 10000 quotes, EJB3 mode, 60 s):

```
summary = 362033 in 00:01:00 = 6026.5/s Avg: 8 Min: 0 Max: 377 Err: 0 (0.00%)
```

Throughput depends on the machine and the Podman VM resources.

## 8. Troubleshooting

| Symptom | Cause and fix |
|---------|---------------|
| `curl` returns `000` / "Empty reply" and the log shows `CWWKG0075E ... ${default.http.port} is not valid` | The HTTP port variable is not defined. `server.xml_postgresql` now defines defaults for `default.http.port` and `default.https.port`. Rebuild the image |
| `MAX_USERS = 500` in the log although `-e MAX_USERS=15000` was passed | An old image with `server.env` baked in. Rebuild |
| *"prepared transactions are disabled"* | PostgreSQL was started without `-c max_prepared_transactions=100` |
| Row counts stop increasing during `buildDB` | The populate is stuck on a database lock. Check with `podman exec daytrader-pg psql -U db_username tradedb -c "select pid, state, pg_blocking_pids(pid), left(query,60) from pg_stat_activity where datname='tradedb' and state<>'idle'"`. If one session is `idle in transaction` and others are blocked by it, run `podman restart daytrader-app`, call `buildDB` once and wait for it to finish. Do not start a second `buildDB` while one is running: each call first deletes all existing data, so a repeated call collides with the first |
| `buildDB` ends quickly and JMeter shows many HTTP errors | `MAX_USERS` / `MAX_QUOTES` are lower than the JMeter ranges. Align them or lower `-JBOTUID`, `-JTOPUID` and `-JSTOCKS` |
| `jmeter` output is only `WARNING: package sun.awt.X11 ...` and no `.log` is created | JMeter was killed, for example because it was run as a detached background job from a tool shell. Run it in a normal terminal |
| `WLTC0020E` warnings in the Liberty log under load | Known and not PostgreSQL-specific. It also occurs with Derby (async CDI events outside a transaction context) |
| HTTP 500 on `Sell` for a holding that was already sold | Pre-existing application behaviour in `TradeSLSBBean.sell()`. It also occurs with Derby |

Useful commands:

```bash
podman logs -f daytrader-app                       # follow application log
podman exec -it daytrader-pg psql -U db_username tradedb
podman ps
```

## 9. Clean up

```bash
podman rm -f daytrader-app daytrader-pg
podman network rm daytrader-net
podman rmi daytrader10-pg        # optional
```

The PostgreSQL data lives inside the container, so removing it deletes the database. To keep the data between runs, add `-v daytrader-pgdata:/var/lib/postgresql/data` to the `podman run` command in section 4.
