+++
title = "Containerize Application"
chapter = false
weight = 1
+++

## Containerize Application

### Expected Outcome:
* Install Docker Tooling
* Build the Pet Store application within a Container
* Deploy the Pet Store application to a Wildfly based Application Server

**Lab Requirements:** None

**Average Lab Time:** 30-45 minutes


### Introduction
The Pet Store application is a Java 7 based application that runs on top of the Wildfly (JBoss) application server. The application is built using Maven. We will walk through the steps to build the application using a Maven container then deploy our WAR file to the Wildfly container. We will be using multi-stage builds to facilitate creating a minimal docker container.

#### Building the Dockerfile

**Step 1**: To get started, switch to the `containerize-application` folder within this repository. Then open the `Dockerfile` in your editor of choice.
```
cd ~/environment/aws-modernization-workshop/modules/containerize-application/
```

#### Reviewing the Dockerfile
The first `stage` of our `Dockerfile` will be to build the application. We will use the `maven:3.5-jdk-7` container from Docker Hub as our base image. The first `stage` of our `Dockerfile` will need to copy the code into the container, install the dependencies, then build the application using Maven.

The second `stage` of our `Dockerfile` will be to build our Wildfly application server. We will need to copy the WAR file we created in the first stage to the second stage. The WAR file is built to `/usr/src/app/target/applicationPetstore.war` in the first stage. We will instruct `docker` to copy that WAR file to the `/opt/jboss/wildfly/standalone/deployments/applicationPetstore.war` in the new `stage`. We will also have `docker` copy the `standalone.xml` file to the proper location `/opt/jboss/wildfly/standalone/configuration/standalone.xml`. Wildfly will deploy that application on boot. 

The jPetstore application requires the PostrgeSQL driver. We install that during image build by declaring a `RUN` statement in the `Dockerfile` like the one below.

```
# install postgresql support
RUN mkdir -p $JBOSS_HOME/modules/system/layers/base/org/postgresql/main
COPY ./postgresql $JBOSS_HOME/modules/system/layers/base/org/postgresql/main
RUN /bin/sh -c '$JBOSS_HOME/bin/standalone.sh &' \
  && sleep 10 \
  && $JBOSS_HOME/bin/jboss-cli.sh --connect --command="/subsystem=datasources/jdbc-driver=postgresql:add(driver-name=postgresql,driver-module-name=org.postgresql, driver-class-name=org.postgresql.Driver)" \
  && $JBOSS_HOME/bin/jboss-cli.sh --connect --command=:shutdown \
  && rm -rf $JBOSS_HOME/standalone/configuration/standalone_xml_history/ \
  && rm -rf $JBOSS_HOME/standalone/log/*
```

Let's now review the full `Dockerfile` step for step and walk through the result. The first stage will be based off our `build tools` container, in this case that is maven. This container could be a container within your organization that houses all the tools your development team needs. We alias this first stage as the `build` stage.
```
FROM maven:3.5-jdk-7 AS build
```

Next we are going to install the dependencies. We do this ahead of time so that we can leverage the Docker build cache to increase the speed of build times. To accomplish this we copy the pom.xml and install the dependencies. We then copy our application code and build the application. This enables developers to iterate on their code without having to install dependencies each time.
```
# set the working directory
WORKDIR /usr/src/app

# copy just the pom.xml
COPY ./app/pom.xml /usr/src/app/pom.xml

# just install the dependencies for caching
RUN mvn dependency:go-offline
```

Finally, we build the application using `maven`.
```
# copy the application code
COPY ./app /usr/src/app

# package the application
RUN mvn package -Dmaven.test.skip=true
```

Next, in the same `Dockerfile` we create a new stage called `application.` Then from the `build` stage, we copy the WAR file we compiled to the Wildfly deployments folder. Wildfly will deploy this WAR on boot. We then expose port 8080 for our application and port 9990 for Wildfly management. We then set up the prerequisites for the `ENTRYPOINT` script to start the application.

