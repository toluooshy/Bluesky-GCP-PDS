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
  - Memory: 1.7 GB
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
5. In Connections, add the VM from earlier with its external IP address to Authorized Networks.
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

## 5. SSH into VM and Install Websocat

Websocat is needed for consuming the ATProto firehose.

```bash
sudo wget -qO /usr/local/bin/websocat https://github.com/vi/websocat/releases/latest/download/websocat.x86_64-unknown-linux-musl
sudo chmod a+x /usr/local/bin/websocat
websocat --version
```

Test connection:

```bash
websocat "wss://jetstream2.us-east.bsky.network/subscribe?wantedCollections=app.bsky.feed.post" > output.json
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
pip install --break-system-packages websockets aiohttp asyncio asyncpg python-dotenv fastapi uvicorn sentence-transformers
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
import aiohttp
import os
from datetime import datetime
from dotenv import load_dotenv
from sentence_transformers import SentenceTransformer
import logging

# ----------------------------------------------------------
# Logging setup
# ----------------------------------------------------------
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(message)s",
)
logger = logging.getLogger(__name__)

# ----------------------------------------------------------
# Environment setup
# ----------------------------------------------------------
load_dotenv()

DB_HOST = os.getenv("DB_HOST")
DB_PORT = int(os.getenv("DB_PORT", 5432))
DB_NAME = os.getenv("DB_NAME")
DB_USER = os.getenv("DB_USER")
DB_PASSWORD = os.getenv("DB_PASSWORD")

# Load embedding model
model = SentenceTransformer("all-MiniLM-L6-v2")

FIREHOSE_URL = "wss://jetstream2.us-east.bsky.network/subscribe?wantedCollections=app.bsky.feed.post"

# ----------------------------------------------------------
# SQL Definitions
# ----------------------------------------------------------
CREATE_POSTS_TABLE_SQL = """
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

CREATE_AUTHORS_TABLE_SQL = """
CREATE TABLE IF NOT EXISTS authors (
    id TEXT PRIMARY KEY,
    handle TEXT,
    display_name TEXT,
    description TEXT,
    posts_text TEXT,
    display_name_embedding VECTOR(384),
    handle_embedding VECTOR(384),
    description_embedding VECTOR(384),
    posts_embedding VECTOR(384),
    followers_count INTEGER DEFAULT 0,
    follows_count INTEGER DEFAULT 0,
    posts_count INTEGER DEFAULT 0,
    updated_at TIMESTAMP
);
"""

INSERT_POST_SQL = """
INSERT INTO posts (repo, rkey, cid, text, created_at, embedding, raw)
VALUES ($1, $2, $3, $4, $5, $6, $7);
"""

UPSERT_AUTHOR_SQL = """
INSERT INTO authors (
    id, handle, display_name, description, posts_text,
    display_name_embedding, handle_embedding, description_embedding, posts_embedding,
    followers_count, follows_count, posts_count, updated_at
)
VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13)
ON CONFLICT (id) DO UPDATE
SET
    handle = EXCLUDED.handle,
    display_name = EXCLUDED.display_name,
    description = EXCLUDED.description,
    posts_text = LEFT(EXCLUDED.posts_text || authors.posts_text, 500),
    display_name_embedding = EXCLUDED.display_name_embedding,
    handle_embedding = EXCLUDED.handle_embedding,
    description_embedding = EXCLUDED.description_embedding,
    posts_embedding = EXCLUDED.posts_embedding,
    followers_count = EXCLUDED.followers_count,
    follows_count = EXCLUDED.follows_count,
    posts_count = EXCLUDED.posts_count,
    updated_at = GREATEST(EXCLUDED.updated_at, authors.updated_at);
"""

# ----------------------------------------------------------
# Database initialization
# ----------------------------------------------------------
async def init_db():
    """Connects to DB and ensures tables exist."""
    conn = await asyncpg.connect(
        host=DB_HOST,
        port=DB_PORT,
        user=DB_USER,
        password=DB_PASSWORD,
        database=DB_NAME,
        ssl="require"
    )
    await conn.execute("CREATE EXTENSION IF NOT EXISTS vector;")
    await conn.execute(CREATE_POSTS_TABLE_SQL)
    await conn.execute(CREATE_AUTHORS_TABLE_SQL)
    await conn.close()
    logger.info("Database initialized and tables ensured.")

# ----------------------------------------------------------
# Helper functions
# ----------------------------------------------------------
def extract_text(record):
    """Extract post text + alt text from embedded images."""
    text = record.get("text", "")
    alt_texts = []
    embed = record.get("embed", {})
    if embed.get("$type", "").startswith("app.bsky.embed.images"):
        for img in embed.get("images", []):
            alt = img.get("alt")
            if alt:
                alt_texts.append(alt)
    combined_text = text + " " + " ".join(alt_texts)
    return combined_text.strip()


async def fetch_profile(session, did):
    """Fetch profile info for a DID from Bluesky API."""
    url = f"https://public.api.bsky.app/xrpc/app.bsky.actor.getProfile?actor={did}"
    try:
        async with session.get(url, timeout=10) as resp:
            if resp.status == 200:
                data = await resp.json()
                return {
                    "handle": data.get("handle"),
                    "display_name": data.get("displayName", ""),
                    "description": data.get("description", ""),
                    "followers_count": data.get("followersCount", 0),
                    "follows_count": data.get("followsCount", 0),
                    "posts_count": data.get("postsCount", 0),
                }
            else:
                logger.warning(f"Failed to fetch profile for {did}: {resp.status}")
                return None
    except Exception as e:
        logger.error(f"Error fetching profile for {did}: {e}")
        return None

# ----------------------------------------------------------
# Firehose processing loop
# ----------------------------------------------------------
async def handle_firehose():
    """Listen to firehose and store posts and authors."""
    db = await asyncpg.create_pool(
        host=DB_HOST,
        port=DB_PORT,
        user=DB_USER,
        password=DB_PASSWORD,
        database=DB_NAME,
        ssl="require"
    )

    async with aiohttp.ClientSession() as session:
        while True:
            try:
                async with websockets.connect(FIREHOSE_URL, ping_interval=20, ping_timeout=10) as ws:
                    logger.info("Connected to Bluesky firehose.")

                    async for message in ws:
                        try:
                            evt = json.loads(message)
                            commit = evt.get("commit", {})
                            collection = commit.get("collection")
                            operation = commit.get("operation")

                            if collection != "app.bsky.feed.post" or operation != "create":
                                continue

                            repo = evt.get("did")
                            rkey = commit.get("rkey")
                            cid = commit.get("cid")
                            record = commit.get("record", {})
                            record_json = json.dumps(record)

                            # Combine text + alt texts
                            combined_text = extract_text(record)

                            # Parse creation time
                            created_at = None
                            created_at_str = record.get("createdAt")
                            if created_at_str:
                                dt = datetime.fromisoformat(created_at_str.replace("Z", "+00:00"))
                                created_at = dt.replace(tzinfo=None)

                            # Generate post embedding
                            post_embedding = model.encode(combined_text).tolist()
                            post_embedding_str = f"[{','.join(map(str, post_embedding))}]"

                            # Insert post
                            await db.execute(
                                INSERT_POST_SQL,
                                repo, rkey, cid, combined_text, created_at, post_embedding_str, record_json
                            )
                            logger.info(f"Inserted post from {repo}")

                            # Check if author exists
                            existing_author = await db.fetchrow("SELECT id FROM authors WHERE id = $1", repo)
                            if not existing_author:
                                profile = await fetch_profile(session, repo) or {}
                                handle = profile.get("handle", repo)
                                display_name = profile.get("display_name", "")
                                description = profile.get("description", "")
                                followers_count = profile.get("followers_count", 0)
                                follows_count = profile.get("follows_count", 0)
                                posts_count = profile.get("posts_count", 0)

                                posts_text = combined_text[:500]
                                updated_at = created_at

                                # Embeddings
                                display_name_emb = f"[{','.join(map(str, model.encode(display_name).tolist()))}]"
                                handle_emb = f"[{','.join(map(str, model.encode(handle).tolist()))}]"
                                desc_emb = f"[{','.join(map(str, model.encode(description).tolist()))}]"
                                posts_emb = f"[{','.join(map(str, model.encode(posts_text).tolist()))}]"

                                await db.execute(
                                    UPSERT_AUTHOR_SQL,
                                    repo, handle, display_name, description, posts_text,
                                    display_name_emb, handle_emb, desc_emb, posts_emb,
                                    followers_count, follows_count, posts_count, updated_at
                                )
                                logger.info(f"Inserted new author {repo} ({handle}) with {followers_count} followers")
                            else:
                                # Update existing author’s recent posts
                                posts_text = combined_text[:500]
                                posts_emb = f"[{','.join(map(str, model.encode(posts_text).tolist()))}]"
                                await db.execute("""
                                    UPDATE authors
                                    SET posts_text = LEFT($1 || posts_text, 500),
                                        posts_embedding = $2,
                                        updated_at = GREATEST($3, updated_at)
                                    WHERE id = $4
                                """, posts_text, posts_emb, created_at, repo)
                                logger.info(f"Updated author {repo}")

                        except Exception as e:
                            logger.error(f"Error processing message: {e}", exc_info=True)

            except websockets.ConnectionClosedError as e:
                logger.warning(f"WebSocket closed: {e}. Reconnecting in 5s...")
                await asyncio.sleep(5)
            except Exception as e:
                logger.error(f"WebSocket error: {e}", exc_info=True)
                await asyncio.sleep(5)

# ----------------------------------------------------------
# Entrypoint
# ----------------------------------------------------------
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

DATABASE_URL = f"postgresql://{DB_USER}:{DB_PASSWORD}@{DB_HOST}:{DB_PORT}/{DB_NAME}"

@asynccontextmanager
async def lifespan(app: FastAPI):
    logger.info("Starting up: creating DB connection pool...")
    app.state.pool = await create_pool(dsn=DATABASE_URL)
    yield
    logger.info("Shutting down: closing DB connection pool...")
    await app.state.pool.close()

app = FastAPI(lifespan=lifespan)

# Enable CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# --- TEXT SEARCH ENDPOINTS ---

@app.get("/search/posts")
async def search_posts(q: str = Query(...)):
    """
    Search for posts by text (ILIKE).
    """
    logger.info(f"Received post search query: {q}")
    async with app.state.pool.acquire() as conn:
        rows = await conn.fetch(
            """
            SELECT * FROM posts
            WHERE text ILIKE $1
            ORDER BY created_at DESC
            LIMIT 50
            """,
            f"%{q}%",
        )
    return [dict(row) for row in rows]


@app.get("/search/authors")
async def search_authors(q: str = Query(...), use_embedding: bool = Query(False)):
    """
    Search for authors by display_name, handle, description, or posts_text.
    Ranking is primarily by fame (followers_count + posts_count).
    Optional: use_embedding=True will rank by embedding similarity first.
    """
    logger.info(f"Received author search query: {q} (use_embedding={use_embedding})")
    async with app.state.pool.acquire() as conn:
        if use_embedding:
            # Use embedding similarity if requested
            rows = await conn.fetch(
                """
                SELECT *,
                       1 - (posts_embedding <=> $1) AS similarity,
                       (followers_count + posts_count) AS fame_score
                FROM authors
                WHERE posts_embedding IS NOT NULL
                ORDER BY similarity DESC, fame_score DESC, updated_at DESC
                LIMIT 50
                """,
                f"[{','.join(map(str, q))}]" if isinstance(q, list) else f"%{q}%",
            )
        else:
            # Text-based search
            rows = await conn.fetch(
                """
                SELECT *,
                       (followers_count + posts_count) AS fame_score
                FROM authors
                WHERE
                    display_name ILIKE $1
                    OR handle ILIKE $1
                    OR description ILIKE $1
                    OR posts_text ILIKE $1
                ORDER BY fame_score DESC, updated_at DESC
                LIMIT 50
                """,
                f"%{q}%",
            )
    return [dict(row) for row in rows]

# --- VECTOR SIMILARITY SEARCH ENDPOINTS ---

@app.post("/vector/search/posts")
async def vector_search_posts(vector: list[float]):
    """
    Find posts whose embeddings are most similar to the provided 384-dim vector.
    """
    if len(vector) != 384:
        return {"error": "Vector must be 384-dimensional."}

    vector_str = f"[{','.join(map(str, vector))}]"

    async with app.state.pool.acquire() as conn:
        rows = await conn.fetch(
            """
            SELECT *,
                   1 - (embedding <=> $1) AS similarity
            FROM posts
            WHERE embedding IS NOT NULL
            ORDER BY embedding <=> $1
            LIMIT 25
            """,
            vector_str,
        )

    return [dict(row) for row in rows]


@app.post("/vector/search/authors")
async def vector_search_authors(vector: list[float]):
    """
    Find authors whose posts_embedding are most similar to the provided 384-dim vector.
    """
    if len(vector) != 384:
        return {"error": "Vector must be 384-dimensional."}

    vector_str = f"[{','.join(map(str, vector))}]"

    async with app.state.pool.acquire() as conn:
        rows = await conn.fetch(
            """
            SELECT *,
                   1 - (posts_embedding <=> $1) AS similarity
            FROM authors
            WHERE posts_embedding IS NOT NULL
            ORDER BY posts_embedding <=> $1
            LIMIT 25
            """,
            vector_str,
        )

    return [dict(row) for row in rows]

# --- ROOT ENDPOINT ---

@app.get("/")
async def root():
    return {
        "message": "Cosine API online",
        "endpoints": [
            "/search/posts",
            "/search/authors",
            "/vector/search/posts",
            "/vector/search/authors"
        ]
    }


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
