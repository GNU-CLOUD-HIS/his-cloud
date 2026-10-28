FROM openjdk:17-ea-jdk

RUN mkdir /app
WORKDIR /app

copy build/libs/*-SNAPSHOT.jar /app/app.jar

EXPOSE 8081
ENTRYPOINT ["java","-jar","app.jar"]