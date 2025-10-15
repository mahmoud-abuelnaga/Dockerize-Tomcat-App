FROM maven:3.9.11-amazoncorretto-21-alpine AS build-stage

WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean install

FROM tomcat:10.1.48-jre25-temurin-noble AS production
ARG version="v2"
COPY --from=build-stage /app/target/vprofile-${version}.war /usr/local/tomcat/webapps/ROOT.war