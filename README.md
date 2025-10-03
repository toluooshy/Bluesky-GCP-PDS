# Self-Hosted ATProto PDS with Firehose Ingestion and Search API

This guide walks you through setting up a self-hosted Bluesky PDS that ingests data from the ATProto firehose and exposes a FastAPI-powered search API. It uses Google Cloud Platform (GCP), Python, and Caddy for HTTPS proxying.

---

## 1. Create a GCP VM

**Settings:**

- OS: Ubuntu 24.04 LTS
- Machine Type: `e2-small`
- Storage: 40GB
- Enable: HTTP & HTTPS traffic

---

## 2. Set Up Firewall Rule

1. Go to **VPC Network > Firewall Rules**.
2. Click **Create Firewall Rule**.
3. Set:
   - **Name**: `allow-web`
   - **Targets**: Apply to specific instances (select your VM)
   - **Source IP ranges**: `0.0.0.0/0`
   - **Protocols and Ports**: `tcp:80,443,8000`

---

## 3. Assign Static External IP

1. Go to **VPC Network > External IP addresses**.
2. Reserve a static IP.
3. Assign it to your VM.

---

## 4. Create Cloud SQL (PostgreSQL 17)

**Settings:**

- Edition: Enterprise
- Region/Zone: Choose close to your VM
- Configuration:
  - Storage: 10GB SSD
  - vCPU: 1
  - Memory: 3.75 GB
  - Enable Public IP
- Add your VM’s external IP to **Authorized Networks**

---

### Create Database, User, and Password

1. In the **Cloud SQL console**, go to your new PostgreSQL instance.
2. Under **Users**, click **Create User Account**:
   - **Username**: e.g. `pds_user`
   - **Password**: choose a strong password (you’ll put this in your `.env` later)
3. Under **Databases**, click **Create Database**:
   - **Database name**: e.g. `pds_db`
   - Leave other settings default.
4. Your instance now has:
   - Host/IP: `<your-instance-public-ip>`
   - Database: `pds_db`
   - User: `pds_user`
   - Password: `<your-password>`

---

### Test Connection from VM

SSH into your VM and install `psql`:

```bash
sudo apt update
sudo apt install -y postgresql-client
```

Run:

```bash
psql "host=<your-instance-public-ip> dbname=pds_db user=pds_user password=<your-password> sslmode=require"
```

If successful, you’ll see the PostgreSQL prompt:

```
pds_db=>
```

Type `\q` to quit.

```

---

Do you want me to also **update the `.env` example** in your guide to include the new `pds_user` and `pds_db` placeholders so it’s consistent?
```

## 5. SSH into VM and Install Websocat

Websocat is needed for consuming the ATProto firehose.

```bash
sudo wget -qO /usr/local/bin/websocat https://github.com/vi/websocat/releases/latest/download/websocat.x86_64-unknown-linux-musl
sudo chmod a+x /usr/local/bin/websocat
websocat --version
```

Test connection:

```bash
websocat "wss://jetstream.bsky.network/subscribe?wantedCollections=app.bsky.feed.post" > output.json
```

