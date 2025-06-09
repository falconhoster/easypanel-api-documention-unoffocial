# EasyPanel API Documentation (Unofficial)

This document details the API endpoints for EasyPanel, as reverse-engineered from the official WHMCS module. The API appears to be based on a tRPC-like JSON-RPC protocol over HTTP. All requests are sent to a base URL, which is your EasyPanel instance URL (e.g., `https://panel.example.com`).

## 1. Authentication

All API requests must be authenticated using a bearer token. The token should be included in the `Authorization` header of every request. You can generate this token in your EasyPanel instance under **Settings -> Users -> Generate API Key**.

**Header Example:**
```
Authorization: YOUR_API_KEY_HERE
Content-Type: application/json
```

## 2. API Request/Response Format

### Request Format
- **GET Requests:** Parameters are URL-encoded within a single `input` query parameter, which contains a JSON object.
  ```
  GET /api/trpc/resource.action?input={"json":{"param":"value"}}
  ```
- **POST Requests:** The body of the request is a JSON object, typically containing a `json` key with the actual payload.
  ```json
  {
    "json": {
      "param": "value"
    }
  }
  ```

### Key Concepts
- **`projectName`**: Projects in EasyPanel act as containers for services. In the context of the WHMCS module, the `projectName` is typically the client's UUID from the WHMCS database.
- **`serviceName`**: The unique name of a service within a project.
- **`serviceType`**: The type of the service, which determines the endpoint path. The module identifies this based on the service name. Common types are `app`, `mysql`, `postgres`, `mongo`, `redis`. The API paths are constructed dynamically, e.g., `/api/trpc/services.app.{action}` or `/api/trpc/services.mysql.{action}`.

---

## 3. Panel Management API

Endpoints for managing the EasyPanel instance itself.

### 3.1. Get Panel Status
Retrieves metadata about the EasyPanel instance, including its version.

- **Endpoint:** `update.getStatus`
- **Method:** `GET`
- **URL:** `/api/trpc/update.getStatus?input={"json":null}`
- **cURL Example:**
  ```bash
  curl -X GET 'https://your-easypanel.com/api/trpc/update.getStatus?input=%7B%22json%22%3Anull%7D' \
  -H 'Authorization: YOUR_API_KEY_HERE'
  ```
- **Success Response (Example):**
  ```json
  {
    "result": {
      "data": {
        "json": {
          "version": "1.43.0",
          "latestVersion": "1.43.0",
          "needsUpdate": false
        }
      }
    }
  }
  ```

---

## 4. Project Management API

Endpoints for creating and managing projects.

### 4.1. List Projects and Services
Retrieves a list of all projects on the panel and the services within them.

- **Endpoint:** `projects.listProjectsAndServices`
- **Method:** `GET`
- **URL:** `/api/trpc/projects.listProjectsAndServices?input={"json":null}`
- **cURL Example:**
  ```bash
  curl -X GET 'https://your-easypanel.com/api/trpc/projects.listProjectsAndServices?input=%7B%22json%22%3Anull%7D' \
  -H 'Authorization: YOUR_API_KEY_HERE'
  ```

### 4.2. Inspect Project
Retrieves detailed information about a single project and its services.

- **Endpoint:** `projects.inspectProject`
- **Method:** `GET`
- **URL:** `/api/trpc/projects.inspectProject?input={"json":{"projectName":"<PROJECT_NAME>"}}`
- **URL Parameters:**
  - `projectName` (string, required): The name of the project to inspect.
- **cURL Example:**
  ```bash
  # Note: The JSON in the input parameter must be URL-encoded
  curl -X GET 'https://your-easypanel.com/api/trpc/projects.inspectProject?input=%7B%22json%22%3A%7B%22projectName%22%3A%22project-uuid-123%22%7D%7D' \
  -H 'Authorization: YOUR_API_KEY_HERE'
  ```

### 4.3. Create Project
Creates a new, empty project.

- **Endpoint:** `projects.createProject`
- **Method:** `POST`
- **URL:** `/api/trpc/projects.createProject`
- **Request Body:**
  ```json
  {
    "json": {
      "name": "project-uuid-123"
    }
  }
  ```
- **Parameters:**
| Parameter | Type | Description |
| :--- | :--- | :--- |
| `name` | String | The name for the new project. Must be unique. |
- **cURL Example:**
  ```bash
  curl -X POST 'https://your-easypanel.com/api/trpc/projects.createProject' \
  -H 'Authorization: YOUR_API_KEY_HERE' \
  -H 'Content-Type: application/json' \
  -d '{"json":{"name":"project-uuid-123"}}'
  ```

---

## 5. Service Management API

Endpoints for creating and managing services within a project.

### 5.1. Create Service from Schema
Creates one or more services (e.g., an application and its database) from a template schema.

