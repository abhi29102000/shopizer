FROM eclipse-temurin:17-jre-alpine
RUN mkdir /opt/app /files
COPY target/shopizer.jar /opt/app
COPY SALESMANAGER.h2.db /
COPY ./files /files
CMD ["java", "-jar", "/opt/app/shopizer.jar"]