More on Jetstream: [https://docs.bsky.app/blog/jetstream](https://docs.bsky.app/blog/jetstream)

---

## 6. Install and Configure PDS

Follow official guide:

- [https://atproto.com/guides/self-hosting](https://atproto.com/guides/self-hosting)

---

## 7. Install Python and Dependencies

```bash
sudo apt update
sudo apt install -y python3-pip
pip install --break-system-packages websockets asyncio asyncpg python-dotenv fastapi uvicorn sentence-transformers
```

Create your `.env` file:

```env
DB_HOST=your-cloudsql-ip-or-private-ip
DB_PORT=5432
DB_NAME=yourdbname
DB_USER=yourdbuser
DB_PASSWORD=yourdbpassword
```

---

## 8. Python Scripts

Create and configure the following Python files:

- `debug.py` — for basic connection checks
- `ingest.py` — for firehose ingestion
- `prune.py` — to clean old data
- `api.py` — FastAPI-based search API

debug.py

```python
import asyncio
import websockets
import json
import logging

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(message)s",
)
logger = logging.getLogger(__name__)

# Bluesky firehose endpoint
FIREHOSE_URL = "wss://jetstream2.us-east.bsky.network/subscribe?wantedCollections=app.bsky.feed.post"

OUTPUT_FILE = "output.jsonl"  # line-delimited JSON for easier inspection

async def main():
    async with websockets.connect(FIREHOSE_URL) as ws:
        logger.info("Connected to Bluesky firehose.")

        with open(OUTPUT_FILE, "w") as outfile:
            count = 0
            async for message in ws:
                logger.debug(f"Raw message: {message}")
                try:
                    evt = json.loads(message)
                    logger.debug(f"Event keys: {list(evt.keys())}")

                    commit = evt.get("commit", {})
                    collection = commit.get("collection")
                    logger.debug(f"Collection: {collection}")

                    if collection != "app.bsky.feed.post":
                        continue

                    repo = evt.get("did")
                    rkey = commit.get("rkey")
                    cid = commit.get("cid")
                    record = commit.get("record", {})

                    text = record.get("text")
                    created_at = record.get("createdAt")

                    row = {
                        "repo": repo,
                        "rkey": rkey,
                        "cid": cid,
                        "text": text,
                        "created_at": created_at,
                        "raw": record,
                    }

                    outfile.write(json.dumps(row) + "\n")
                    outfile.flush()

                    count += 1
                    logger.info(f"Wrote post #{count} from {repo}")

                    if count >= 20:
                        logger.info("Reached 20 posts. Exiting.")
                        break

                except Exception as e:
                    logger.error(f"Error processing message: {e}")

if __name__ == "__main__":
    asyncio.run(main())
```

ingest.py

```python
import asyncio
import websockets
import json
import asyncpg
import os
from datetime import datetime
from dotenv import load_dotenv
from sentence_transformers import SentenceTransformer
import logging

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(message)s",
)
logger = logging.getLogger(__name__)

# Load environment variables from .env file
load_dotenv()

# Database configuration from environment variables
DB_HOST = os.getenv("DB_HOST")
DB_PORT = int(os.getenv("DB_PORT", 5432))
DB_NAME = os.getenv("DB_NAME")
DB_USER = os.getenv("DB_USER")
DB_PASSWORD = os.getenv("DB_PASSWORD")

# Load MiniLM v6 model (384-d)
model = SentenceTransformer("all-MiniLM-L6-v2")

# Bluesky firehose WebSocket endpoint
FIREHOSE_URL = (
    "wss://jetstream2.us-east.bsky.network/subscribe?wantedCollections=app.bsky.feed.post"
)

# SQL to create the table
CREATE_TABLE_SQL = """
CREATE TABLE IF NOT EXISTS posts (
    id SERIAL PRIMARY KEY,
    repo TEXT,
    rkey TEXT,
    cid TEXT,
    text TEXT,
    created_at TIMESTAMP,
    embedding VECTOR(384),
    raw JSONB
);
"""

# SQL to insert a post
INSERT_POST_SQL = """
INSERT INTO posts (repo, rkey, cid, text, created_at, embedding, raw)
VALUES ($1, $2, $3, $4, $5, $6, $7);
"""

async def init_db():
    """Connects to the database and ensures the posts table exists."""
    conn = await asyncpg.connect(
        host=DB_HOST,
        port=DB_PORT,
        user=DB_USER,
        password=DB_PASSWORD,
        database=DB_NAME,
        ssl="require"
    )
    # Enable the vector extension if not enabled already
    await conn.execute("CREATE EXTENSION IF NOT EXISTS vector;")
    await conn.execute(CREATE_TABLE_SQL)
    await conn.close()
    logger.info("Database initialized and table ensured.")

async def handle_firehose():
    """Connects to the Bluesky firehose and stores posts in the database."""
    db = await asyncpg.create_pool(
        host=DB_HOST,
        port=DB_PORT,
        user=DB_USER,
        password=DB_PASSWORD,
        database=DB_NAME,
        ssl="require"
    )

    while True:
        try:
            async with websockets.connect(
                FIREHOSE_URL,
                ping_interval=20,      # send ping every 20 seconds
                ping_timeout=10        # wait 10 seconds for pong before considering connection lost
            ) as ws:
                logger.info("Connected to Bluesky firehose.")

                async for message in ws:
                    logger.debug(f"Raw message: {message}")
                    try:
                        evt = json.loads(message)
                        logger.debug(f"Event keys: {list(evt.keys())}")

                        commit = evt.get("commit", {})
                        collection = commit.get("collection")
                        operation = commit.get("operation")
                        logger.debug(f"Collection: {collection} | Operation: {operation}")

                        # Only handle creates for app.bsky.feed.post
                        if collection != "app.bsky.feed.post" or operation != "create":
                            continue

                        repo = evt.get("did")
                        rkey = commit.get("rkey")
                        cid = commit.get("cid")
                        record = commit.get("record", {})
                        record_json = json.dumps(record)

                        text = record.get("text")
                        created_at_str = record.get("createdAt")

                        # Convert ISO string to naive datetime
                        created_at = None
                        if created_at_str:
                            dt = datetime.fromisoformat(created_at_str.replace("Z", "+00:00"))
                            created_at = dt.replace(tzinfo=None)

                        # Generate raw embedding
                        raw_embedding = model.encode(text).tolist()

                        # Convert raw embedding list to pgvector string format
                        embedding = f"[{','.join(map(str, raw_embedding))}]"

                        await db.execute(
                            INSERT_POST_SQL,
                            repo, rkey, cid, text, created_at, embedding, record_json
                        )

                        logger.info(f"Inserted post from {repo}")

                    except Exception as e:
                        logger.error(f"Error processing message: {e}", exc_info=True)

        except websockets.ConnectionClosedError as e:
            logger.warning(f"WebSocket connection closed with error: {e}. Reconnecting in 5 seconds...")
            await asyncio.sleep(5)  # wait before reconnecting
        except Exception as e:
            logger.error(f"Unexpected error in WebSocket connection: {e}", exc_info=True)
            logger.info("Reconnecting in 5 seconds...")
            await asyncio.sleep(5)

async def main():
    await init_db()
    await handle_firehose()

if __name__ == "__main__":
    asyncio.run(main())
```

prune.py

```python
import asyncio
import asyncpg
import os
import logging

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(message)s",
)
logger = logging.getLogger(__name__)

# Constants
TABLE_NAME = "posts"
SIZE_LIMIT_BYTES = 6 * 1024 * 1024 * 1024  # 6GB
DELETE_BATCH_SIZE = 100
PRUNE_INTERVAL_SEC = 1

# Database configuration from environment variables
DB_HOST = os.getenv("DB_HOST")
DB_PORT = int(os.getenv("DB_PORT", 5432))
DB_NAME = os.getenv("DB_NAME")
DB_USER = os.getenv("DB_USER")
DB_PASSWORD = os.getenv("DB_PASSWORD")

# Assemble the database URL
DATABASE_URL = f"postgresql://{DB_USER}:{DB_PASSWORD}@{DB_HOST}:{DB_PORT}/{DB_NAME}"

async def get_table_size(conn):
    row = await conn.fetchrow("""
        SELECT pg_total_relation_size($1) AS size
    """, TABLE_NAME)
    return row["size"]

async def prune_oldest_rows(conn):
    logger.info("Pruning oldest rows...")
    result = await conn.execute(f"""
        DELETE FROM {TABLE_NAME}
        WHERE ctid IN (
            SELECT ctid FROM {TABLE_NAME}
            ORDER BY created_at ASC
            LIMIT {DELETE_BATCH_SIZE}
        )
    """)
    logger.info(f"Deleted rows: {result}")

async def run_pruner():
    conn = await asyncpg.connect(DATABASE_URL)
    logger.info("Pruner started. Monitoring table size...")

    try:
        while True:
            size = await get_table_size(conn)
            logger.info(f"Table size: {round(size / 1024 / 1024, 2)} MB")

            if size > SIZE_LIMIT_BYTES:
                await prune_oldest_rows(conn)
            await asyncio.sleep(PRUNE_INTERVAL_SEC)
    finally:
        await conn.close()
        logger.info("Pruner stopped and connection closed.")

if __name__ == "__main__":
    asyncio.run(run_pruner())
```

api.py

```python
from fastapi import FastAPI, Query
from fastapi.middleware.cors import CORSMiddleware
from asyncpg import create_pool
import uvicorn
import os
from contextlib import asynccontextmanager
import logging

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(message)s",
)
logger = logging.getLogger(__name__)

# Database configuration from environment variables
DB_HOST = os.getenv("DB_HOST")
DB_PORT = int(os.getenv("DB_PORT", 5432))
DB_NAME = os.getenv("DB_NAME")
DB_USER = os.getenv("DB_USER")
DB_PASSWORD = os.getenv("DB_PASSWORD")

# Assemble the database URL
DATABASE_URL = f"postgresql://{DB_USER}:{DB_PASSWORD}@{DB_HOST}:{DB_PORT}/{DB_NAME}"

@asynccontextmanager
async def lifespan(app: FastAPI):
    logger.info("Starting up: creating DB connection pool...")
    app.state.pool = await create_pool(dsn=DATABASE_URL)
    yield
    logger.info("Shutting down: closing DB connection pool...")
    await app.state.pool.close()

app = FastAPI(lifespan=lifespan)

# Add CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Or specify allowed origins instead of ["*"]
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/search")
async def search_posts(q: str = Query(...)):
    logger.info(f"Received search query: {q}")
    async with app.state.pool.acquire() as conn:
        rows = await conn.fetch(
            "SELECT * FROM posts WHERE text ILIKE $1 ORDER BY created_at DESC LIMIT 50", f"%{q}%"
        )
        logger.info(f"Returned {len(rows)} results")
        return [dict(row) for row in rows]

if __name__ == "__main__":
    logger.info("Starting API server...")
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## 9. Shell Scripts to Manage Services

Create these 3 files:

- `run_ingest.sh`
- `run_prune.sh`
- `run_api.sh`

Paste the following into each, adjusting the script name:

run_ingest.sh

```shell
#!/bin/bash

SCRIPT="ingest.py"
PID_FILE="ingest.pid"
LOG_FILE="ingest.log"

set -a
source .env
set +a

start() {
  if [ -f "$PID_FILE" ] && kill -0 $(cat "$PID_FILE") 2>/dev/null; then
    echo "$SCRIPT is already running with PID $(cat "$PID_FILE")"
    exit 1
  fi

  nohup python3 "$SCRIPT" > "$LOG_FILE" 2>&1 &
  echo $! > "$PID_FILE"
  echo "Started $SCRIPT with PID $(cat "$PID_FILE")"
}

stop() {
  if [ ! -f "$PID_FILE" ]; then
    echo "No PID file found for $SCRIPT"
    exit 1
  fi

  PID=$(cat "$PID_FILE")
  if kill -0 "$PID" 2>/dev/null; then
    kill "$PID"
    echo "Stopped $SCRIPT (PID $PID)"
    rm "$PID_FILE"
  else
    echo "Process $PID not running"
    rm "$PID_FILE"
  fi
}

status() {
  if [ -f "$PID_FILE" ] && kill -0 $(cat "$PID_FILE") 2>/dev/null; then
    echo "$SCRIPT is running with PID $(cat "$PID_FILE")"
  else
    echo "$SCRIPT is not running"
  fi
}

case "$1" in
  start) start ;;
  stop) stop ;;
  status) status ;;
  *) echo "Usage: $0 {start|stop|status}" ;;
