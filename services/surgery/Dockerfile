FROM eclipse-temurin:17-jre-jammy

RUN mkdir /app
WORKDIR /app

COPY ./build/libs/*-SNAPSHOT.jar /app/app.jar

EXPOSE 8084
ENTRYPOINT ["java", "-jar", "app.jar"]