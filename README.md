# Apache NiFi - FDLC
Project name: Apache NiFi - Flow Development Life Cycle
<!---
> Outline a brief description of your project.
> Live demo [_here_](https://www.example.com). If you have the project hosted somewhere, include the link here. -->


## Table of Contents
* [Technologies Used](#technologies-used)
* [General Information](#general-information)
* [Setup](#setup)
    * [Local](#local)
    * [Jenkins](#jenkins)
* [Deploy](#deploy)
* [Room for Improvement](#room-for-improvement)
<!-- * [License](#license) -->

## Technologies Used
- Apache NiFi - version 1.18.0
- Apache NiFi Registry - version 1.18.0
- ... - version x.y.z


## General Information
The purpose of this Apache NiFi project is to implement a Flow Development Life Cycle from the local environment to the development environment deployed on OpenShift platform.  
The flow is as follows (below are the details about each step): 
1. The front-end will send a request towards the gnc-proxy microservice 
2. The gnc-proxy microservice will check if the user who made the request has the proper permission
3. The gnc-proxy microservice will request a new token to ADFS
4. The gnc-proxy microservice will forward the request coming from the front-end to NGINX
5. NGINX will check the validity of the token and forward the request to the ML service originally requested by the front-end.
6. The front-end will recive a response from the microservice

### Apache NiFi Basic Concept
The requests coming from the front-end will be POST requests and inside the body it will be specified:
* _flow_: the service to contact depending on the use case
* _processor_: the endpoint to forward the request to (multiple endpoints will be available for each service)
* _processor group_: the request method to send to the endpoint (get, post, etc.)
* _input_: any parameters to pass as input to the machine learning model (parameter names must be specified UPPER CASE)
An example of a request body is as follows:  
```
{
    "resource": "1740-d/gnc-uc-1-inference-real-time",
    "endpoint": "UC01/corr_gnc",
    "scope": "post",
    "input": {
        "START_DATE":"2020-01-01",
        "END_DATE":"2020-12-31",
        "MODEL_ID":1
    }
}
```

### User Profile Verification
The POST requests coming from the front-end will contain a JWT that must be verified before proceeding with the next steps. The microservice will have to extract the user ID associated with the token, then contact the _anagraficaUtenti_ service ([https://summer-gs.snam.it/editable-fields-service/anagraficaUtenti/getProfiloUtente/{user_id}](https://summer-gs.snam.it/editable-fields-service/anagraficaUtenti/getProfiloUtente/{user_id})) and retrieve the user profile. Once the user profile has been extracted, it sends a request to the _editable_ API ([https://summer-gs.snam.it/editable-fields-service/editable/{user_profile}](https://summer-gs.snam.it/editable-fields-service/editable/{user_profile})). If the list of features assigned to the user profile contains the label identifying the user group that has access to ML services (e.g. "_GNC_") it will proceed with the next steps, otherwise it will return a 401 code (unauthorized user).    


### Request for a new token to ADFS
NGINX is federated with the Single Sign-On system AD FS (Active Directory Federation Services), therefore the service authentication will take place through the AD FS of Snam (identity provider that manages all applications with SSO authentication).  
Being a machine-to-machine scenario, the authentication type is OpenID Connect OAuth 2.0, and the flow type is Client Credential (Client ID and Secret).  
The service will request an access token from the AD FS, which will be included in the call addressed to the services deployed on Azure Red Hat OpenShift. 
POST requests for the generation of a new token are sent to [https://authlgn.snam.it/adfs/oauth2/token](https://authlgn.snam.it/adfs/oauth2/token) and must contain:  
- *client_id*: provided by AM SSO. 
- *client_secret*: provided by the AM SSO. 
- *grant_type*: Must be set to _client_credentials_. 
- *resource*: Indicate the identifier of the API to be accessed. 
- *scope*: Indicate the grants to apply for the API to be accessed. 

**Note**: the *client_id* and *client_secret* were stored as OpenShift secret and then defined as environment variables within the pod on which the service runs. 


### Send request to NGINX
The response obtained from ADFS returns the parameters: 
- *access_token* 
- *token_type*
- *expires_in* 

The access token is then used to send a new POST request to NGINX which will contains, as body, only the "input" section of the request coming from the frontend. 
Upon receipt of the request from the service, NGINX will extract the JWT and once the validity is checked will forward the request to the specified endpoint.

For further details about the flow and some examples of calls, please refer to the document validated by the Security by Design team.  


## Setup
<!---
What are the project requirements/dependencies? Where are they listed? A requirements.txt or a Pipfile.lock file perhaps? Where is it located?

Proceed to describe how to install / setup one's local environment / get started with the project.
-->

The following instructions will guide you to install Apache NiFi and Apache NiFi Registry. 

### Local

#### Preconditions: Java
On Ubuntu systems proceed as follows: 
1. Download and install openjdk-8-jdk:
    ```console
    sudo apt-get install openjdk-8-jdk
    ```
2. Verify that the installation was successful: 
    ```console
    java -version
    ```
3. Retrive the Java installation path: 
    ```console
    readlink -f $(which java)
    ```
4. Edit the _~/.bashrc_ file by adding the following line with the path returned by the previous command, for example: 
    ```console
    export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
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

#### Preconditions: Git Repository
The following instructions are based on using Ubuntu as the system and GitHub as the hosting service of the Git repository. In this case, proceed as follows: 
1. Navigate to your GitHub profile and create a new repository
2. Clone the repository
3. Go back to the developer setting in GitHub and create a new personal access token 
4. Copy the generated token 
5. Go back to the Apache NiFi Registry installation directory
6. Edit the _conf/providers.xml_ by changing the _flowPersistenceProvider_ class to the _GitFlowPersistenceProvider_ and set it to point the cloned repository in the following way:       
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
        
        <flowPersistenceProvider>
            <class>org.apache.nifi.registry.provider.flow.git.GitFlowPersistenceProvider</class>
            <property name="Flow Storage Directory">/mnt/c/Users/febarusco/VSCode/apache-nifi-fdlc/flow_storage</property>
            <property name="Remote To Push">origin</property>
            <property name="Remote Access User">fbarusco@gmail.com</property>
            <property name="Remote Access Password">PERSONAL_ACCESS_TOKEN_HERE</property>
            <property name="Remote Clone Repository"></property>
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
7. start NiFi registry:
    ```console
    sudo bin/nifi-registry.sh start
    ``` 
4. To monitor the application log execute the following command: 
    ```console
    tail -f ../logs/nifi-registry-app.log
    ```
6. Wait for NiFi Registry to start (this may take some time)
8. Access the Apache NiFi Registry UI from the browser at the following link: http://127.0.0.1:18080/nifi-registry
9. Navigate to Settings to create a new bucket 
10. Start up the NiFi by moving to the _nifi-1.18.0/bin_ folder and executing the following command: 
    ```console
    sudo ./nifi.sh start
    ```
11. Access the Apache NiFi UI from the browser at the following link: https://127.0.0.1:8443/nifi/
12. Register the new registry client by navigating to the Controller Settings from the global menu. 
13. Set the _URL_ registry client property by pointing to the following URL: http://127.0.0.1:18080/
14. From the NiFi canvas, drop a new Processor Group and 'Start version control' from the menu
-  

Se hai precedentemente fermato NiFi e hai dimenticato il nome utente e la password generata automaticamente, la puoi recuperare dai log generati al momento dell'installazione, ad esempio: 
    ```console
    cat logs/nifi-app_2022-10-27_17.0.log | grep 'Generated'
    ```
E' comunque consigliato di generare le proprie password nel seguente modo..

More details: https://nifi.apache.org/docs/nifi-registry-docs/html/administration-guide.html#gitflowpersistenceprovider 
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
3. Move to the _nifi-1.18.0/bin_ folder and edit the _nifi-env.sh_ script by adding the following ling:
    ```console
    export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/
    ```
4. From the same folder, start NiFi: 
    ```console
    sudo ./nifi.sh start
    ```
5. To monitor the application log execute the following command: 
    ```console
    tail -f ../logs/nifi-app.log
    ```
6. Wait for NiFi to start (this may take several minutes)
7. Retrieve the username and password automatically generated by NiFi: 
    ```console
    cat ../logs/nifi-app.log | grep 'Generated Username'
    cat ../logs/nifi-app.log | grep 'Generated Password'
    ```
8. Access the Apache NiFi UI from the browser at the following link: https://127.0.0.1:8443/nifi/

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
3. Move to the _nifi-registry-1.18.0/bin_ folder and start NiFi registry:
    ```console
    sudo ./nifi-registry.sh start
    ``` 
4. To monitor the application log execute the following command: 
    ```console
    tail -f ../logs/nifi-registry-app.log
    ```
6. Wait for NiFi Registry to start (this may take some time)
8. Access the Apache NiFi Registry UI from the browser at the following link: http://127.0.0.1:18080/nifi-registry

### Jenkins 
1. Login to [Jenkins](https://jenkins-1499-ci-cd.apps.ocp.snamretegas.priv/) with your credentials
2. Select the CI directory (blue dot: last build went well, grey: last build has been cancelled, red: last build failed; the W indicates the weather and represents the outcome of the last 5 builds)
3. Create the specification that links to the Git repository: 
    1. Select "New Item" from the menu on the left and assigning a name to the pipeline you want to create (e.g. gnc-proxy)
    2. Among the various options select: "Do not allow concurrent builds", "Discard old builds" assigning "Max # of builds to keep" equal to 5. 
    3. Among the pipeline options select "Pipeline script from SCM" as definition, then "Git" as SCM, paste the BitBucket repository address within the "Repository URL" field, as credentials those associated with the Jenkins user for DevOps tools, specify the branch and if the Jenkins file is simply named "Jenkinsfile", leave the option "Script Path" with the default value. 
    4. Once saved Jenkins will create the new pipeline. 
4. Proceed with the build in Jenkins by selecting "Build Now" and monitor the execution by selecting "Console output". Once the build is successful, navigate to OpenShift and verify that the pod is up and running.

<!---
## Usage
How does one go about using it?
Provide various use cases and code examples here.

`write-your-code-here`
-->

## Deploy
Four enviroments are defined: dev, test (formazione), pre-prod, prod.  
To deploy on dev:
1) tag develop branch using semantic versioning followed by _svil suffix (ex. 1.0.15_svil). 
2) On [Jenkins](https://jenkins-1499-ci-cd.apps.ocp.snamretegas.priv/) select CI folder, gnc-proxy pipeline, execute compilation manually and the service is deployed in dev
 
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