### Github scraping script

```python

import requests

# Replace 'your_github_token' with your actual GitHub personal access token
GITHUB_TOKEN = 'your_github_token'

# List of GitHub repositories to check
repos = [
    'owner/repo1',
    'owner/repo2',
    'owner/repo3',
    # Add more repositories as needed
]

# Docker image to search for in Dockerfiles
docker_image = 'acr.ga.com/nginx-alpine:latest'

# GitHub API URL
GITHUB_API_URL = 'https://api.github.com'

# Headers for authentication
headers = {
    'Authorization': f'token {GITHUB_TOKEN}',
    'Accept': 'application/vnd.github.v3+json'
}

def check_dockerfile_for_image(repo):
    # Construct the URL to get the contents of the repository
    contents_url = f'{GITHUB_API_URL}/repos/{repo}/contents/'

    response = requests.get(contents_url, headers=headers)
    if response.status_code != 200:
        print(f'Failed to fetch contents of {repo}')
        return False

    # Get the list of files and directories in the root of the repository
    contents = response.json()
    for item in contents:
        if item['type'] == 'file' and item['name'].lower() == 'dockerfile':
            # Get the contents of the Dockerfile
            dockerfile_url = item['download_url']
            dockerfile_response = requests.get(dockerfile_url)
            if dockerfile_response.status_code == 200:
                dockerfile_contents = dockerfile_response.text
                # Check if the specified Docker image is in the Dockerfile
                if docker_image in dockerfile_contents:
                    return True
    return False

def main():
    repos_with_docker_image = []

    for repo in repos:
        if check_dockerfile_for_image(repo):
            repos_with_docker_image.append(repo)

    if repos_with_docker_image:
        print('Repositories with the specified Docker image in their Dockerfile:')
        for repo in repos_with_docker_image:
            print(repo)
    else:
        print('No repositories found with the specified Docker image in their Dockerfile.')

if __name__ == '__main__':
    main()


```

### End

k=1
url*1=https://www.cnn.com/
user=myuser
password=mypass
eval "curl -u $user:$password \"\${url*$k}\""

java -jar target/app.jar --library-dependencies target/lib/

spearate jar with target

```xml
<project>
    <!-- Other configurations -->

    <build>
        <plugins>
            <!-- Copy dependencies to the lib folder -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-dependency-plugin</artifactId>
                <version>3.1.2</version>
                <executions>
                    <execution>
                        <id>copy-dependencies</id>
                        <phase>prepare-package</phase>
                        <goals>
                            <goal>copy-dependencies</goal>
                        </goals>
                        <configuration>
                            <outputDirectory>${project.build.directory}/lib</outputDirectory>
                        </configuration>
                    </execution>
                </executions>
            </plugin>

            <!-- Build a standalone JAR -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <version>3.2.0</version>
                <configuration>
                    <archive>
                        <manifest>
                            <addClasspath>true</addClasspath>
                            <classpathPrefix>lib/</classpathPrefix>
                            <mainClass><!-- Your main class here --></mainClass>
                        </manifest>
                    </archive>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

To configure mutual TLS (mTLS) with backend services in Spring Cloud Gateway using only properties.yaml (application.yaml), you can leverage Spring Boot's auto-configuration capabilities along with custom properties. Here's how you can achieve this:

Configure SSL/TLS for Spring Cloud Gateway:

You can configure SSL/TLS for Spring Cloud Gateway using properties in the application.yaml file. Specify the keystore and truststore paths along with other SSL-related properties.

```yaml
server:
  port: 8443
  ssl:
    key-store: classpath:keystore.p12
    key-store-password: changeit
    key-store-type: PKCS12
    trust-store: classpath:truststore.p12
    trust-store-password: changeit
    trust-store-type: PKCS12
    client-auth: need
```

Configure Route for Backend Service:

Define routes in the application.yaml file to specify the backend services along with any additional configuration required for mTLS.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: backend_route
          uri: https://backend.example.com
          predicates:
            - Path=/backend/**
          filters:
            - PrefixPath=/api
```

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

Ensure Backend Service Requires mTLS:

Make sure your backend service is configured to require client certificates for authentication.

Testing and Debugging:

Test your setup to ensure mutual TLS authentication is working correctly between the gateway and backend services. Debug any issues that may arise during the configuration process.

With the above configuration, Spring Cloud Gateway should automatically handle mTLS authentication with the backend services without requiring any Java code changes. Make sure to replace the placeholder values (keystore.p12, truststore.p12, https://backend.example.com, etc.) with the actual values relevant to your setup.
