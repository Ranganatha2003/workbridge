# WorkBridge — Backend Project Setup

## 1. Purpose

This document defines the initial setup of the WorkBridge backend using Java and Spring Boot.

The objective is to establish a clean, maintainable backend foundation before implementing business functionality.

This phase covers:

- Development environment.
- Java and Spring Boot configuration.
- Maven project setup.
- Required dependencies.
- Package structure.
- Application configuration.
- MySQL connectivity.
- Initial application startup.

No business modules will be implemented in this step.

---

## 2. Development Environment

| Tool | Purpose |
|---|---|
| IntelliJ IDEA | Java development environment |
| Java JDK 17 | Backend development |
| Spring Boot | Backend framework |
| Maven | Dependency management and build |
| MySQL 8.x | Relational database |
| MySQL Workbench | Database management |
| Git | Version control |
| GitHub | Source code repository |
| Docker | Backend containerization |

**Java version:** Java 17.

**Spring Boot version:** Select a stable Spring Boot 3.x release compatible with Java 17 when creating the project. Record the exact version in the project configuration and keep it consistent across the application.

---

## 3. Create the Spring Boot Project

Use Spring Initializr or IntelliJ IDEA to generate the Spring Boot project.

Project configuration:

| Setting | Value |
|---|---|
| Project | Maven |
| Language | Java |
| Group | `com.workbridge` |
| Artifact | `workbridge-backend` |
| Name | `workbridge-backend` |
| Package name | `com.workbridge` |
| Packaging | Jar |
| Java | 17 |

The generated project will be the foundation for the backend.

---

## 4. Required Dependencies

The following dependencies are required for the existing WorkBridge functionality.

| Dependency | Purpose |
|---|---|
| Spring Web | Build REST APIs |
| Spring Data JPA | Database persistence |
| Spring Security | Authentication and authorization |
| MySQL Driver | MySQL connectivity |
| Validation | Request validation |
| Lombok | Reduce repetitive Java code |
| Spring Boot DevTools | Development convenience |

JWT support will also be required for authentication. Select compatible JWT libraries and versions when implementing the security module.

**Important:** Avoid adding unrelated dependencies. Add a dependency only when it supports an existing WorkBridge requirement.

---

## 5. Initial Backend Directory Structure

The initial project structure will be:

```text
workbridge-backend/
│
├── src/
│   │
│   ├── main/
│   │   ├── java/
│   │   │   └── com/
│   │   │       └── workbridge/
│   │   │           │
│   │   │           ├── config/
│   │   │           ├── security/
│   │   │           ├── controller/
│   │   │           ├── service/
│   │   │           ├── repository/
│   │   │           ├── entity/
│   │   │           ├── dto/
│   │   │           ├── exception/
│   │   │           │
│   │   │           └── WorkBridgeApplication.java
│   │   │
│   │   └── resources/
│   │       └── application.properties
│   │
│   └── test/
│
├── pom.xml
└── .gitignore
```

The packages will be populated gradually as we implement each module.

---

## 6. Maven Configuration

The `pom.xml` file manages:

- Project metadata.
- Java version.
- Spring Boot dependencies.
- Third-party libraries.
- Build configuration.

The Maven configuration must use a Spring Boot parent version compatible with Java 17.

Dependency versions should be managed through the Spring Boot dependency management wherever possible.

Avoid manually specifying versions for Spring Boot-managed dependencies unless there is a specific compatibility requirement.

---

## 7. Application Configuration

The application configuration will be maintained in:

`src/main/resources/application.properties`

Configuration responsibilities include:

- Application name.
- Server port.
- MySQL connection.
- Database username and password.
- JPA configuration.
- Hibernate schema behavior.

Database credentials should not be committed to GitHub. Use environment variables or a local configuration file excluded from version control.

Example configuration using environment variables:

```properties
spring.application.name=workbridge-backend

server.port=${SERVER_PORT:8080}

spring.datasource.url=${DB_URL}
spring.datasource.username=${DB_USERNAME}
spring.datasource.password=${DB_PASSWORD}

spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
```

**Note:** `ddl-auto=update` is suitable for initial local development only. The schema management strategy must be reviewed before production deployment.

---

## 8. MySQL Database Preparation

Create the development database:

```sql
CREATE DATABASE workbridge_db;
```

The backend will connect to this database using the configured environment variables.

Example local environment values:

```text
DB_URL=jdbc:mysql://localhost:3306/workbridge_db
DB_USERNAME=your_mysql_username
DB_PASSWORD=your_mysql_password
SERVER_PORT=8080
```

Do not put actual passwords in the committed configuration file.

Database tables and relationships will be created during Phase 3 — Database Design and Implementation.

---

## 9. Initial Application Class

The generated Spring Boot application class will be:

`WorkBridgeApplication.java`

Its responsibility is to start the Spring Boot application and initialize the application context.

No business functionality should be placed inside the main application class.

---

## 10. Initial Application Startup

After generating the project and configuring the dependencies:

1. Open the project in IntelliJ IDEA.
2. Allow Maven to download dependencies.
3. Configure the MySQL environment variables.
4. Ensure the MySQL server is running.
5. Run `WorkBridgeApplication.java`.
6. Verify that the Spring Boot application starts successfully.
7. Check the application startup logs.

Expected result:

- Maven build succeeds.
- Spring Boot application starts.
- Application context initializes successfully.
- MySQL connection configuration is valid.
- Application listens on the configured port.

---

## 11. Git Configuration

The backend source code will be maintained in the existing WorkBridge GitHub repository.

Sensitive files must be excluded from version control.

Example `.gitignore` entries:

```gitignore
target/
.idea/
*.iml
.env
.env.*
application-local.properties
```

Commit the Maven project configuration and application source code, but never commit database passwords or JWT secrets.

---

## 12. Completion Criteria

- [ ] Spring Boot project generated.
- [ ] Java 17 configured.
- [ ] Maven project imports successfully.
- [ ] Required dependencies added.
- [ ] Backend package structure created.
- [ ] MySQL development database created.
- [ ] Application configuration completed.
- [ ] Sensitive configuration excluded from Git.
- [ ] Spring Boot application starts successfully.

**End of Backend Project Setup Document**
