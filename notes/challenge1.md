/************** Setup SQL Docker Image Locally **************/

### Download Sql Docker Image ###
``` 
docker pull mcr.microsoft.com/mssql/server:2017-latest 
```

### Create Network ###
``` 
docker network create <network_name_of choice> 
```

### Run SQL Docker Image ###
``` 
docker run --rm --network <network_name_of_choice> \ 
    -e "ACCEPT_EULA=Y" \
    -e "SA_PASSWORD=<server password>" \
    -p 1433:1433 \
    --name sql1 -h sql1 \
    -d \
    mcr.microsoft.com/mssql/server:2017-latest 
```

### Create DB from SQL Server Plugin ###
-- Create a new database called '<DatabaseName> (see default in repo)'
-- Connect to the 'master' database to run this snippet
USE master
GO
-- Create the new database if it does not exist already
IF NOT EXISTS (
    SELECT name
        FROM sys.databases
        WHERE name = N'DatabaseName'
)
CREATE DATABASE mydrivingDB

###Find command to create DB in terminal ###

## Populate data
```
docker run \ 
    --network <network_name_of_choice>  \
    -e SQLFQDN=<name_of_instance> \
    -e SQLUSER=<sql_user_name> \
    -e SQLPASS=<sql_pasword> \ 
    -e SQLDB=mydrivingDB \ 
    openhack/data-load:v1
```

/************** Run Docker Images Locally **************/

### Dockerfile 0 - user-java ###
```
docker build -f ../../dockerfiles/Dockerfile_0 -t "tripinsights/user-java:1.0" .
```

```
docker run --network <network name> -d \ 
    -p 8080:80 \
    --name user-java \
    -e "SQL_USER=<sql_user_login>"  \
    -e "SQL_PASSWORD=<sql_password>" \
    -e "SQL_SERVER=sql1" \ 
    tripinsights/user-java:1.0  
```

###### health check command for user-java #########
```
curl -i -X GET 'http://localhost:8080/api/user-java/healthcheck'
```

### Dockerfile 1 - tripviewer ###
```
docker build -f ../../dockerfiles/Dockerfile_1 -t "tripinsights/tripviewer:1.0" .
```

```
docker run  
    --network <network_name_of_choice> -d \
    -p 8081:80 
    --name tripviewer 
    -e "USERPROFILE_API_ENDPOINT=http://$ENDPOINT" 
    -e "TRIPS_API_ENDPOINT=http://$ENDPOINT" 
    tripinsights/tripviewer:1.0
```

###### health check command for tripviewer #########
``` 
curl -i -X GET 'http://localhost:8081/api/user/healthcheck' 
```

### Dockerfile 2 - userprofile ###
```
docker build -f ../../dockerfiles/Dockerfile_2 -t "tripinsights/userprofile:1.0" .
```

```
docker run 
    --network network_name_of_choice -d 
    -p 8082:80 
    --name userprofile 
    -e "SQL_USER=<sql_user_name>"  
    -e "SQL_PASSWORD=<sql_password>" 
    -e "SQL_SERVER=sql1" 
    -e SQL_DBNAME="mydrivingDB" 
    tripinsights/userprofile:1.0
```

###### health check command for user #########    
```
curl -i -X GET 'http://localhost:8082/api/user' 
```

### Dockerfile 3 poi ###
```
docker build -f ../../dockerfiles/Dockerfile_3 -t "tripinsights/poi:1.0" .
```

```
docker run 
    --network <network_name_of_choice> -d 
    -p 8083:80 
    --name poi 
    -e "SQL_USER=<sql_user_name>"  
    -e "SQL_PASSWORD=<sql_password" 
    -e "SQL_SERVER=sql1" 
    -e "ASPNETCORE_ENVIRONMENT=Local" 
    tripinsights/poi:1.0
```

###### health check command for user-java #########
```
curl -i -X GET 'http://localhost:8083/api/poi' 
```

### Dockerfile 4 trips ###
```
docker build -f ../../dockerfiles/Dockerfile_4 -t "tripinsights/trips:1.0" . 
```

```
docker run 
    --network <network_name_of_choice> -d \
    -p 8084:80 \ 
    --name trips \
    -e "SQL_USER=<sql_user_name>" \
    -e "SQL_PASSWORD=<sql_password>" 
    -e "SQL_SERVER=sql1" \
    -e "OPENAPI_DOCS_URI=http://$EXTERNAL_IP" \ 
    tripinsights/trips:1.0
```

###### health check command for trips #########
```
curl -i -X GET 'http://localhost:8084/api/trips'
```