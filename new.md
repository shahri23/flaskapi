Sure! Below is the updated Python script that connects to both PostgreSQL and Redis containers running in the same pod and retrieves the desired information. The script collects information about the PostgreSQL database name, version, and uptime, as well as the Redis server version and uptime. It then stores all this information into a single JSON file named `database_info.json` in the `/tmp/` directory.

Make sure you have the required libraries installed by running:

```bash
pip install pymongo psycopg2-binary redis
```

Here's the updated Python script:

```python
import os
import json
import pymongo
from pymongo import MongoClient
import psycopg2
import redis
from datetime import datetime, timezone

# MongoDB Configurations
mongo_username = 'your_mongo_username'
mongo_password = 'your_mongo_password'
mongo_port = 27017
mongo_connection_string = f"mongodb://{mongo_username}:{mongo_password}@localhost:{mongo_port}/"

# PostgreSQL Configurations
pg_username = 'your_pg_username'
pg_password = 'your_pg_password'
pg_port = 5432
pg_connection_string = f"postgresql://{pg_username}:{pg_password}@localhost:{pg_port}/"

# Redis Configurations
redis_port = 6379
redis_connection = redis.StrictRedis(host='localhost', port=redis_port, decode_responses=True)

# Collect MongoDB information
mongo_client = MongoClient(mongo_connection_string)
mongo_server_info = mongo_client.server_info()
mongo_info = {
    "database_name": mongo_server_info["db"],
    "database_version": mongo_server_info["version"],
    "is_master": mongo_server_info["ismaster"],
    "time_since_up": str(datetime.now(timezone.utc) - mongo_server_info["uptimeEstimate"]),
}
mongo_client.close()

# Collect PostgreSQL information
pg_conn = psycopg2.connect(pg_connection_string)
pg_cursor = pg_conn.cursor()
pg_cursor.execute("SELECT current_database(), version();")
pg_database_name, pg_version = pg_cursor.fetchone()
pg_cursor.close()
pg_conn.close()

pg_info = {
    "database_name": pg_database_name,
    "database_version": pg_version,
    "time_since_up": str(datetime.now(timezone.utc) - datetime.fromtimestamp(redis_connection.info()['uptime_in_seconds'], timezone.utc))
}

# Collect Redis information
redis_info = {
    "redis_version": redis_connection.info()['redis_version'],
    "time_since_up": str(datetime.now(timezone.utc) - datetime.fromtimestamp(redis_connection.info()['uptime_in_seconds'], timezone.utc))
}

# Save the collected information to a JSON file
output_file_path = "/tmp/database_info.json"
database_info = {
    "mongodb": mongo_info,
    "postgresql": pg_info,
    "redis": redis_info
}

with open(output_file_path, "w") as output_file:
    json.dump(database_info, output_file, indent=4)

print("Database information has been collected and saved to:", output_file_path)
```

Save this script as, for example, `database_info.py`, and run it in the sidecar container within the same pod as the MongoDB, PostgreSQL, and Redis containers. The script will connect to each database, collect the desired information, and store it as a JSON file named `database_info.json` in the `/tmp/` directory.

Make sure to replace `your_mongo_username`, `your_mongo_password`, `your_pg_username`, and `your_pg_password` with the actual credentials for MongoDB and PostgreSQL. Additionally, ensure that the sidecar container has access to the `/tmp/` directory for saving the JSON file. Adjust the script accordingly if your databases are running on different hosts or ports.
