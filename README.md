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

## Local Docker Setup for Development, testing and troubleshooting
### Development environment setup
* Install IDE (InteliJ)
* Install JDK
* Install Maven to compile from command line
* Install git
* Checkout the projects from github
* Open the base-connector project from InteliJ and ensure Java modules imported as Maven projects
* The Web Services project may need WSLD compiled before the project is usable without errors
* Install Docker

### Local Docker setup
* Build the Docker image of base-engine - Refer 1 to 4 lines from "fire up the docker instance" section to create image
* Run the docker container - Refer 5th line from "fire up the docker instance" section to run container 
* Login to docker container

	```
	docker exec -i -t dw-engine1 bash
	```

* Copy the cloud 66 qat environment variables to a environment.properties file to source the environment properties. In future, if the DEV environment becomes available, then DEV  cloud 66 environment properties should be copied and used.
* Copy private key file to the running container. Private key file path need to be provided as input while executing the process using ssh option
 
	```
	docker cp 'private key file' dw-engine1:/local/git
	```

### SSH vs Non-SSH
Base-connectors, application works on AWS along with trusted hosts. (DB hosts, Heroku/Salesforce etc). Base connectors typically need database host name, userid, password. However on local environment, the access to database is restricted with SSH tunnel. This means, the shell scripts and Java code has to be able to distinguish the connection based on the parameters supplied and use appropriate low level library to gain database access.

### SSH instructions
* Need teachstone email address and password
* Use "Teachstone VPN Connections - setup" document to set up VPN connection
* Use ssh-genkey or PuTTYgen to generate SSH keys (public key and private key) for locally
* Coordinate with DEV manager to register ssh public key in SSH host machine
* In order to access the cloud 66 qat DB from local, need to get SSH host name, port number, userid and password. Also need to note down the SSH forward routing (eg. mySQL port 3607 forward to 3607 on local host/docker)
* Use the ssh key, uid/pwd and try to access the DB from Database tool (like Mysql work bench, HeidiSQL)
* Login to docker container

	```
	docker exec -i -t dw-engine1 bash
	```

* Modify the scripts of interest (eg. generate_rdbms_models.sh) to provide SSH key, SSH host name etc. parameters. Example
	```
	java -cp "$homeDir/lib/*" com.anant.teachstone.tools.metamodel.processor.MySqlModelProcessor -d $RDBMS_MODEL_STAGING_DIRECTORY -s ssh -h $MYSQL_HOST -p 3306 -u $MYSQL_USERID -P $MYSQL_PWD -H "ssh-host" -S "ssh-port" -U "ssh-user" -K /local/git/id_rsa -C 3306
	```

### Executing model generator processing on local docker-setup
* First check process is executing successfully on local machine. 

	Open the base-connector project from InteliJ IDE

	Select MySqlModelProcessor as main class for processing

	Provided below parameters 
	```
	-d Staging directory
	-s ssh
	-h database host name
	-p database port 
	-u database userid
	-P database password
	-H ssh host
	-S ssh port
	-U ssh userid
	-K private key file path
	-C ssh-forward-port
	```
	Run the process

	On successful completetion mysql.model.csv file is available under "staging directory"/rdbms_model folder  

* Build project using Maven. metamodel-bin.zip file will be created in target folder
* unzip metamodel-bin.zip file into metamodel-bin directory under target folder 
* Start dw-engine container. 
* Copy newly generated jars to local docker container.
	```
	docker cp $TSDEV_ROOT/dw/base-connectors/collect/MetaModelCollector/target/metamodel-bin/metamodel/metamodel.jar dw-engine1:/volume/git/base-connectors/lib/
	docker cp $TSDEV_ROOT/dw/base-connectors/collect/MetaModelCollector/target/metamodel-bin/metamodel-1.0-SNAPSHOT.jar dw-engine1:/volume/git/base-connectors/lib/  
	```

	In case new library is added, copy corresponding jars
  
* Login to docker container
* Model generation from RDBMS writes to S3 bucket. Comment corresponding line generate_models_from_rdbms.sh so that testing is limited to local
* Move to base-collector folder
	```
	cd /volume/git/base-collectors
	```
* Edit dw-run-all-job.sh file. Retain only "Model generation from RDBMS" (generate_models_from_rdbms.sh) process. Comment out rest of the process.
* Start model generation process.
	```
	./dw-run-all-job.sh
	```
* review for expected results

	```
	wc -l /opt/base/staging/qat/YYYY/MM/DD/rdbms_model/mysql.model.csv
	```