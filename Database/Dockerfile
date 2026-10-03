FROM mysql:9.7-oracle

COPY ./initdb/schema.sql /docker-entrypoint-initdb.d/01-schema.sql
COPY ./initdb/data.sql /docker-entrypoint-initdb.d/02-data.sql

WORKDIR /var/lib/mysql

ARG employee_db_version=1.0.0

EXPOSE 3306