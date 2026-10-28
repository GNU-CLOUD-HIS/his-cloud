FROM eclipse-temurin:17-jre
WORKDIR /app
COPY build/libs/*-SNAPSHOT.jar /app/app.jar

EXPOSE 8083
ENTRYPOINT ["java", "-jar", "app.jar", "--server.port=8083"]