esac
```

run_prune.sh

```shell
#!/bin/bash

SCRIPT="prune.py"
PID_FILE="prune.pid"
LOG_FILE="prune.log"

set -a
source .env
set +a

start() {
  if [ -f "$PID_FILE" ] && kill -0 $(cat "$PID_FILE") 2>/dev/null; then
    echo "$SCRIPT is already running with PID $(cat "$PID_FILE")"
    exit 1
  fi

  nohup python3 "$SCRIPT" > "$LOG_FILE" 2>&1 &
  echo $! > "$PID_FILE"
  echo "Started $SCRIPT with PID $(cat "$PID_FILE")"
}

stop() {
  if [ ! -f "$PID_FILE" ]; then
    echo "No PID file found for $SCRIPT"
    exit 1
  fi

  PID=$(cat "$PID_FILE")
  if kill -0 "$PID" 2>/dev/null; then
    kill "$PID"
    echo "Stopped $SCRIPT (PID $PID)"
    rm "$PID_FILE"
  else
    echo "Process $PID not running"
    rm "$PID_FILE"
  fi
}

status() {
  if [ -f "$PID_FILE" ] && kill -0 $(cat "$PID_FILE") 2>/dev/null; then
    echo "$SCRIPT is running with PID $(cat "$PID_FILE")"
  else
    echo "$SCRIPT is not running"
  fi
}

