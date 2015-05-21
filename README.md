<!--

     Copyright 2005-2015 Red Hat, Inc.

     Red Hat licenses this file to you under the Apache License, version
     2.0 (the "License"); you may not use this file except in compliance
     with the License.  You may obtain a copy of the License at

        http://www.apache.org/licenses/LICENSE-2.0

     Unless required by applicable law or agreed to in writing, software
     distributed under the License is distributed on an "AS IS" BASIS,
     WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or
     implied.  See the License for the specific language governing
     permissions and limitations under the License.

-->
# ESB Transactions

## Overview
This example will show you how to leverage the JTA transaction manager provided by Fuse ESB when working with JMS
or JTA Camel endpoints.  We will setup a route that reads messages from a queue and inserts information into a database
using JTA and XA transactions and deploy that onto JBoss Fuse 6.2.

## What You Will Learn
In studying this example you will learn:
* how to set up an XA-aware DataSource
* how to configure a JPA persistence unit
* how to leverage Fuse ESB's JTA and JPA support in your routes
* how to configure a JMS component to support XA
* how to define a transactional route
* how to configure a ResourceManager that can recover XA transactions after a crash

# Initial setup

We will refer to the root directory of `camel-persistence-part2` project as `$PROJECT_HOME`.

## Prerequisites
Before building and running this example you need:

* Maven 3.2.3 or higher
* JDK 1.7
* JBoss Fuse 6.2.0
* Docker-enabled operating system

## Files in the Example
* `pom.xml` - the Maven POM file for building the example
* `datasource` - contains the JDBC data source definition
* `database` - contains the persistence unit definition and the JPA entity beans
* `routing` - contains the transactional Camel routes
* `features` - contains the Apache Karaf features definition that allows for easy installation of this example

For more information about these Maven modules, have a look at the README.md file in every module directory.

# Setting up docker-based databases

To perform tests in more realistic environments, we can leverage the power of Docker to run more advanced database servers.
Of course you can use existing database instances. The below examples are just here for completeness.

## PostgreSQL database

We can use *official* PostgreSQL docker image available at [docker hub](https://registry.hub.docker.com/_/postgres/).
You can use any of available methods to access PostgreSQL server (e.g., by mapping ports or connecting to containers IP address directly).

1. Start PostgreSQL server docker container:

        $ docker run -d --name fuse-postgresql-server -e POSTGRES_USER=fuse -e POSTGRES_PASSWORD=fuse -p 5432:5432 postgres:9.4

2. Create `transactions` database from the `fuse-postgresql-server` container:

        $ docker exec -ti fuse-postgresql-server /bin/bash
        root@b052efff5a53:/# psql -U fuse -d fuse
        psql (9.4.0)
        Type "help" for help.
        fuse=# create database transactions owner fuse encoding 'utf8';
        CREATE DATABASE
        fuse=# \q

3. Initialize database `transactions` by creating schema, table and populating the table with data.

    Simply run:

        $ cd $PROJECT_HOME/datasource
        $ mvn -Ppostgresql

    This will produce standard Maven output with single information:

        [INFO] --- exec-maven-plugin:1.3.2:java (default-cli) @ datasource ---
        10:39:54.826 INFO  [o.j.f.e.t.t.DbInsert] : Database postgresql initialized successfully

4. Configure PostgreSQL database to allow XA transactions by setting `max_prepared_transactions` to the value equal or greater
than `max_connections` setting (`100` in the case of `postgres:9.4` image).

        root@b052efff5a53:/# sed -i 's/^#max_prepared_transactions = 0/max_prepared_transactions = 200/' /var/lib/postgresql/data/postgresql.conf

5. Restart `fuse-postgresql-server` container. Your PostgreSQL database is ready to use.

## Building the Example
In the directory where this README.md file is found, run `mvn clean install` to build the example.

## Running the Example
We will refer to the directory that contains your Fuse ESB installation as `$ESB_HOME`.

### Configuring additional users
Before we can start Fuse ESB, we have to make sure we configure a user we can use later on to connect to the embedded
message broker and send messages to a queue.  Edit the `$ESB_HOME/etc/users.properties` file and add a line that says:

    admin=admin,Administrator

The syntax for this line is &lt;userid&gt;=&lt;password&gt;,&lt;role&gt;, so we're creating a user called `admin` with a password `admin`
who has the `Administrator` role.

### Start JBoss Fuse
Start JBoss Fuse with these commands

* on Linux/Unix/MacOS: `bin/fuse`
* on Windows: `bin\fuse.bat`

### Adding the features repository
To allow for easy installation of the example, we created a features descriptor.  On Fuse ESB's console, add the
extra features repository with this command:

    JBossFuse:karaf@root> features:addurl mvn:org.fusesource.example.transactions/features/1.0-SNAPSHOT/xml/features

### Install the example using the feature
First, install the feature itself using this command:

    JBossFuse:karaf@root> features:install transactions-openjpa-demo

Using `osgi:list` in the console, you should now see this demo's bundles at the bottom of the list.


### Using jconsole to send JMS messages
Open `jconsole` and connect to the running Fuse ESB Enterprise instance.  If the instance is running locally, connect to
the process called `org.apache.karaf.main.Main`.

On the MBeans tab, navigate to `org.apache.activemq` &rarr; `Broker` &rarr; `amq` &rarr; `Queue` &rarr; `Input.Flights`.  Send a few
messages to the queue using the `sendTextMessage(String body, String user, String password)` operation.  For the second
and third parameter, use the username and password you configured earlier.  The first parameter will become the flight ID
in the database, so just use your imagination for that one ;)

The ESB log file will contain logging output similar to :
2013-03-13 12:33:28,946 | INFO| r[Input.Flights] | route1 | ? ? | 130 - org.apache.camel.camel-core - 2.10.0.redhat-60015 | Received JMS message TXL-1000
2013-03-13 12:33:28,958 | INFO| r[Input.Flights] | route1 | ? ? | 130 - org.apache.camel.camel-core - 2.10.0.redhat-60015 | Storing [flight TXL-1000 from DEN to LAS] in the database


### Verifying the result
Now, head back to `ij` and run this SQL query:

    ij> select * from flights;

You will see new database rows for every message you sent, using the message body as the flight number.

## More information
For more information see:

* JBoss Fuse 6.1 - [Transaction Guide](https://access.redhat.com/site/documentation/en-US/Red_Hat_JBoss_Fuse/6.1/html/Transaction_Guide/index.html)



## NOTE: 
For more verbose logging about the use of XA transactions, this logging 
configuration can be applied on the Karaf shell:

log:set DEBUG org.apache.activemq.transaction
log:set DEBUG org.springframework.transaction
log:set DEBUG org.springframework.jms.connection.JmsTransactionManager
log:set DEBUG org.springframework.orm.jpa.JpaTransactionManager
log:set TRACE org.apache.geronimo.transaction.manager.WrapperNamedXAResource
log:set DEBUG org.apache.geronimo.transaction.log
log:set DEBUG org.jencks

This will log every tx.begin, tx.prepare and tx.commit operation to data/log/fuse.log.
