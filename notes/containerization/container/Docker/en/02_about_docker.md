# About Docker

[English](../en/02_about_docker.md) | [繁體中文](../zh-tw/02_about_docker.md) | [日本語](../ja/02_about_docker.md) | [Back to Index](../README.md)

### Docker Basic Architecture
Docker's basic architecture consists of several key components:
1. Docker Engine
   - Provides a standardized container environment
   - Manages system resource allocation (CPU, memory, etc.)
   - Serves as the foundational platform for running containers

2. Docker Container
   - A virtualized runtime space
   - Obtains system resources through Docker Engine
   - Multiple containers can run simultaneously

3. Docker Image
   - Contains all the environment and files required by the application
   - Can be customized based on existing images (e.g., CentOS)
   - The smallest unit of containerized deployment

### Cross-Platform Deployment Mechanism
The key to Docker's cross-platform deployment is:
1. Standardized Environment
   - Install Docker Engine on any operating system (Linux, Windows, MacOS)
   - All environments follow the same containerization standard

2. Docker Hub
   - A cloud space for sharing Docker images
   - Developers can upload their own images
   - Others can download and use these images
   - Enables global resource sharing among developers

### Practical Operation Architecture
In practice, Docker's workflow is as follows:

1. Development Phase
   - Write a Dockerfile to define image specifications
   - Use the `docker build` command to create images
   - Use `docker images` to view existing images

2. Runtime Phase
   - Use `docker run` to start containers
   - Use `docker container ls` to monitor running containers
   - Use the `-p` parameter to set up network ports
   - Use `docker network ls` to view network settings

3. Data Persistence
   - Use Docker Volume for data storage
   - Data is stored outside the container
   - Use the `-v` parameter to mount volumes
   - Use `docker volume ls` to view existing volumes

### Diagram
1. By default, containers are isolated and require special configuration to be exposed externally
2. The design of volumes ensures that data is not lost even if the container is deleted

```
+----------------------+
|        Host          |
+----------------------+
|     Docker Engine    |
+----------------------+
|      Network         | <--- docker run -p / docker network ls
+----------------------+
|      Container       | <--- docker run / docker container ls
|   +--------------+   |
|   |   Volume     |   | <--- docker run -v / docker volume ls
|   +--------------+   |
+----------------------+
|      Image           | <--- docker build / docker images
|   +------------+     |
|   | Dockerfile |     |
|   +------------+     |
+----------------------+
```

#### Description
- This diagram shows the Docker architecture from the perspective of the Docker image, moving upward layer by layer.
- Each layer represents a component in the Docker architecture, from the bottom Image (with Dockerfile inside) to the top Host.
- Volume is a data storage block inside the container, related to external data persistence.
- Docker Engine is the core layer between Network and Host, responsible for coordinating all components.
- Common commands for each layer are marked on the right, corresponding to management and query operations.

According to the diagram, each Docker component can be managed and queried with corresponding commands, which helps in understanding Docker's layered design and practical workflow. 