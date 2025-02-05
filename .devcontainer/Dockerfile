# Optimized Multi-Stage Dockerfile with Checkstyle Suppressions

# Stage 1: Dependency Caching and Build
FROM maven:3.8.6-eclipse-temurin-17-alpine AS builder
WORKDIR /app

# Create Checkstyle suppressions file
RUN echo '<?xml version="1.0"?>' > checkstyle-suppressions.xml && \
    echo '<!DOCTYPE suppressions PUBLIC "-//Checkstyle//DTD SuppressionFilter Configuration 1.2//EN" "https://checkstyle.org/dtds/suppressions_1_2.dtd">' >> checkstyle-suppressions.xml && \
    echo '<suppressions>' >> checkstyle-suppressions.xml && \
    echo '    <suppress files=".*\.jar" checks="NoHttp"/>' >> checkstyle-suppressions.xml && \
    echo '    <suppress files=".*\.pom" checks="NoHttp"/>' >> checkstyle-suppressions.xml && \
    echo '</suppressions>' >> checkstyle-suppressions.xml

# Copy only necessary files for dependency resolution
COPY pom.xml .
COPY src ./src
COPY .mvn .mvn
COPY mvnw ./
RUN chmod +x mvnw

# Efficient dependency download and build
RUN ./mvnw dependency:go-offline \
    -Dcheckstyle.suppressions.location=checkstyle-suppressions.xml \
    --no-transfer-progress \
    -Dmaven.wagon.rto=5000

# Compile the application with optimizations
RUN ./mvnw package \
    -DskipTests \
    -Dcheckstyle.suppressions.location=checkstyle-suppressions.xml \
    -Dmaven.compiler.source=17 \
    -Dmaven.compiler.target=17 \
    -Dmaven.compiler.optimize=true \
    -Dmaven.compiler.debug=false

# Stage 2: Slim Runtime Image
FROM eclipse-temurin:17-jre-alpine
LABEL maintainer="Your Organization" \
      description="Optimized Java Application Container"

WORKDIR /app

# Copy only the necessary JAR file
COPY --from=builder /app/target/*.jar app.jar
COPY --from=builder /app/checkstyle-suppressions.xml checkstyle-suppressions.xml

# Create a non-root user
RUN addgroup -S appuser && adduser -S -G appuser appuser
USER appuser

# Optimized JVM Configuration
ENV JAVA_TOOL_OPTIONS="-XX:+UseG1GC \
    -XX:MaxRAMPercentage=75.0 \
    -XX:+OptimizeStringConcat \
    -XX:+UseStringDeduplication \
    -XX:+ParallelRefProcEnabled \
    -Djava.security.egd=file:/dev/./urandom"

# Expose application port
EXPOSE 8080

# Lightweight entrypoint with checkpoint
ENTRYPOINT ["java", "-jar", "app.jar"]

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -q -O- http://localhost:8080/actuator/health || exit 1
