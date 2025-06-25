# docker run -it --rm -v ./:/usr/src/mymaven -w /usr/src/mymaven maven:3.8.5-openjdk-17 mvn clean install
FROM maven:3.8.5-openjdk-17 as build
WORKDIR /usr/src/app
COPY ./spring-boot /usr/src/app
RUN mvn clean package -DskipTests
# run a terminal command to verify the build

FROM wodby/openjdk:17-jre-alpine
COPY --from=build /usr/src/app/target/*.jar app.jar
ENTRYPOINT ["java","-jar","/app.jar"]

