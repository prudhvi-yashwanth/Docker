# Production-Ready Spring Boot Dockerfile

```dockerfile
# Use a lightweight JRE (not JDK) to save space
FROM eclipse-temurin:17-jre-alpine

# Set the working directory
WORKDIR /app

# Create a non-root user for security (banking auditors require this)
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy the pre-built JAR from the target folder
# (Assume it was built outside Docker, e.g., by GitHub Actions)
COPY target/app.jar /app/app.jar

# Change ownership to the non-root user
RUN chown -R appuser:appgroup /app

# Switch to the non-root user
USER appuser

# Set default JVM options (can be overridden at runtime)
ENV JAVA_OPTS="-Xmx512m"

# Document the port
EXPOSE 8080

# Run the application using exec form for proper signal handling
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar /app/app.jar"]
```


## Explanation

Starting with a lightweight **JRE Alpine** image keeps the size small. A **non-root user** is created immediately because running as root inside a container is a security risk — especially for a banking client like SBI.

The JAR is copied, ownership is transferred to the non-root user, and default JVM heap limits are set via `ENV JAVA_OPTS`. The `ENTRYPOINT` uses `sh -c` specifically to allow the `$JAVA_OPTS` environment variable to expand properly at runtime, so operations can tune memory without rebuilding the image.