case "$1" in
  start) start ;;
  stop) stop ;;
  status) status ;;
  *) echo "Usage: $0 {start|stop|status}" ;;
esac
```

run_api.sh

```shell
#!/bin/bash

SCRIPT="api.py"
PID_FILE="api.pid"
LOG_FILE="api.log"

set -a
source .env
set +a

start() {
  if [ -f "$PID_FILE" ] && kill -0 $(cat "$PID_FILE") 2>/dev/null; then
    echo "$SCRIPT is already running with PID $(cat "$PID_FILE")"
    exit 1
  fi

  nohup python3 "$SCRIPT" > "$LOG_FILE" 2>&1 &
  echo $! > "$PID_FILE"
  echo "Started $SCRIPT with PID $(cat "$PID_FILE")"
}

stop() {
  if [ ! -f "$PID_FILE" ]; then
    echo "No PID file found for $SCRIPT"
    exit 1
  fi

  PID=$(cat "$PID_FILE")
  if kill -0 "$PID" 2>/dev/null; then
    kill "$PID"
    echo "Stopped $SCRIPT (PID $PID)"
    rm "$PID_FILE"
  else
    echo "Process $PID not running"
    rm "$PID_FILE"
  fi
}

status() {
  if [ -f "$PID_FILE" ] && kill -0 $(cat "$PID_FILE") 2>/dev/null; then
    echo "$SCRIPT is running with PID $(cat "$PID_FILE")"
  else
    echo "$SCRIPT is not running"
  fi
}

