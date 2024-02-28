## Release Tag

If you're not using a release branch and only using release tags directly from the main branch, the process simplifies a bit. Here's how you can modify the process:

```yaml
name: Merge to Main Action
on:
  pull_request:
    types: [closed]
jobs:
  check_merge:
    if: github.event.pull_request.merged == true && github.event.pull_request.base.ref == 'main' && contains(github.event.pull_request.head.repo.full_name, github.repository)
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: List changed files
        id: list_files
        run: echo "::set-output name=files::$(git diff --name-only ${{ github.event.pull_request.base.sha }} ${{ github.event.pull_request.head.sha }})"

      - name: Echo files if merged into cdp-release folder
        run: |
          files="${{ steps.list_files.outputs.files }}"
          for file in $files; do
            if [[ $file == "cdp-release/"* ]]; then
              echo "File '$file' was merged into the cdp-release folder."
            fi
          done
```

Ensure your local main branch is up-to-date:

```bash
git checkout main
git pull origin main
```

Prepare your code for release:

Make any final adjustments, updates, or bug fixes needed for the release.
Update any version-related files (e.g., package.json, README.md) with the new version number if necessary.
Commit your changes:

```bash
git add .
git commit -m "Prepare for release <version>"
```

Push the changes to the remote repository:

```bash
git push origin main
```

Tag the commit with the release version:

```bash
git tag -a v<version> -m "Version <version>"
git push origin v<version>
```

Replace <version> with the version number you're releasing. For example:

```bash
git tag -a v1.0.0 -m "Version 1.0.0"
git push origin v1.0.0
```

Verify the tag and release on your repository hosting service (e.g., GitHub, GitLab):

Check your repository's tags section to ensure the tag has been created successfully.
Optionally, publish release notes or documentation:

You might want to create release notes or update documentation to reflect the changes introduced in this release.
This streamlined process eliminates the need for managing release branches, making it simpler and more direct.

```yaml
name: Create Grafana Dashboard

on:
  push:
    branches:
      - main
    tags:
      - "v*" # Trigger on push to tags starting with 'v' on the main branch

jobs:
  create_dashboard:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Install dependencies (if needed)
        run: npm install # Adjust as per your project's requirements

      - name: Execute script to create Grafana dashboard
        env:
          GRAFANA_API_KEY: ${{ secrets.GRAFANA_API_KEY }} # Assuming you have a secret named GRAFANA_API_KEY
        run: python create_dashboard.py # Adjust the script name and language as needed
```

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
  workflow_run:
    workflows: ["Create Stage Dashboard"]
    types:
      - completed

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

      - name: Create Pull Request to Main
        uses: peter-evans/create-pull-request@v3
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          branch: dev
          title: "Merge dev branch into main"
          body: "This pull request is to merge changes from dev branch into main."
          labels: "automated-pr"
          reviewers: "username1, username2" # Add reviewers as necessary

      - name: Delete Dev Branch After PR Approval
        if: github.event_name == 'pull_request' && github.event.action == 'closed' && github.event.pull_request.merged == true && github.event.pull_request.base.ref == 'main'
        run: |
          git push origin --delete dev
```

flow down

```yaml
name: Create Grafana Dashboard
on:
  workflow_dispatch:
    inputs:
      dashboard_uid:
        description: "Dashboard UID"
        required: true
      environment:
        description: "Choose one of the following environments: java, node, namespace"
        required: true
        default: "java" # Default value, but user must choose one of the provided options
jobs:
  create_dashboard:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Install dependencies (if needed)
        run: npm install # Adjust as per your project's requirements
      - name: Make curl GET request to Grafana API and store response
        id: grafana_api_request
        run: |
          ENVIRONMENT=${{ github.event.inputs.environment }}
          DASHBOARD_UID=${{ github.event.inputs.dashboard_uid }}
          if [ "$ENVIRONMENT" == "java" ]; then
            curl -X GET -H "Authorization: Bearer $GRAFANA_API_KEY" https://your-grafana-instance/api/dashboards/uid/$DASHBOARD_UID > java.json
          elif [ "$ENVIRONMENT" == "node" ]; then
            curl -X GET -H "Authorization: Bearer $GRAFANA_API_KEY" https://your-grafana-instance/api/dashboards/uid/$DASHBOARD_UID > node.json
          elif [ "$ENVIRONMENT" == "namespace" ]; then
            curl -X GET -H "Authorization: Bearer $GRAFANA_API_KEY" https://your-grafana-instance/api/dashboards/uid/$DASHBOARD_UID > namespace.json
          else
            echo "Invalid environment selected."
            exit 1
          fi

      - name: Save API response to JSON file
        run: echo "${{ steps.grafana_api_request.outputs.stdout }}" > ${{ github.event.inputs.environment }}.json
```

2. Workflow Triggered by PR to Main
   This workflow runs whenever a pull request is made to the main branch. It will use the generated JSON file from the previous workflow.

```yaml
name: Create Stage Dashboard

on:
  workflow_run:
    workflows: ["Fetch Dashboard and Store"]
    types:
      - completed

jobs:
  create_stage_dashboard:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Install Terraform
        uses: hashicorp/setup-terraform@v1
        with:
          terraform_version: 1.0.0

      - name: Initialize Terraform
        run: terraform init

      - name: Apply Terraform Configuration
        run: terraform apply -auto-approve
```

3. Workflow Triggered by PR to Release Tag
   This workflow runs whenever a pull request is made with a tag that starts with release/. It also utilizes the JSON file generated earlier.

```yaml
name: Create Release Dashboard

on:
  push:
    tags:
      - "release/*"

jobs:
  create_release_dashboard:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v2

      # Add steps to create the release dashboard
      - name: Install Terraform
        uses: hashicorp/setup-terraform@v1
        with:
          terraform_version: 1.0.0

      - name: Initialize Terraform
        run: terraform init

      - name: Apply Terraform Configuration
        run: terraform apply -auto-approve
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
