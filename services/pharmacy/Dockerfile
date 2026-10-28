FROM eclipse-temurin:17-jdk-jammy

RUN mkdir /app
WORKDIR /app

ADD ./build/libs/*.jar /app/app.jar

EXPOSE 8082
ENTRYPOINT ["java", "-jar", "app.jar"]