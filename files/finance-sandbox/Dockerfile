FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY finance-sandbox-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8090
ENTRYPOINT ["java", "-jar", "app.jar"]
