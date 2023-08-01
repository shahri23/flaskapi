To achieve the desired information for PostgreSQL, Redis, and MongoDB servers without knowing the specific database names, you can use the information_schema and Redis `INFO` command for PostgreSQL and Redis respectively. For MongoDB, you can use the admin database, which is always present and contains server-level information.

Here's the updated Python script:

```python
import os
import json
import pymongo
import psycopg2
import redis
from datetime import datetime, timezone

# Function to get PostgreSQL server information
def get_postgresql_info(username, password, host='localhost', port=5432):
    connection_string = f"postgresql://{username}:{password}@{host}:{port}/postgres"
    conn = psycopg2.connect(connection_string)
    cursor = conn.cursor()

    # Get PostgreSQL server version
    cursor.execute("SELECT version();")
    postgres_version = cursor.fetchone()[0]

    # Get a list of all databases and their sizes
    cursor.execute("SELECT datname, pg_database_size(oid) FROM pg_database;")
    database_list = cursor.fetchall()

    # Close the cursor and connection
    cursor.close()
    conn.close()

    return {
        "server_type": "PostgreSQL",
        "server_version": postgres_version,
        "databases": [
            {
                "database_name": db[0],
                "size_bytes": db[1],
            }
            for db in database_list
        ]
    }

# Function to get Redis server information
def get_redis_info(username, password, host='localhost', port=6379):
    redis_connection = redis.StrictRedis(host=host, port=port, password=password, decode_responses=True)

    # Get Redis server information
    redis_info = redis_connection.info()

    return {
        "server_type": "Redis",
        "server_version": redis_info['redis_version'],
        "uptime_seconds": redis_info['uptime_in_seconds'],
        "is_master": "master" if redis_info['role'] == 'master' else "slave"
    }

# Function to get MongoDB server information
def get_mongodb_info(username, password, host='localhost', port=27017):
    connection_string = f"mongodb://{username}:{password}@{host}:{port}/admin"
    client = pymongo.MongoClient(connection_string)

    # Get MongoDB server version
    mongodb_version = client.server_info()['version']

    # Get a list of all databases and their sizes
    db_list = client.list_database_names()
    database_list = [
        {
            "database_name": db,
            "size_bytes": client[db].command("dbStats")["dataSize"]
        }
        for db in db_list
    ]

    # Close the connection
    client.close()

    return {
        "server_type": "MongoDB",
        "server_version": mongodb_version,
        "databases": database_list
    }

def main():
    # Replace these with your actual credentials
    postgres_username = 'your_postgres_username'
    postgres_password = 'your_postgres_password'

    redis_username = 'your_redis_username'
    redis_password = 'your_redis_password'

    mongo_username = 'your_mongo_username'
    mongo_password = 'your_mongo_password'

    # Get server information for PostgreSQL, Redis, and MongoDB
    postgres_info = get_postgresql_info(postgres_username, postgres_password)
    redis_info = get_redis_info(redis_username, redis_password)
    mongo_info = get_mongodb_info(mongo_username, mongo_password)

    # Combine all the server information
    server_info = {
        "PostgreSQL": postgres_info,
        "Redis": redis_info,
        "MongoDB": mongo_info
    }

    # Save the collected information to a JSON file
    output_file_path = "/tmp/result.json"
    with open(output_file_path, "w") as output_file:
        json.dump(server_info, output_file, indent=4)

    print("Server information has been collected and saved to:", output_file_path)

if __name__ == "__main__":
    main()
```

In this updated script, we have defined three functions to get server information for PostgreSQL, Redis, and MongoDB. Each function connects to the respective server, retrieves the required information, and returns it in a dictionary format.

The `main()` function then calls these functions with the provided credentials and combines all the collected information into a single dictionary called `server_info`.

The script then saves the `server_info` dictionary as a JSON file named `result.json` in the `/tmp/` directory.

Please make sure to replace the placeholders `your_postgres_username`, `your_postgres_password`, `your_redis_username`, `your_redis_password`, `your_mongo_username`, and `your_mongo_password` with your actual credentials.

When you run this script, it will gather the server information for PostgreSQL, Redis, and MongoDB and save it in a single JSON file called `result.json` in the `/tmp/` directory.
