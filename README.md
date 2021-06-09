# Creating an IBM MQ Channel Between IBM Cloud Pak for Integration on IBM Z and Traditional z/OS

## Prerequisites

1. OpenShift 4.6.x installed
2. OpenShift command line (oc) installed
3. Cloud Pak for Integration 2020.3.1 installed
4. IBM MQ on z/OS Installed
5. Queue manager, transmission queue, and listener all configured on MQ on z/OS with TLS 1.2 or higher.
6. MQ Explorer installed on your local workstation

## Purpose of Demonstration

This demonstration will show how easily one can install IBM MQ on Cloud Pak for Integration and configure a channel to communicate with IBM MQ running on traditional z/OS. This demonstration assumes that Cloud Pak for Integration is installed on Red Hat OpenShift, and that IBM MQ is installed and configured on z/OS.

## Demo Steps

### Install IBM MQ on Cloud Pak for Integration

IBM MQ on Cloud Pak for Integration is delivered as an operator and can be installed through the OpenShift OperatorHub.

1. Log into OpenShift. Ensure that your credentials allow for the installation of operators into your project of choice.
1. Navigate to Operators -> OperatorHub -> search for IBM MQ and ensure that you are in the correct project where you want IBM MQ to be deployed.
![OperatorHub Search](https://github.com/mmondics/mq-cp4int-zos/blob/main/Images/2021-06-07-10-37-34.png)

    *Note: If you do not see IBM MQ in the search results, you might not have the correct CatalogSource instsalled on your OpenShift cluster. It can be created with the following YAML file.*

    ```YAML
    apiVersion: operators.coreos.com/v1alpha1
    kind: CatalogSource
    metadata:
    name: ibm-operator-catalog
    namespace: openshift-marketplace
    spec:
    displayName: ibm-operator-catalog 
    publisher: IBM Content
    sourceType: grpc
    image: docker.io/ibmcom/ibm-operator-catalog
    updateStrategy:
        registryPoll:
        interval: 45m
    ```

1. Click on the IBM MQ tile and click the Install button.
1. Select the update channel that is supported by your version of Cloud Pak for Integration and OpenShift. [Refer here](https://www.ibm.com/docs/en/ibm-mq/9.2?topic=containers-version-support-mq-operator)
1. Choose your installation mode and approval strategy based on your cluster and preferences.
1. Click Install.

Wait for the operator to successfully install.
![Operator Installed](https://github.com/mmondics/mq-cp4int-zos/blob/main/Images/2021-06-07-10-45-45.png)

You now have the ability to create a containerized queue manager on Cloud Pak for Integration.

### Create a Queue Manager from YAML file

1. In the OpenShift console, copy your login command by clicking on your username in the top right corner -> Copy Login Command -> enter your credentials -> Display Token -> copy the `oc login` command.
1. In a terminal session where the `oc` commmand line is installed, paste the `oc login` command and press enter.
1. Create and edit a file named externalqmgr.yaml with the command

    `vi externalqmgr.yaml`
1. Paste the following YAML file that will create a ConfigMap, Secret, QueueManager, and Route.

    ````YAML
    kind: ConfigMap
    apiVersion: v1
    metadata:
    name: mqexternalmqsc
    namespace: <YOUR_PROJECT>
    data:
    mqexternal.mqsc: |-
        define ql(APPQ)
        DEFINE CHANNEL('<YOUR_CHANNEL_NAME>') CHLTYPE(RCVR) TRPTYPE(TCP) SSLCAUTH(OPTIONAL) SSLCIPH('ANY_TLS12_OR_HIGHER') 
        set chlauth('<YOUR_CHANNEL_NAME>') TYPE(BLOCKUSER) USERLIST(NOBODY)
        REFRESH SECURITY TYPE(CONNAUTH)
    ---
    kind: Secret
    apiVersion: v1
    metadata:
    name: mqexternalqmgrcert
    namespace: <YOUR_PROJECT>
    data:
    tls.crt: <YOUR_TLS_CRT>
    tls.key: <YOUR_TLS_KEY>
    type: Opaque
    ---
    apiVersion: mq.ibm.com/v1beta1
    kind: QueueManager
    metadata:
    name: externalmq
    namespace: <YOUR_PROJECT>
    spec:
    version: <MQ_VERSION>
    license:
        accept: true
        license: <MQ_LICENSE>
        use: "NonProduction"
    pki:
        keys:
        - name: default
        secret:
            secretName: mqexternalqmgrcert
            items:
            - tls.key
            - tls.crt
    web:
        enabled: true
    queueManager:
        type: ephemeral
        mqsc:
        - configMap:
            name: mqexternalmqsc
            items:
                - mqexternal.mqsc
    template:
        pod:
        containers:
            - env:
                - name: MQSNOAUT
                value: 'yes'
            name: qmgr
    ---
    kind: Route
    apiVersion: route.openshift.io/v1
    metadata:
    name: mq-traffic-mq-externalqmgr-ibm-mq-qm
    namespace: <YOUR_PROJECT>
    spec:
    host: '<SNI_ADDRESS_MAPPING_FOR_THE_CHANNEL>'
    to:
        kind: Service
        name: externalmq-ibm-mq
    port:
        targetPort: 1414
    tls:
        termination: passthrough
    wildcardPolicy: None
    ```

1. Modify the YAML file to suit your environment. There are various portions of the YAML file above that need to be modified:
    * <YOUR_PROJECT> : Change to the name of your project where the queue manager will be deployed
    * <YOUR_CHANNEL_NAME> : Change to the receiver channel name. This should match the IBM MQ sender channel on the z/OS system.
    * <YOUR_TLS_CRT> : Change to the base64 encoded tls.crt that matches the tls.crt of the IBM MQ on z/OS queue manager.
    * <YOUR_TLS_KEY> : Change to the base64 encoded tls.key that matches the tls.key of the IBM MQ on z/OS queue manager.
    * <MQ_VERSION> : Change to the IBM MQ version and ensure that it matches the IBM MQ update channel you selected earlier and is supported with your CP4Int and OpenShift versions. [Refer here](https://www.ibm.com/docs/en/ibm-mq/9.2?topic=containers-version-support-mq-operator).
    * <MQ_LICENSE> : Change to the IBM MQ license that supports your MQ_VERSION and Cloud Pak for Integration version. [Refer Here](https://www.ibm.com/docs/en/ibm-mq/9.2?topic=mqibmcomv1beta1-licensing-reference).
    * <SNI_ADDRESS_MAPPING_FOR_THE_CHANNEL> : You must map the channel name to the SNI address. [Refer here](https://www.ibm.com/docs/en/ibm-mq/9.1?topic=containers-connecting-queue-manager-deployed-in-openshift-cluster).

1. Create the objects defined in the YAML file with the command:

    `oc create -f externalqmgr.yaml`

1. Change into the project you specified in the YAML file with the command:

    `oc project <YOUR_PROJECT>`

1. Check if the queue manager named externalmq was successfully created with the command:

    `oc get qmgr`

    ```txt
    NAME         PHASE
    externalmq   Running
    ```

    *Note: If your queue manager does not have the Phase: Running after about a minute, there is likely something wrong with your YAML file and will require some debugging. If this is the case, you can start by looking at the events provided by `oc describe qmgr/externalmq`. You can also check the qmgr pod logs by looking at `oc logs pod/externalmq-ibm-mq-0`*

1. Once you see your queue manager is running correctly, you can look at the pod it created with the command:

    `oc get pods`

    ```txt
    oc get pods
    NAME                         READY   STATUS    RESTARTS   AGE
    externalmq-ibm-mq-0          1/1     Running   0          27m
    ```

1. You can look at the IBM MQ pod logs with the command:

    `oc logs pod/externalmq-ibm-mq-0`

    *If you have worked with IBM MQ on other platforms in the past, you will notice that the logs look the same as the IBM MQ startup messages on other platforms and architectures.*

    *You will also notice from the pod logs that the IBM MQ instance started up with the parameters specified in the ConfigMap portion of the YAML file.*

If your Queue Manager and Pod are *Running* and you do not see any concerning error messages in the Pod logs, you can move on and explore what you just created from the Cloud Pak for Integration console.

### Explore the Queue Manager from the Cloud Pak for Integration Console

1. Navigate to your Cloud Pak for Integration console. You can find the address from the command line by entering:

    `oc get route -n <PROJECT_WHERE_CP4INT_IS_INSTALLED> -o custom-columns=NAME:metadata.name,HOST:spec.host`

    ```txt
    NAME                                   HOST
    externalmq-ibm-mq-qm                   externalmq-ibm-mq-qm-atg-cp4i.apps.atsocpd1.dmz
    externalmq-ibm-mq-web                  externalmq-ibm-mq-web-atg-cp4i.apps.atsocpd1.dmz
    integration-navigator-pn               integration-navigator-pn-atg-cp4i.apps.atsocpd1.dmz
    mq-traffic-mq-externalqmgr-ibm-mq-qm   icp12e-74-6f-2e-61-74-73-6f-63-70-64-1.chl.mq.ibm.com
    ```

    Copy the HOST associated with the integration-navigator-pn route - in my case, it is integration-navigator-pn-atg-cp4i.apps.atsocpd1.dmz.

    Navigate to `http://<YOUR_HOST>` in a web browser.

    ![CP4Int Console Login](https://github.com/mmondics/mq-cp4int-zos/blob/main/Images/2021-06-07-17-13-58.png)
1. Log in with your OpenShift credentials.
1. Click on the *Runtimes* tab and you will see the Queue Manager named externalmq that you created in the command line.
    ![CP4Int Runtimes](https://github.com/mmondics/mq-cp4int-zos/blob/main/Images/2021-06-07-17-17-35.png)
1. Click on the *externalmq* link to open the dashboard for your IBM MQ instance.
    *Note: you may need to log in with your OpenShift credentials again.*
    ![IBM MQ Dashboard](https://github.com/mmondics/mq-cp4int-zos/blob/main/Images/2021-06-08-10-16-53.png)
1. Click on the *Manage externalmq* tile on the left side of the page.
    ![Manage externalmq](https://github.com/mmondics/mq-cp4int-zos/blob/main/Images/2021-06-08-10-26-32.png)
    On this page, you can see the local queue named APPQ, which was defined in the ConfigMap portion of the YAML file. There are no messages in this queue yet.

    ```YAML
    define ql(APPQ)
    ```

1. Click on the *Communication* tab to the right of Queues, and then click on the *Queue Manager Channels* in the left side menu.
    ![qmgr channels](https://github.com/mmondics/mq-cp4int-zos/blob/main/Images/2021-06-08-10-34-57.png)
    On this page, you can see the Receiver channel that you defined in the ConfigMap portion of the YAML file:

    ```YAML
    DEFINE CHANNEL('<YOUR_CHANNEL_NAME>') CHLTYPE(RCVR) TRPTYPE(TCP) SSLCAUTH(OPTIONAL) SSLCIPH('ANY_TLS12_OR_HIGHER') 
    ```

With the queue manager, local queue, listener and receiver channel configured, we're now ready to start our z/OS queue manager and start sending messages to the APPQ local queue on Cloud Pak for Integration.

### Put Test Message from IBM MQ Explorer

Open your IBM Q Explorer session and connect to the queue manager running on z/OS.

1. Create a sender channel by right clicking Channels -> New -> Sender Channel.
    ![New sender channel](https://github.com/mmondics/mq-cp4int-zos/blob/main/Images/2021-06-08-14-56-01.png)
1. Enter the following values for channel parameters:

    * Name: the same name you used for <YOUR_CHANNEL_NAME> in the YAML file.
    * Transmission protocol: TCP
    * Connection name: <YOUR_BASTION_IP>(443)
        * *You can find this by logging onto the OpenShift cluster's bastion node and running `ifconfig -a`, or by asking your system administrator*
    * Transmission queue: <YOUR_TRANSMISSION_QUEUE>
        * *Look at the Queues section of your z/OS queue manager, or ask your system administrator*
    * SSL Cipher Spec: <YOUR_SSL_CIPHER>
        * *This must be TLS 1.2 or higher*

1. Click Finish.

    After creation, the sender channel with have an *Inactive* status.

1. Right click on the sender channel and select *Start*.

    The channel should now have a *Running* status, and if you go back to the Cloud Pak for Integration IBM MQ console, the associated Receiver channel should also be *Running*.

    ![Start Sender](https://github.com/mmondics/mq-cp4int-zos/blob/main/Images/2021-06-08-15-38-38.png)

    ![Running Receiver](https://github.com/mmondics/mq-cp4int-zos/blob/main/Images/2021-06-08-15-40-38.png)

    ***IF this is not the case, stop here and debug the channel connection until both are Running.***

1. Create a remote queue from IBM MQ Explorer by right clicking Queues -> Remote Queue Definition
1. Enter the following values for remote queue parameters:

    * Name: a unique name of your choice
        * *e.g. ATS.REMOTE.QUEUE in my example*
    * Remote queue: APPQ
    * Remote queue manager: externalmq
    * Transmission queue: the same transmission queue <YOUR_TRANSMISSION_QUEUE> that you used in the channel configuration

1. Click *Finish*

    We are now able to test the connection between our sender and receiver channels and pass a message between them.

1. Right click on the remote queue you just created and select *Put Test Message*.

1. Type your message in for *Message data* nad press the *Put message* button, then click *Close*

    ![Put Test](https://github.com/mmondics/mq-cp4int-zos/blob/main/Images/2021-06-08-16-23-19.png)

1. Go back to the Cloud Pak for Integration IBM MQ console and open the APPQ local queue.

    ![APPQ](https://github.com/mmondics/mq-cp4int-zos/blob/main/Images/2021-06-08-16-25-00.png)

You will see your test message appear in the APPQ local queue, and you have successfull passed a message from an IBM MQ sender channel on z/OS to IBM MQ receiver channel on OpenShift via IBM Cloud Pak for Integration.

### Generate Messages from Java Application on z/OS (application not provided - demo only)

This channel setup does not need to be limited to putting test messages onto the queue from IBM MQ Explorer. You can develop various types of applications to send and/or receive messages and IBM MQ supports various languages such as Java, JMS, C++, .NET, and messaging APIs for Node.js, Ruby, Java, Python, and more. You can find more information [here](https://www.ibm.com/docs/en/ibm-mq/9.2?topic=mq-developing-applications).

For the sake of demonstration, I will use a Java application running on the same z/OS system where I have the queue manager running. The application will select a random famous quote and put it onto the que as a message.

If I submit a job from z/OS with the following JCL:

```jcl
000001 //ZOSMFIDI JOB NOTIFY=USER1,REGION=0M, USER=USER1, 
000002 // CLASS=A,MSGCLASS=H,MSGLEVEL=(1,1)               
000003 //EXPORT EXPORT SYMLIST=(*)                        
000004 // SET QUEUE='ATS.REMOTE.QUEUE'                    
000005 // SET IDENTITY='USER1'                            
000006 //AMS EXEC PUTPROC,COMMAND=PUTMSG                  
```

And then look back at the APPQ messages in the IBM MQ on Cloud Pak for Integration console, I see the generated messages.

![Quote Message](https://github.com/mmondics/mq-cp4int-zos/blob/main/Images/2021-06-08-22-19-26.png)

This is a very simple example of the types of application messages you can send to your containerized queue manager running on Cloud Pak for Integration. For more examples and sample applications, [refer here](https://www.ibm.com/docs/en/ibm-mq/9.2?topic=mq-developing-applications).