- **Endpoint:** `templates.createFromSchema`
- **Method:** `POST`
- **URL:** `/api/trpc/templates.createFromSchema`
- **Request Body (Example for WordPress):**
  ```json
  {
    "json": {
      "name": "Wordpress",
      "projectName": "project-uuid-123",
      "schema": {
        "services": [
          {
            "type": "app",
            "data": {
              "serviceName": "1_wordpress",
              "env": "WORDPRESS_DB_HOST=project-uuid-123_1_mysql\nWORDPRESS_DB_USER=mysql\nWORDPRESS_DB_PASSWORD=GeneratedPassword\nWORDPRESS_DB_NAME=project-uuid-123",
              "source": { "type": "image", "image": "wordpress:latest" },
              "domains": [{ "host": "$(EASYPANEL_DOMAIN)", "https": true, "port": 80, "path": "/" }],
              "mounts": [{ "type": "volume", "name": "data", "mountPath": "/var/www/html" }]
            }
          },
          {
            "type": "mysql",
            "data": { "serviceName": "1_mysql", "password": "GeneratedPassword" }
          }
        ]
      }
    }
  }
  ```
- **Note:** The `data` object for each service contains the specific configuration for that service type.

### 5.2. Destroy Service
Permanently deletes a service.

- **Endpoint:** `services.{serviceType}.destroyService`
- **Method:** `POST`
- **URL:** `/api/trpc/services.app.destroyService` (example for `app` type)
- **Request Body:**
  ```json
  {
    "json": {
      "projectName": "project-uuid-123",
      "serviceName": "1_wordpress"
    }
  }
  ```

### 5.3. Start/Enable Service
Starts a stopped service.

- **Endpoint:**
    - `services.{serviceType}.startService` (for app services on EasyPanel >= v1.43.0)
    - `services.{serviceType}.enableService` (for other services or older EasyPanel versions)
- **Method:** `POST`
- **URL:** `/api/trpc/services.app.startService`
- **Request Body:**
  ```json
  {
    "json": {
      "projectName": "project-uuid-123",
      "serviceName": "1_wordpress"
    }
  }
  ```

### 5.4. Stop/Disable Service
Stops a running service.

- **Endpoint:**
    - `services.{serviceType}.stopService` (for app services on EasyPanel >= v1.43.0)
    - `services.{serviceType}.disableService` (for other services or older EasyPanel versions)
- **Method:** `POST`
- **URL:** `/api/trpc/services.app.stopService`
- **Request Body:**
  ```json
  {
    "json": {
      "projectName": "project-uuid-123",
      "serviceName": "1_wordpress"
    }
  }
  ```

### 5.5. Restart Service
Restarts a service.

- **Endpoint:** `services.{serviceType}.restartService`
- **Method:** `POST`
- **URL:** `/api/trpc/services.app.restartService`
- **Request Body:**
  ```json
  {
    "json": {
      "projectName": "project-uuid-123",
      "serviceName": "1_wordpress"
    }
  }
  ```

### 5.6. Update Service Resources
Updates the CPU and memory limits for a service.

- **Endpoint:** `services.{serviceType}.updateResources`
- **Method:** `POST`
- **URL:** `/api/trpc/services.app.updateResources`
- **Request Body:**
  ```json
  {
    "json": {
      "projectName": "project-uuid-123",
      "serviceName": "1_wordpress",
      "resources": {
        "memoryReservation": 0,
        "memoryLimit": 1024,
        "cpuReservation": 0,
        "cpuLimit": 1024
      }
    }
  }
  ```
- **Parameters:**
| Parameter | Type | Description |
| :--- | :--- | :--- |
| `memoryLimit` | Integer | The maximum RAM in MB the service can use. |
| `cpuLimit` | Integer | The maximum CPU the service can use. (e.g., 1024 for 1 core). |

### 5.7. Update App Domains
Assigns or updates domains for an `app` type service. This is often called immediately after creating a service.

- **Endpoint:** `services.app.updateDomains`
- **Method:** `POST`
- **URL:** `/api/trpc/services.app.updateDomains`
- **Request Body:**
  ```json
  {
    "json": {
      "projectName": "project-uuid-123",
      "serviceName": "1_wordpress",
      "domains": [
        {
          "host": "blog.example.com",
          "https": true,
          "port": 80,
          "path": "/",
          "middlewares": [],
          "certificateResolver": "",
          "wildcard": false
        }
      ]
    }
  }
  ```

---

## 6. File Browser API (Integrated Service)

The module also interacts with a File Browser service, which appears to have a separate, more traditional RESTful API.

### 6.1. Get Auth Token
Authenticates with the File Browser service to get a session token. This is **different** from the main EasyPanel API key.

- **Endpoint:** `/api/login`
- **Method:** `POST`
- **Request Body:**
  ```json
  {
    "username": "your-filebrowser-user",
    "password": "your-filebrowser-password"
  }
  ```
- **Success Response:** A string containing the JWT token.

### 6.2. Create Directory
Creates a directory. The path is specified in the URL.

