FROM postgres:14.1-alpine

COPY sqlScripts/01-CreateScheme.sql /docker-entrypoint-initdb.d/
COPY sqlScripts/02-InsertData.sql /docker-entrypoint-initdb.d/
