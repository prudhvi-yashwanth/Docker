# Multi-Stage Dockerfile — Spring Boot (Maven)

```dockerfile
# ---------- STAGE 1: BUILDER ----------
# Use the full Maven + JDK image to compile the code
FROM maven:3.8-openjdk-17 AS builder

WORKDIR /build

# Copy ONLY pom.xml first to cache dependencies (speeds up rebuilds)
COPY pom.xml .
RUN mvn dependency:go-offline

# Now copy the source code and build the JAR
COPY src ./src/
RUN mvn clean package -DskipTests

# ---------- STAGE 2: RUNTIME ----------
# Use the lightweight JRE for the final image
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

# Create a non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy the JAR from the builder stage (not from the local machine!)
COPY --from=builder /build/target/app.jar /app/app.jar

# Set ownership and switch to non-root user
RUN chown -R appuser:appgroup /app
USER appuser

# Set JVM options
ENV JAVA_OPTS="-Xmx512m"

EXPOSE 8080

# Run the application
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar /app/app.jar"]
```

---

## Explanation

This is a **multi-stage build** with two stages:

**Stage 1 — `builder`:** Uses the full Maven + JDK image to compile the code. `pom.xml` is copied first and `mvn dependency:go-offline` is run to cache all dependencies in their own layer. This means subsequent builds only re-download dependencies when `pom.xml` changes — not on every code change.

**Stage 2 — runtime:** Uses only the lightweight **JRE Alpine** image. The final `app.jar` is copied from the builder stage using `COPY --from=builder`. Maven, the JDK, and all source code are left behind entirely.

**Result:** The final image drops from ~700MB to **under 200MB**, and security improves because build tools are not present in the runtime container.
