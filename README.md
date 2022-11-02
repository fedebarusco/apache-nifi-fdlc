# Apache NiFi - FDLC
Project name: Apache NiFi - Flow Development Life Cycle
<!---
> Outline a brief description of your project.
> Live demo [_here_](https://www.example.com). If you have the project hosted somewhere, include the link here. -->


## Table of Contents
* [Technologies Used](#technologies-used)
* [General Information](#general-information)
    * [Apache NiFi Basic Concept](#apache-nifi-basic-concept)
* [Setup](#setup)
    * [Local](#local)
        * [Preconditions: Java](#preconditions-java)
        * [Apache NiFi Installation](#apache-nifi-installation)
        * [Apache NiFi Registry Installation](#apache-nifi-registry-installation)
        * [Apache NiFi Registry Setup](#apache-nifi-registry-setup)
    * [Jenkins](#jenkins)
* [Deploy](#deploy)
* [Room for Improvement](#room-for-improvement)
<!-- * [License](#license) -->

## Technologies Used
- Apache NiFi - version 1.18.0
- Apache NiFi Registry - version 1.18.0
- ... - version x.y.z


## General Information
The purpose of this Apache NiFi project is to implement a Flow Development Life Cycle from the local to the development environment deployed on OpenShift platform.  
The flow is as follows (below are the details about each step): 
1. The developer produce a new flow that will be pushed in a git repository 
2. Jenkins .. 

### Apache NiFi Basic Concept
Apache NiFi is a dataflow system based on the concepts of flow-based programming. It supports powerful and scalable directed graphs of data routing, transformation, and system mediation logic. NiFi has a web-based user interface for design, control, feedback, and monitoring of dataflows. 
The main components of an Apache NiFi dataflow are:
* _FlowFile_: Each piece of data that the user brings into NiFi for processing and distribution is referred to as a FlowFile. A FlowFile is made up of two parts: Content (the data itself) and Attributes, which are key-value pairs that are associated with the Content. 
* _Processor_: The Processor is the NiFi component that is used to listen for incoming data; pull data from external sources; publish data to external sources; and route, transform, or extract information from FlowFiles.
* _Process Group_: When a dataflow becomes complex, it often is beneficial to reason about the dataflow at a higher, more abstract level. NiFi allows multiple components, such as Processors, to be grouped together into a Process Group. 
* _Relationship_: Each Processor has zero or more Relationships defined for it. These Relationships are named to indicate the result of processing a FlowFile. After a Processor has finished processing a FlowFile, it will route (or “transfer”) the FlowFile to one of the Relationships.
* _Controller Service_: Controller Services are extension points that, after being added and configured by a DataFlow Manager (DFM) in the User Interface, will start up when NiFi starts up and provide information for use by other components (such as processors or other controller services). For instance, the StandardSSLContextService provides the ability to configure keystore and/or truststore properties once and reuse that configuration throughout the application. The idea is that, rather than configure this information in every processor that might need it, the controller service provides it for any processor to use as needed.

For more details about Apache NiFi terminology, refer to the official [User Guide](https://nifi.apache.org/docs/nifi-docs/html/user-guide.html#terminology).

## Setup
<!---
What are the project requirements/dependencies? Where are they listed? A requirements.txt or a Pipfile.lock file perhaps? Where is it located?

Proceed to describe how to install / setup one's local environment / get started with the project.
-->

The following instructions will install Apache NiFi 1.18.0 and Apache NiFi Registry 1.18.0 and will configure the Registry to store the flow in a git repository. 

### Local

The following instructions are based on Ubuntu 22.04.1 LTS operating system and GitHub as the hosting service of the git repository.

#### Preconditions: Java

On Ubuntu systems proceed as follows: 
1. Download and install openjdk-11-jre:
    ```console
    sudo apt-get install openjdk-11-jre
    ```
2. Verify that the installation was successful: 
    ```console
    java -version
    ```
3. Retrive the Java installation path: 
    ```console
    readlink -f $(which java)
    ```
4. Edit the _~/.bashrc_ file by adding the following line with the path returned by the previous command, for instance: 
    ```console
    export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
    ```
5. Read and execute the content of the  _~/.bashrc_ edited file: 
    ```console
    source ~/.bashrc
    ```
6. Verify that the _JAVA_HOME_ variable has been set correctly:
    ```console
    echo $JAVA_HOME
    ```
7. Reboot the system to terminate the Java setup procedure

#### Apache NiFi Installation

On Ubuntu systems proceed as follows: 
1. Download the package as follows:
    ```console
    wget https://archive.apache.org/dist/nifi/1.18.0/nifi-1.18.0-bin.zip
    ```
2. Unzip the downloaded package:
    ```console
    unzip nifi-1.18.0-bin.zip
    ```
3. Move to the _nifi-1.18.0/bin_ folder and edit the _nifi-env.sh_ script by adding the following line:
    ```console
    export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64/
    ```

#### Apache NiFi Registry Installation

On Ubuntu systems proceed as follows: 
1. Download the package as follows:
    ```console
    wget https://archive.apache.org/dist/nifi/1.18.0/nifi-registry-1.18.0-bin.zip
    ```
2. Unzip the downloaded package:
    ```console
    unzip nifi-registry-1.18.0-bin.zip
    ```
3. Move to the _nifi-registry-1.18.0/bin_ folder and edit the _nifi-registry-env.sh_ script by adding the following line:
    ```console
    export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
    ```

#### Apache NiFi Registry Setup

Proceed as follows: 
1. Navigate to the GitHub profile and create a new repository
2. Go to the developer setting in GitHub and create a new personal access token 
3. Copy the generated token
4. Go back to the local machine and from the root folder _nifi-registry-1.18.0_ of Apache NiFi Registry, clone the repository:
    ```console
    git clone https://github.com/fedebarusco/apache-nifi-fdlc.git
    ``` 
5. Edit the _conf/providers.xml_ by changing the _flowPersistenceProvider_ class to the _GitFlowPersistenceProvider_ and set it to point the cloned repository, for instance:       
    ```xml
    <?xml version="1.0" encoding="UTF-8" standalone="yes"?>
    ...
    <providers>

        <!--
        ...
        -->

        <!--
        <flowPersistenceProvider>
            <class>org.apache.nifi.registry.provider.flow.FileSystemFlowPersistenceProvider</class>
            <property name="Flow Storage Directory">./flow_storage</property>
        </flowPersistenceProvider>
        -->
        
        <flowPersistenceProvider>n
            <class>org.apache.nifi.registry.provider.flow.git.GitFlowPersistenceProvider</class>
            <property name="Flow Storage Directory">./apache-nifi-fdlc</property>
            <property name="Remote To Push">origin</property>
            <property name="Remote Access User">fedebarusco</property>
            <property name="Remote Access Password">YOUR_PERSONAL_ACCESS_TOKEN</property>
            <property name="Remote Clone Repository">https://github.com/fedebarusco/apache-nifi-fdlc.git</property>
        </flowPersistenceProvider>
        
        <!--
        <flowPersistenceProvider>
            <class>org.apache.nifi.registry.provider.flow.DatabaseFlowPersistenceProvider</class>
        </flowPersistenceProvider>
        -->

        <!--
        <eventHookProvider>
            ...
        </eventHookProvider>
        -->

        <extensionBundlePersistenceProvider>
            <class>org.apache.nifi.registry.provider.extension.FileSystemBundlePersistenceProvider</class>
            <property name="Extension Bundle Storage Directory">./extension_bundles</property>
        </extensionBundlePersistenceProvider>

        <!--
        <extensionBundlePersistenceProvider>
            ...
        </extensionBundlePersistenceProvider>
        -->

    </providers>
    ```
6. Move to the _nifi-registry-1.18.0/bin_ folder and start NiFi registry in the background:
    ```console
    sudo ./nifi-registry.sh start
    ``` 
7. To monitor the application log execute the following command: 
    ```console
    tail -f ../logs/nifi-registry-app.log
    ```
8. Wait for NiFi Registry to start (this may take some time)
9. Access the Apache NiFi Registry UI from the browser at the following link: [http://127.0.0.1:18080/nifi-registry](http://127.0.0.1:18080/nifi-registry) 
10. Navigate to _Settings_ and create a new bucket 
11. Start up the NiFi in the background by moving to the _nifi-1.18.0/bin_ folder and executing the following command: 
    ```console
    sudo ./nifi.sh start
    ```
12. To monitor the application log execute the following command: 
    ```console
    tail -f ../logs/nifi-app.log
    ```
13. Wait for NiFi to start (this may take several minutes)
14. Retrieve the username and password automatically generated by NiFi: 
    ```console
    cat ../logs/nifi-app.log | grep 'Generated Username'
    cat ../logs/nifi-app.log | grep 'Generated Password'
    ```
15. Access the Apache NiFi UI from the browser at the following link: [https://127.0.0.1:8443/nifi/](https://127.0.0.1:8443/nifi/)
16. Register the new registry client by navigating to the _Controller Settings_ from the global menu 
17. Set the _URL_ registry client property by pointing to the following URL: [http://127.0.0.1:18080/](http://127.0.0.1:18080/)
18. From the NiFi canvas, drop a new Processor Group and _Start version control_ from the menu
19. Go back to the previously created repository and verify that a new folder, representing the bucket, has been created 

For more details: [Apache NiFi Gitflow Persistence provider](https://nifi.apache.org/docs/nifi-registry-docs/html/administration-guide.html#gitflowpersistenceprovider) 

### Jenkins

<!---
## Usage
How does one go about using it?
Provide various use cases and code examples here.

`write-your-code-here`
-->

On Ubuntu systems proceed as follows: 
1. Download the package as follows:
    ```console
    curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
    ```
    ```console
    echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
    ```
    ```console
    sudo apt-get update && sudo apt-get install jenkins
    ```
2. Verify that the installation was successful: 
    ```console
    jenkins --version
    ```
3. Enable the Jenkins service to start at boot:
    ```console
    sudo systemctl enable jenkins
    ```
4. start the Jenkins service with the command:
    ```console
    sudo systemctl start jenkins
    ```
5. check the status of the Jenkins service with the following command: 
    ```console
    sudo systemctl status jenkins
    ```
6. Retrieve the initial admin password with the following command: 
    ```console
    sudo cat /var/lib/jenkins/secrets/initialAdminPassword
    ```
7. Browse to [http://localhost:8080](http://localhost:8080) (or whichever configured port during Jenkins installation) and complete the initial setup with the plugins installation and the creation of the first administrator user. 

*Notes*: For WSL2 users, instead of using the ```systemctl``` command, simply use ```service```, for instance: ```sudo service jenkins start```

## Deploy
Three enviroments are defined: development, staging, production.  
To deploy on devevelpment environment proceed as follows:
1) Tag develop branch using semantic versioning 
2) On [Jenkins](https://www.jenkins.io/) ..
 
## Room for Improvement
- Add healthcheck 
<!---
Include areas you believe need improvement / could be improved. Also add TODOs for future development.

Room for improvement:
- Improvement to be done 1
- Improvement to be done 2

To do:
- Feature to be added 1
- Feature to be added 2
-->

<!-- Optional -->
<!-- ## License -->
<!-- This project is open source and available under the [... License](). -->