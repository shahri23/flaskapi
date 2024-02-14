1. Workflow to Generate and Commit JSON
   This workflow takes a variable as input, generates a JSON file based on that input, and commits the JSON file back to the repository in a specified subfolder.

```yaml
name: Fetch Dashboard and Store

on:
  workflow_dispatch:
    inputs:
      dashboard_uid:
        description: "Dashboard UID"
        required: true

jobs:
  fetch_dashboard_and_store:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Fetch Dashboard
        run: |
          curl -o dev.json "https://your-grafana-url.com/api/dashboards/uid/${{ github.event.inputs.dashboard_uid }}"
        env:
          GRAFANA_API_TOKEN: ${{ secrets.GRAFANA_API_TOKEN }}

      - name: Create Branch and Push
        run: |
          git checkout -b dev
          mkdir -p dev
          mv dev.json dev/
          git add .
          git commit -m "Add dev.json dashboard"
          git push origin dev
```

2. Workflow Triggered by PR to Main
   This workflow runs whenever a pull request is made to the main branch. It will use the generated JSON file from the previous workflow.

```yaml
name: Update Stage Dashboard

on:
  pull_request_review:
    types: [submitted]
    branches:
      - main
      - "stage/*"

jobs:
  update_stage_dashboard:
    if: github.event.review.state == 'approved'
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Fetch Dev Dashboard
        run: |
          git fetch origin dev
          git checkout FETCH_HEAD -- dev/dev.json

      - name: Update Stage Dashboard
        run: |
          # Your script to update the stage dashboard using dev.json and Grafana API
```

3. Workflow Triggered by PR to Release Tag
   This workflow runs whenever a pull request is made with a tag that starts with release/. It also utilizes the JSON file generated earlier.

```yaml
name: Update Release Dashboard

on:
  pull_request_review:
    types: [submitted]
    branches:
      - main
      - "release/*"

jobs:
  update_release_dashboard:
    if: github.event.review.state == 'approved'
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Fetch Dev Dashboard
        run: |
          git fetch origin dev
          git checkout FETCH_HEAD -- dev/dev.json

      - name: Update Release Dashboard
        run: |
          # Your script to update the release dashboard using dev.json and Grafana API
```

# a gha flow to pull and store data in repo

```python
import json

# Read JSON data from file
with open('mydata.json', 'r') as file:
    data = json.load(file)

# Modify entries occurring before the closing curly brace
data["dashboard"]["title"] = "New Title"
data["folderUid"] = "new-folder-uid"

# Write modified data back to the same file
with open('mydata.json', 'w') as file:
    json.dump(data, file, indent=2)

print("JSON data updated in place.")

```

```sh
#!/bin/bash
# Modify entries occurring before the closing curly brace using jq
modified_json_object=$(jq '.dashboard.title = "New Title" | .folderUid = "new-folder-uid"' mydata.json)
echo "$modified_json_object" > mydata.json
```

```yaml
name: Fetch and Store JSON

on:
  push:
    branches:
      - main

jobs:
  fetch-json:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Perform GET request
        run: |
          curl -o result.json https://your-api-endpoint.com/your/endpoint

      - name: Create directory if not exists
        run: mkdir -p json_data

      - name: Move JSON file to subfolder
        run: mv result.json json_data/

      - name: Commit and push changes
        run: |
          git config --global user.name 'GitHub Actions'
          git config --global user.email 'actions@users.noreply.github.com'
          git add json_data/result.json
          git commit -m "Update JSON data"
          git push
```

# flaskapi

Storing the secret key itself in HashiCorp Vault is a good practice for secure secret management. To achieve this, you'll need to write the JWT secret key to HashiCorp Vault during the setup or initialization phase and then retrieve it during authentication. Here's an updated version of the Flask application that stores the JWT secret key in HashiCorp Vault:

1. Install required packages:
   Ensure you have the necessary Flask, Vault client, and cryptography libraries installed. You can install them using `pip`:

```bash
pip install Flask Flask-RESTful Flask-JWT-Extended hvac cryptography
```

2. Set up HashiCorp Vault:
   Make sure you have HashiCorp Vault installed and running. Configure it with an appropriate authentication mechanism (e.g., token-based or AppRole) and create a secret backend where you want to store your JWT secret key.

3. Modify the Flask application:

```python
from flask import Flask, request
from flask_restful import Resource, Api
from flask_jwt_extended import JWTManager, jwt_required, create_access_token, get_jwt_identity
import hvac
from cryptography.fernet import Fernet

app = Flask(__name__)
app.config['SECRET_KEY'] = None
api = Api(app)
jwt = JWTManager(app)

# Connect to HashiCorp Vault
client = hvac.Client(url='http://your-vault-address:8200', token='your-vault-token')
VAULT_SECRET_PATH = 'secret/flask_app'  # Replace with your desired Vault secret path


# Helper function to generate and write JWT secret key to HashiCorp Vault
def generate_and_store_jwt_secret_key():
    key = Fernet.generate_key()
    secret_data = {'jwt_secret_key': key.decode()}
    client.write(VAULT_SECRET_PATH, **secret_data)


# Helper function to retrieve JWT secret key from HashiCorp Vault
def get_jwt_secret_key():
    secret_response = client.read(VAULT_SECRET_PATH)
    if secret_response is not None and 'data' in secret_response:
        return secret_response['data'].get('jwt_secret_key', None)
    return None


# Check if the JWT secret key exists in Vault; if not, generate and store it.
if get_jwt_secret_key() is None:
    generate_and_store_jwt_secret_key()


# Mocked user data. Replace this with your database or user management system.
users = {
    'user1': {'password': 'password123'},
    'user2': {'password': 'password456'}
}


# Authentication endpoint to get JWT token
class LoginResource(Resource):
    def post(self):
        data = request.get_json()
        username = data.get('username', None)
        password = data.get('password', None)

        if not username or not password:
            return {'message': 'Missing credentials'}, 400

        if users.get(username) and users[username]['password'] == password:
            jwt_secret_key = get_jwt_secret_key()
            if jwt_secret_key:
                app.config['SECRET_KEY'] = jwt_secret_key.encode()  # Set the JWT secret key from Vault
                access_token = create_access_token(identity=username)
                return {'access_token': access_token}, 200
            else:
                return {'message': 'JWT secret key not found in Vault'}, 500
        else:
            return {'message': 'Invalid credentials'}, 401


# Protected endpoint that requires authentication
class DataResource(Resource):
    @jwt_required()
    def get(self):
        current_user = get_jwt_identity()
        # Your data retrieval logic here.
        return {'message': 'Hello, {}! This is protected data.'.format(current_user)}, 200


api.add_resource(LoginResource, '/login')
api.add_resource(DataResource, '/api/data')


if __name__ == '__main__':
    app.run(debug=True)
```

With this updated version, the Flask application will generate a new JWT secret key if it does not already exist in HashiCorp Vault. If the secret key already exists, it will be retrieved and used for JWT generation during authentication. This approach ensures that the JWT secret key is securely managed in HashiCorp Vault and not exposed in the application code.
