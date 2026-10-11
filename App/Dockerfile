FROM rockylinux:9.3

RUN dnf install -y java-17-openjdk \
    && dnf clean all \
    && rm -rf /var/cache/dnf

RUN useradd --system \
    --create-home \
    --shell /sbin/nologin \
    employeeapp

RUN mkdir -p /opt/employee-app

COPY employee-app.jar /opt/employee-app/employee-app.jar

RUN chown employeeapp:employeeapp /opt/employee-app/employee-app.jar

USER employeeapp

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "/opt/employee-app/employee-app.jar"]