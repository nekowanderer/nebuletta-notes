# What Problems Does Docker Solve?

[English](../en/01_what_docker_solved.md) | [繁體中文](../zh-tw/01_what_docker_solved.md) | [日本語](../ja/01_what_docker_solved.md) | [Back to Index](../README.md)

```
              Why Use Docker?

   +--------+     +--------+     +---------+
   | Linux  |     |   Mac  |     | Windows |
   |(Docker)|     |(Docker)|     |(Docker) |
   +--------+     +--------+     +---------+
       |            |             |
       +------------+-------------+
                    |
              +-------------+
              |  Dockerhub  |
              +-------------+
               /           \
      +--------+           +--------+
      |  Image |           |  Image |
      +--------+           +--------+
         /                       \
+----------------+          +-----------------+
| Runtime        |          | Database        |
| Command        |          | Command         |
| Source Code    |          | Testing Data    |
+----------------+          +-----------------+
```

### Simplified Deployment Process

One of Docker's primary functions is to simplify application deployment. In traditional deployment methods, we need to:
1. Install specific programming language runtime environments
2. Execute corresponding startup commands (such as Java's JAR or JavaScript's NPM)
3. Prepare customized code

These steps originally required manual integration by engineers, but Docker packages these elements into a single file called an Image, which includes:
- Runtime environment
- Startup commands
- Application code

### Rapid Database Test Environment Deployment

Docker is not only suitable for application deployment but also for quickly setting up database environments:
1. Choose the required database (such as MySQL or SQL Server)
2. Prepare test data
3. Configure startup commands

Through Docker Images, we can:
- Quickly establish test database environments
- Delete and rebuild at any time
- Ensure environment consistency

### Cross-Platform Deployment Capability

Docker's third important function is enabling cross-platform deployment:
1. Share Images through Docker Hub
2. Run on any platform with Docker Engine installed
3. Support different operating systems including Windows, Mac, and Linux

### Summary

Docker provides three core functions:
1. Simplified application deployment process
2. Quick creation of reusable test environments
3. Cross-platform deployment capability

These features make the development and deployment process more standardized and efficient, significantly reducing the complexity of environment configuration. 