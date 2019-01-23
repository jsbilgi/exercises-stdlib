# base-connectors
Anant - the Base_Connectors are collectors, receivers, processors and transmitters. This repository will store all of the connectors needed.

* collect - gather data from systems locally
* transform - transform data into a standard
* archive - archive data to archive (S3)
* process - load data to base source (data lake) and correlate into (base stage)
* publish - publish data from base search to base objects and base search  

(TRH 3/5/2017) As Teachstone is beginning to maintain this project, we will accumulate documentation below about how to setup the project locally and build/debug the various components.

## initial setup
note: assumes TSDEV_ROOT env var is setup per common teachstone developer setup
```
mkdir $TSDEV_ROOT/dw
cd $TSDEV_ROOT/dw
git clone git@github.com:TeachstoneLLC/base-schema.git
git clone git@github.com:TeachstoneLLC/base-connectors.git
git clone git@github.com:TeachstoneLLC/base-engine.git
git clone git@github.com:TeachstoneLLC/base-devops.git
git clone git@github.com:TeachstoneLLC/base-web.git
```

get /volume/git/environment.properties from the qat base-engine server, something like this (thought the instance id may change in the future)...
```
cx containers exec -s 'dw-engine' -e qat 7d4a03dec84c3011cc0472c3c9d3310d26d732312ffc1018df53bd7f984c1375 /bin/bash
cat the file and copy it to your clipboard; remove HOSTNAME, PWD, JAVA_HOME, HOME, _
you may also want to add HUBSPOT_ITERATION_LIMIT=2 to limit pagination loops
save it as $TSDEV_ROOT/dw/dw-environment.properties
```
TODO: figure out a better way to generate this file from source control

## fire up the docker instance
The easiest way to run all the necessary code is to launch a docker instance
```
cd $TSDEV_ROOT/dw/base-engine
docker rm -f dw-engine1 2>/dev/null || true
docker rmi -f dw-engine 2>/dev/null || true
docker build -t dw-engine .
docker run -v $TSDEV_ROOT/dw:/tsdev/dw --name dw-engine1 -d dw-engine
```

Open a console for the docker instance & build the collector.
```
docker exec -i -t dw-engine1 bash
export homeDir=/tsdev/dw/base-connectors
```

Build the MetaModelCollector
```
cd $homeDir/collect/MetaModelCollector
mvn clean package
tar -xzvf target/*.gz
mkdir -p $homeDir/lib
cp metamodel/*.jar $homeDir/lib
```

Build the DataLoader
```
cd $homeDir/collect/DataLoader
mvn clean package
tar -xzvf target/*.gz
cp data*/*.jar $homeDir/lib
```

Here's a one-liner that encapsulates the DataLoader build steps above for when you are building repeatedly to test changes.
```
cd $homeDir/collect/DataLoader && mvn clean package && tar -xzvf target/*.gz && cp data*/*.jar $homeDir/lib
```

Run the DataLoader end-to-end (not sure this works--I ran single feed jobs instead as described later)
```
## TODO: commit these files with +x already set so we don't have to do this
# find . -name '*.sh' | xargs chmod +x
# bash -x collect/start.sh -d properties/source-map.properties -e ../dw-environment.properties  -s ../out
```

Run the DataLoader for a single feed
```
# note: substitute different sources from $homeDir/collect/properties/source-map.properties in place
# of 'hubspot' and 'contacts'
```
cd $homeDir
mkdir -p ../out

# repeat after each edit of DataLoader code
cd $homeDir/collect/DataLoader && mvn clean package && tar -xzvf target/*.gz && cp data*/*.jar $homeDir/lib && cd $homeDir

# here's the actual run
java -cp "./lib/*" com.teachstone.dw.dataload.hubspot.HubspotLoader hubspot contacts ../dw-environment.properties ../out/

# review for expected results
wc -l ../out/hubspot.contacts.csv

Note that some feeds are large it can take a while to paginate through them all, and we also
have a throttle limiting how many calls we can make in one day, so consider adding
HUBSPOT_ITERATION_LIMIT to your properties file to enable the iteration limit if you are doing a lot of edits/tests.

# Local Docker Setup for Development, testing and troubleshooting
### Development environment setup
Install IDE (InteliJ)
Install JDK
Install Maven to compile from command line
Install git
Checkout the projects from github
Open the base-connector project from InteliJ and ensure Java modules imported as Maven projects
The Web Services project may need WSLD compiled before the project is usable without errors
Install Docker

### Local Docker setup
Build the Docker image of base-engine using the command
Run the docker container using the command
Login to the docker container
Copy the cloud 66 qat environment variables to a environment.properties file to source the environment properties. In future, if the DEV environment becomes available, then DEV  cloud 66 environment properties should be copied and used.
Copy private key file to the running container. Private key file path need t be provided as input while executing the process using ssh option


### SSH vs Non-SSH
Base-connectors, application works on AWS along with trusted hosts. (DB hosts, Heroku/Salesforce etc). Base connectors typically need host name, userid, password there. However on local environment, the access to database is restricted with SSH tunnel. This means, the shell scripts and Java code has to be able to distinguish the connection based on the parameters supplied and use approrpiate low level library to gain database access.

### SSH instructions
Need teachstone email address and password
Use "Teachstone VPN Connections - setup" document to set up VPN connection
Setup SSH Key locally
Coordinate with DEV manager to register your key in SSH host machine
In order to access the cloud 66 qat DB from local, need to get SSH host name and port number. Also need to note down the SSH forward routing (eg. mySQL port 3607 forward to 3607 on local host/docker)
Use the ssh key, uid/pwd and try to access the DB from Database tool (like Mysql work bench, HeidiSQL)  
Modify the scripts of interest (eg. generate_rdbms_models.sh) to provide SSH key, SSH host name etc. parameters.

### Processing on local docker-setup
Copy newly generated jars to local docker container. Login to docker container and start required process.

### Test using Non-SSH option - Used for production execution
pass parameters like below. Chage $MYSQL_HOST, $MYSQL_USERID and $MYSQL_PWD based on database connection

-d $RDBMS_MODEL_STAGING_DIRECTORY -h $MYSQL_HOST -p 3306 -u $MYSQL_USERID -P $MYSQL_PWD

Process is expected fail, if database connectivity is established using VPN,  

### Test using SSH option - Used only for development
pass parameters like below

-d $RDBMS_MODEL_STAGING_DIRECTORY -s ssh -h $MYSQL_HOST -p 3306 -u $MYSQL_USERID -P $MYSQL_PWD -H bastion.teachstone.com -S 22 -U "ssh user" -K "Private key file location" -C "SSH forward routing"

process is expected to complete successfully