case "$1" in
  start) start ;;
  stop) stop ;;
  status) status ;;
  *) echo "Usage: $0 {start|stop|status}" ;;
esac
```

Make them executable:

```bash
chmod +x run_*.sh
```

Run them:

```bash
./run_ingest.sh start
./run_prune.sh start
./run_api.sh start
```

---

## 10. Set Up Caddy Proxy

Your PDS likely already uses Caddy via Docker.

### Find Caddy Container:

```bash
sudo docker ps
```

Look for the container running the `caddy:2` image.

### Locate the Caddyfile:

Caddy config is mounted, often at:

```bash
/pds/caddy/etc/caddy/Caddyfile
```

Edit it:

```bash
sudo nano /pds/caddy/etc/caddy/Caddyfile
```

Add this block under your domain:

```caddy
*.yourdomain.com, yourdomain.com {
    tls {
        on_demand
    }

    # ADD THIS TO THE FILE
    route /api/* {
        uri strip_prefix /api
        reverse_proxy http://0.0.0.0:8000
    }

    reverse_proxy http://localhost:3000
}
```

Restart Caddy:

```bash
sudo docker restart <caddy-container-id>
```

---

## 11. Verify

Test search endpoint:

```
https://yourdomain.com/api/search?q=example
```

You should get results from your ingested posts.

---

## ✅ Done

You now have a working:

- ATProto PDS
- Firehose ingester
- Pruning script
- FastAPI-powered search API
- Reverse proxy via Caddy

You can now build richer services on top of your self-hosted Bluesky data.