```
# create our Wildfly based application server
FROM jboss/wildfly:11.0.0.Final AS application

# install postgresql support
RUN mkdir -p $JBOSS_HOME/modules/system/layers/base/org/postgresql/main
COPY ./postgresql $JBOSS_HOME/modules/system/layers/base/org/postgresql/main
RUN /bin/sh -c '$JBOSS_HOME/bin/standalone.sh &' \
  && sleep 10 \
  && $JBOSS_HOME/bin/jboss-cli.sh --connect --command="/subsystem=datasources/jdbc-driver=postgresql:add(driver-name=postgresql,driver-module-name=org.postgresql, driver-class-name=org.postgresql.Driver)" \
  && $JBOSS_HOME/bin/jboss-cli.sh --connect --command=:shutdown \
  && rm -rf $JBOSS_HOME/standalone/configuration/standalone_xml_history/ \
  && rm -rf $JBOSS_HOME/standalone/log/*

# copy war file from build layer to application layer
COPY --from=build /usr/src/app/target/applicationPetstore.war /opt/jboss/wildfly/standalone/deployments/applicationPetstore.war

# install nc for entrypoint script and copy the entrypoint script
USER root
RUN yum install nc -y
USER jboss
COPY ./docker-entrypoint.sh /opt/jboss/docker-entrypoint.sh

# copy our configuration
COPY ./standalone.xml /opt/jboss/wildfly/standalone/configuration/standalone.xml

# expose the application port and the management port
EXPOSE 8080 9990

# run the application
ENTRYPOINT [ "/opt/jboss/wildfly/bin/standalone.sh" ]
CMD [ "-b", "0.0.0.0", "-bmanagement", "0.0.0.0" ]
```

#### Defining the Application
For this workshop we use the tool `docker-compose` to simulate a full-stack development environment that consists of multiple containers communicating with each other. 

**Step 1**: Download the docker-compose binary
```
sudo curl -kLo ~/bin/docker-compose https://github.com/docker/compose/releases/download/1.22.0/docker-compose-$(uname -s)-$(uname -m)
sudo chmod +x ~/bin/docker-compose
exec $SHELL
```

Next, we need to define how our application will run. We do this by defining the structure of our application and it's dependencies in a `docker-compose.yml` file. This file contains the complete environment required for our application. 

**Step 2**: Open the file called `docker-compose.yml`. Let's review the contents. To start your file should look like this:
```
version: '3.4'

services:
```

The next section defines the PostgreSQL service. The PostgreSQL service will run our database. We will use the official PostgreSQL image available from Docker Hub. Next, we will map the PostgreSQL port `5432` to the our machine port for easy access. Finally, we will define a few environment variables to configure our instance.
```
version: '3.4'

services:

  postgres:
    image: postgres:9.6
    ports:
      - 5432:5432
    environment:
      - 'POSTGRES_DB=petstore'
      - 'POSTGRES_USER=admin'
      - 'POSTGRES_PASSWORD=password'
```

Finally, we will define our Pet Store application. Docker Compose supports building containers as well, so we will use a special syntax for defining this container. In our `yaml` file we will create a new service called `petstore` and configure our build configuration. Next, will add a `depends_on` config so that the `petstore` container boots after our `postgres` container. Similar to our `postgres` ports we will map port 8080 to our machine for easy access. Now we will use some environment variables to configure our database with the application.

```
  petstore:
    build: ./
    depends_on:
      - postgres
    ports:
      - 8080:8080
    environment:
      - 'DB_URL=jdbc:postgresql://postgres:5432/petstore?ApplicationName=applicationPetstore'
      - 'DB_HOST=postgres'
      - 'DB_PORT=5432'
      - 'DB_NAME=petstore'
      - 'DB_USER=admin'
      - 'DB_PASS=password'
```

#### Running the Application
To run the application we will execute the following Docker Compose commands.

**Step 1**: Run the database container in the background (`-d` or daemon flag). We don't need the database logs to clog our application logs.
```
docker-compose up -d postgres
```

**Step 2**: Build out petstore application.
```
docker-compose build petstore
```

**Step 3**: Run the application container in the foreground and live stream the logs to stdout. If you hit an error hit `Ctrl+C`, make updates to the Dockerfile and re-build the container using step 2.
```
docker-compose up petstore
```

**Step 4**: To preview the application you will need to click *Preview* from the top menu of the Cloud9 environment. This will open a new window and pre-populate the full URL to your preview domain.
