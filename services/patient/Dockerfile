FROM eclipse-temurin:17-jre
RUN mkdir /app
WORKDIR /app
COPY ./build/libs/patient-service-0.0.1-SNAPSHOT.jar /app/app.jar
EXPOSE 8087
ENTRYPOINT ["java", "-jar", "app.jar"]