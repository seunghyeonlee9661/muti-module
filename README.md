# 🌟 Multi-Module Spring Project with Docker Compose & GitHub Actions
![image](https://github.com/user-attachments/assets/b589649f-6eaf-4350-8a9b-a568d0eb40b3)

Welcome to the **Multi-Module Spring Project** repository! This project demonstrates how to set up a multi-module Spring Boot application using **Docker Compose** for containerization and **GitHub Actions** for continuous deployment.

## 📁 Project Structure

The project is organized as follows:

```
root-project/
├── .github/workflows
│   └── deploy.yml
├── module-common/
│   ├── Dockerfile
│   ├── build.gradle
│   └── src/
├── module-A/
│   ├── Dockerfile
│   ├── build.gradle
│   └── src/
├── module-B/
│   ├── Dockerfile
│   ├── build.gradle
│   └── src/
├── gradle/
├── docker-compose.yml
├── gradlew
├── build.gradle
└── settings.gradle
```

- **`deploy.yml`**: Handles deployment tasks post `git push`, including change detection and Docker execution.
- **`module-common/`**: The base module referenced by other modules.
- **`module-A/`**: The module where the application runs.
- **`docker-compose.yml`**: Commands to build the container using Dockerfile from the specified path.
- **`Dockerfile`**: Handles the creation of the Docker container using the specified files.

## 🚀 Deployment Workflow

The deployment process is automated using GitHub Actions. Here’s a breakdown of the workflow defined in `deploy.yml`:

### `deploy.yml`

```yaml
name: Deploy to Ubuntu Server
on:
  push:
    branches:
      - main
      - develop
jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3
        with:
          fetch-depth: 2  # Ensure previous commits are fetched for change detection

      - name: Set up SSH
        uses: webfactory/ssh-agent@v0.7.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

      - name: Copy files via SSH
        run: |
          rsync -avz -e "ssh -o StrictHostKeyChecking=no" ./ ${{ secrets.USER }}@${{ secrets.HOST }}:/home/leesh/spring/Multi-module

      - name: Deploy with Docker Compose
        run: |
          ssh -o StrictHostKeyChecking=no ${{ secrets.USER }}@${{ secrets.HOST }} << 'EOF'
          cd /home/leesh/spring/Multi-module
          
          # Detect which modules changed
          changed_modules=$(git diff --name-only HEAD^ HEAD)

          # Build only if specific directories have changed
          if echo "$changed_modules" | grep -q '^module-A/'; then
            echo "Changes detected in module-A"
            docker-compose build module-A
          fi

          if echo "$changed_modules" | grep -q '^module-B/'; then
            echo "Changes detected in module-B"
            docker-compose build module-B
          fi
          
          docker-compose up -d
          EOF
```

This workflow automates deployment to the Ubuntu server by detecting changes in specific modules and building only those that have been modified.

## 🐳 Docker Configuration

### `docker-compose.yml`

```yaml
version: '3.8'

services:
  mysql:
    image: mysql:8.0
    container_name: mysql
    environment:
      MYSQL_ROOT_PASSWORD: root_password
      MYSQL_DATABASE: database
      MYSQL_USER: user
      MYSQL_PASSWORD: password
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql

  redis:
    image: redis:latest
    container_name: redis
    ports:
      - "6379:6379"

  module-a:
    build:
      context: ./  # Root context for the build
      dockerfile: module-a/Dockerfile
    container_name: module-a
    ports:
      - "8081:8080"
    depends_on:
      - mysql
      - redis

  module-b:
    build:
      context: ./  # Root context for the build
      dockerfile: module-b/Dockerfile
    container_name: module-b
    ports:
      - "8082:8080"
    depends_on:
      - mysql
      - redis

volumes:
  mysql_data:
```

This `docker-compose.yml` file sets up the services needed for the application, such as MySQL and Redis, and configures each module as a separate Docker container.

### Module `Dockerfile`

Each module has its own `Dockerfile` to define how it should be built and run in a container. Here’s an example from `module-a`:

```docker
# Base image
FROM openjdk:17-jdk-slim

# Set working directory
WORKDIR /app

# Copy Gradle files from the root context to the service context
COPY gradlew /app/
COPY gradle /app/gradle/
COPY build.gradle /app/

COPY module-common/build.gradle /app/module-common/
COPY module-common/src /app/module-common/src

COPY module-a/build.gradle /app/module-a/
COPY module-a/src /app/module-a/src

#Copy settings.gradle for module
COPY module-a/settings.gradle /app/

# Ensure gradlew is executable
RUN chmod +x gradlew

# Build the application
RUN ./gradlew build -x test --stacktrace

# Copy the built JAR file to /app.jar
RUN cp module-a/build/libs/module-a.jar /app.jar

# Expose port
EXPOSE 8080

# Run the application
CMD ["java", "-jar", "/app.jar"]
```

This `Dockerfile` handles the build process for `module-a`, setting up the working directory, copying necessary files, and building the JAR file to be executed.

## 🛠️ Gradle Configuration

### Root `build.gradle`

The root `build.gradle` file manages common dependencies and plugins for all submodules:

```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.3'
    id 'io.spring.dependency-management' version '1.1.6'
}

bootJar {
    enabled = false
}
jar {
    enabled = true
}

repositories {
    mavenCentral()
}

subprojects { // Applies these settings to all submodules
    group 'com.example'
    version '0.0.1-SNAPSHOT'

    sourceCompatibility = '17'

    apply plugin: 'java'
    apply plugin: 'java-library'
    apply plugin: 'org.springframework.boot'
    apply plugin: 'io.spring.dependency-management'

    configurations {
        compileOnly {
            extendsFrom annotationProcessor
        }
    }

    repositories {
        mavenCentral()
    }

    dependencies { // Dependencies applied to all submodules
        implementation 'org.springframework.boot:spring-boot-starter'
        testImplementation 'org.springframework.boot:spring-boot-starter-test'
        implementation 'org.springframework.boot:spring-boot-starter-web'
        developmentOnly 'org.springframework.boot:spring-boot-devtools'
    }

    test {
        useJUnitPlatform()
    }
}
```

### Module-specific `build.gradle`

Each module can have its own `build.gradle` file to define specific dependencies and configurations. Here's an example from `module-a`:

```gradle
dependencies {
    implementation project(':module-common') // Includes the common module as a dependency
}

springBoot {
    mainClass.set('com.example.ModuleAApplication') // Specifies the main class for the module
}

bootJar {
    archiveFileName = 'module-a.jar' // Defines the name of the output JAR file
}
```

### Common Module

The `module-common` contains shared logic and dependencies that other modules can use. For example:

```gradle
bootJar {
    enabled = false
}
jar {
    enabled = true
}

dependencies {
    // Common dependencies can be added here
}
```

The common module is included as a dependency in other modules like `module-a` to avoid code duplication and ensure consistency across the project.

## 🐛 Troubleshooting

If you encounter the following error:

```plaintext
Caused by: org.gradle.api.InvalidUserDataException: Main class name has not been configured and it could not be resolved from classpath
```

Make sure that each module has its own correct `settings.gradle` file, and that it only includes the necessary modules. Incorrect configurations in `settings.gradle` can lead to the build process attempting to include unnecessary modules, causing this error.

## 📝 Conclusion

Setting up a multi-module Spring Boot project with Docker Compose and GitHub Actions allows you to build scalable, maintainable applications. This approach helps in managing microservices efficiently, ensuring that only the necessary parts of the application are rebuilt and redeployed when changes are made.