- **Endpoint:** `/api/resources/{path}`
- **Method:** `POST`
- **URL:** `/api/resources/user-directory-name/?override=false`
- **Authentication:** Requires `X-Auth: YOUR_FILEBROWSER_TOKEN` header.

### 6.3. Add User
Creates a new user within File Browser.

- **Endpoint:** `/api/users`
- **Method:** `POST`
- **Authentication:** Requires `X-Auth: YOUR_FILEBROWSER_TOKEN` header.
- **Request Body:**
  ```json
  {
    "what": "user",
    "which": [],
    "data": {
      "scope": "/user-directory-name",
      "locale": "en",
      "viewMode": "mosaic",
      "singleClick": false,
      "perm": {
        "admin": false, "execute": true, "create": true,
        "rename": true, "modify": true, "delete": true,
        "share": false, "download": true
      },
      "commands": [],
      "username": "new-username",
      "password": "new-password"
    }
  }
 
 Of course. Here is the continuation of the EasyPanel API documentation, focusing on implementation patterns, practical examples from the code, and a summary of how WHMCS actions map to API calls.

---

## 7. API Implementation Details & Patterns

The following patterns were observed in the `sdk.php` file and are crucial for correctly interacting with the API.

### 7.1. Dynamic Service Endpoints (`serviceType`)

Many service-related actions do not use a fixed URL. The endpoint path is constructed dynamically based on the type of service being managed.

-   **Pattern:** `/api/trpc/services.{serviceType}.{action}`
-   **Example:** To restart a WordPress application, the endpoint is `/api/trpc/services.app.restartService`. To restart its associated MySQL database, the endpoint is `/api/trpc/services.mysql.restartService`.

The WHMCS module determines the `{serviceType}` by scanning the service's name for keywords. The logic is as follows:
-   If the name contains "mysql", "mariadb", "postgres", "mongo", or "redis", that keyword is used as the `serviceType`.
-   Otherwise, it defaults to `app`.

### 7.2. Service Creation via Schemas

The most powerful creation endpoint is `templates.createFromSchema`. It allows you to define and deploy a complex, multi-service application (like a web app and its database) in a single API call. The `sdk.php` file provides perfect examples.

#### Example: WordPress Schema Breakdown

This is the JSON payload constructed by the SDK to deploy a WordPress instance.

```json
{
  "json": {
    "name": "Wordpress",
    "projectName": "project-uuid-123",
    "schema": {
      "services": [
        {
          "type": "app",
          "data": {
            "serviceName": "1_wordpress",
            "env": "WORDPRESS_DB_HOST=project-uuid-123_1_mysql\nWORDPRESS_DB_USER=mysql\nWORDPRESS_DB_PASSWORD=GeneratedPassword\nWORDPRESS_DB_NAME=project-uuid-123",
            "source": {
              "type": "image",
              "image": "wordpress:latest"
            },
            "domains": [
              {
                "host": "$(EASYPANEL_DOMAIN)",
                "https": true,
                "port": 80,
                "path": "/"
              }
            ],
            "mounts": [
              {
                "type": "volume",
                "name": "data",
                "mountPath": "/var/www/html"
              },
              {
                "type": "file",
                "content": "upload_max_filesize = 100M\npost_max_size = 100M\n",
                "mountPath": "/usr/local/etc/php/conf.d/custom.ini"
              }
            ]
          }
        },
        {
          "type": "mysql",
          "data": {
            "serviceName": "1_mysql",
            "password": "GeneratedPassword"
          }
        }
      ]
    }
  }
}
```

**Key Components:**
*   **`schema.services`**: An array where each object represents a service to be created.
*   **`type`**: The service type (`app`, `mysql`, etc.), which dictates the available `data` fields.
*   **`data.serviceName`**: The unique name for this service within the project.
*   **`data.env`**: A newline-separated string of environment variables. This is how services are configured and linked.
    *   **Service Linking**: `WORDPRESS_DB_HOST` is set to the `serviceName` of the MySQL container (`project-uuid-123_1_mysql`), allowing WordPress to connect to it.
    *   **Special Variables**: The schema can use variables that EasyPanel replaces at deploy time:
        *   `$(PROJECT_NAME)`: The name of the project.
        *   `$(PRIMARY_DOMAIN)`: The primary domain configured for the service.
        *   `$(EASYPANEL_DOMAIN)`: A placeholder for the domain to be assigned.
*   **`data.source`**: Defines where the application code comes from. Here, it's a Docker `image`.
*   **`data.mounts`**: Defines persistent storage.
    *   `type: "volume"`: Creates a persistent Docker volume.
    *   `type: "file"`: Creates a file on the fly with the specified `content` at the `mountPath`.
*   **`data.domains`**: Configures how the service is exposed to the internet.


### **Disclaimer**

This API documentation was created by reverse-engineering the provided WHMCS module. It is not official and may be incomplete or contain inaccuracies. It should, however, provide a strong and reliable foundation for anyone looking to build integrations with the EasyPanel API.
