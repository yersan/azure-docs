---
title: Deploy a Java application with Red Hat JBoss Enterprise Application Platform (EAP) on an Azure Red Hat OpenShift 4 cluster
description: Deploy a Java application with Red Hat JBoss Enterprise Application Platform on an Azure Red Hat OpenShift 4 cluster.
author: yborgess
ms.date: {@date}
ms.topic: article
keywords: java, jakartaee, javaee, microprofile, EAP, JBoss EAP, ARO, OpenShift, JBoss Enterprise Application Platform
---

# Deploy a Java application with Red Hat JBoss Enterprise Application Platform on an Azure Red Hat OpenShift 4 cluster

This guide demonstrates how to deploy a Jakarta EE 8 application running on Red Hat JBoss Enterprise Application Platform (JBoss EAP) that connects to a Microsoft SQL Server to an Azure Red Hat OpenShift (ARO) 4 cluster by using [EAP Helm Charts](https://jbossas.github.io/eap-charts).

The guide takes a traditional Jakarta EE 8 application and drives you by the process of migrating it to a container orchestrator like Red Hat OpenShift running on Azure. First, it describes how you can package your application as a [Bootable Jar](https://access.redhat.com/documentation/en-us/red_hat_jboss_enterprise_application_platform/7.4/html/using_jboss_eap_xp_3.0.0/the-bootable-jar_default) to run it locally connecting the application to a docker Microsoft SQL Server Container. Finally, it shows you how you can deploy the Microsoft SQL Server on OpenShift and how to deploy three replicas of the JBoss EAP application by using Helm Charts.

The application is a stateful application that stores information on the HTTP Session. It makes use of the JBoss EAP clustering capabilities and uses the following Jakarta EE 8 and Microprofile 4.0 technologies:
* Jakarta Server Faces
* Enterprise Java Beans
* Java Persistence
* Microprofile Health

> [!IMPORTANT]
> This article uses a Microsoft SQL Server docker image running on a Linux container on Red Hat OpenShift. Before choosing to run a SQL Server container for production use cases, please review the [support policy for SQL Server Containers](https://support.microsoft.com/help/4047326/support-policy-for-microsoft-sql-server) to ensure that you are running on a supported configuration.

> [!IMPORTANT]
> While ARO is jointly engineered, operated, and supported by Red Hat and Microsoft to provide an integrated support experience, the software you run on top of ARO, including that described in this article, is subject to its own support and license terms. For details about support of ARO, see [Support lifecycle for Azure Red Hat OpenShift 4](../support-lifecycle.md). For details about support of the software described in this article, see the main pages for that software as listed in the article.
> * For support for JBoss Enterprise Application Platform, please go to [Red Hat JBoss Enterprise Application Platform](https://www.redhat.com/en/technologies/jboss-middleware/application-platform).

## Prerequisites

> [!NOTE]
> Azure Red Hat OpenShift requires a minimum of 40 cores to create and run an OpenShift cluster. The default Azure resource quota for a new Azure subscription does not meet this requirement. To request an increase in your resource limit, see [Standard quota: Increase limits by VM series](../azure-portal/supportability/per-vm-quota-requests.md). Note that the free trial subscription isn't eligible for a quota increase, [upgrade to a Pay-As-You-Go subscription](../cost-management-billing/manage/upgrade-azure-subscription.md) before requesting a quota increase.

1. Prepare a local machine with Unix-like operating system installed (for example, Ubuntu, macOS).
1. Install a Java SE implementation (for example, [AdoptOpenJDK OpenJDK 11 LTS/OpenJ9](https://adoptopenjdk.net/?variant=openjdk11&jvmVariant=openj9)).
1. Install [Maven](https://maven.apache.org/download.cgi) 3.6.3 or higher.
1. Install [Docker](https://docs.docker.com/get-docker/) for your OS.
1. Install [Azure CLI](/cli/azure/install-azure-cli) 2.29.2 or later.
1. Clone the code for this demo application on your local system. The demo application is on [GitHub](https://github.com/yersan/todo-list).
1. Follow the instructions in [Create an Azure Red Hat OpenShift 4 cluster](./tutorial-create-cluster.md).

   Though the "Get a Red Hat pull secret" step is labeled as optional, **it is required for this article**.  The pull secret enables your Azure Red Hat OpenShift cluster to find the JBoss EAP application images.

   If you plan to run memory-intensive applications on the cluster, specify the proper virtual machine size for the worker nodes using the `--worker-vm-size` parameter. For more information, see:

   * [Azure CLI to create a cluster](/cli/azure/aro#az_aro_create)
   * [Supported virtual machine sizes for memory optimized](./support-policies-v4.md#memory-optimized)

1. Connect to the cluster by following the steps in [Connect to an Azure Red Hat OpenShift 4 cluster](./tutorial-connect-cluster.md).
   * Follow the steps in "Install the OpenShift CLI" 
   * Connect to an Azure Red Hat OpenShift cluster using the OpenShift CLI with the user `kubeadmin`

1. Execute the following command to create the OpenShift project for this demo application:

    ```bash
    $ oc new-project eap-demo
    Now using project "eap-demo" on server "https://api.zhbq0jig.northeurope.aroapp.io:6443".

    You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app rails-postgresql-example

    to build a new example application in Ruby. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=k8s.gcr.io/serve_hostname
    ```

1. Execute the following command to add the view role to the default service account. This role is needed so the application can discover other pods and form a cluster with them:

    ```bash
    $ oc policy add-role-to-user view system:serviceaccount:$(oc project -q):default -n $(oc project -q)
    clusterrole.rbac.authorization.k8s.io/view added: "system:serviceaccount:eap-demo:default"
    ```

## Prepare the application

At this stage, you have cloned the demo application and your local repository is on the `main` branch. The demo application is a simple Jakarta EE 8 application that creates, reads, updates, and deletes records on a Microsoft SQL Server. Usually, you could have a JBoss EAP server installed on your local machine and configured with the required drivers and data source to allow connections from your application to the database server available on your environment.

When you are targeting OpenShift, you might want to trim the capabilities of your JBoss EAP server, for example, to reduce the security exposure of the provisioned server and reduce the overall footprint. You could also want to include some Microprofile specs to make your application more suitable for running on an OpenShift environment. When using JBoss EAP, one way to accomplish this is by packaging your application and your server in a single deployment unit known as Bootable JAR. Let's do that by adding the required changes to our demo application.

Navigate to your demo application local repository and change the branch to `bootable-jar`:

```bash
todo-list (main) $ git checkout bootable-jar
Switched to branch 'bootable-jar'
todo-list (bootable-jar) $
```

Let's do a quick review about what we have changed:

1. We have added the `wildfly-jar-maven` plugin to provision the server and the application in a single executable JAR file. The OpenShift deployment unit will be now our server with our application.
1. On the maven plugin, we have specified a set of Galleon layers. This configuration allows us to trim the server capabilities to only what we need.
1. Our application uses JSF with Ajax requests, which means there will be information stored in the HTTP Session. We don't want to lose such information if a pod is removed. We could save this information on the client and send it back on each request. However, there are cases where you may not distribute certain information to the clients. For this demo, we have chosen to replicate the session across all pod replicas. To do it, we have added `<distributable />` on the `web.xml` that together with the server clustering capabilities will make the HTTP Session distributable across all pods.
1. We have added two Microprofile Health Checks that allow identifying when the application is live and ready to receive requests.


## Run the application locally

Before deploying the application on OpenShift, we are going to run it locally to verify how it works. For the database, we are going to use a containerized Microsoft SQL Server running on Docker.

### Run the Microsoft SQL database on Docker

Follow the next steps to get the database server running on Docker and configured for the demo application:

1. Start a Docker container running the Microsoft SQL Server. You can get more information about this procedure by checking [this](https://docs.microsoft.com/en-us/sql/linux/quickstart-install-connect-docker) guide.

    ```bash
    $ sudo docker run \
    -e 'ACCEPT_EULA=Y' \
    -e 'SA_PASSWORD=Passw0rd!' \
    -p 1433:1433 --name mssqlserver -h mssqlserver \
    -d --rm mcr.microsoft.com/mssql/server:2019-latest
    ```

1. Login into the server and create the `todos_db` database. This database name is expected by the demo application.

    ```bash
    $ sudo docker exec -it mssqlserver "bash"
    mssql@mssqlserver:/$ /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P 'Passw0rd!'
    1> CREATE DATABASE todos_db
    2> GO
    1> exit
    mssql@mssqlserver:/$ exit
    ```

### Run the demo application locally

Follow the next steps to build and run the application locally.

1. Build the Bootable JAR. When we are building the Bootable JAR, we need to specify the database driver version we want to use:
 
    ```bash
    todo-list (bootable-jar) $  mvn clean package -Dbootable-jar \
    -Dorg.jboss.eap.datasources.mssqlserver.driver.version=7.4.1.jre11
    ```

1. Launch the Bootable JAR by using the following command. When we are launching the application, we need to pass the required environment variables to configure the data source:

    ```bash
    todo-list (bootable-jar) $ java -jar ./target/todo-list-bootable.jar -Djboss.node.name=node1 \
    -Denv.MSSQLSERVER_USER=SA -Denv.MSSQLSERVER_PASSWORD=Passw0rd! -Denv.MSSQLSERVER_JNDI=java:/comp/env/jdbc/mssqlds -Denv.MSSQLSERVER_URL=jdbc:sqlserver://localhost:1433;database=todos_db
    ```

1. (Optional) If you want to verify the clustering capabilities, you can also launch more instances of the same application by changing the `jboss.node.name` argument and, to avoid conflicts with the port numbers, shifting the numbers by using `jboss.socket.binding.port-offset` argument. For example, to launch a second instance that will represent a new pod on OpenShift, you can execute the following command in a new terminal window:
    
    ```bash  
    todo-list (bootable-jar) $ java -jar ./target/todo-list-bootable.jar -Djboss.node.name=node2 -Djboss.socket.binding.port-offset=10 \
    -Denv.MSSQLSERVER_USER=SA -Denv.MSSQLSERVER_PASSWORD=Passw0rd! -Denv.MSSQLSERVER_JNDI=java:/comp/env/jdbc/mssqlds -Denv.MSSQLSERVER_URL=jdbc:sqlserver://localhost:1433;database=todos_db
    ```

    > [!NOTE]
    > By default the Bootable JAR configures the JGroups subsystem to use the UDP protocol and sends messages to discover other cluster members to the 230.0.0.4 multicast address. To properly verify the clustering capabilities on your local machine, your Operating System should be capable of sending and receiving multicast datagrams and route them to the 230.0.0.4 IP through your ethernet interface. Check your network configuration and verify this is working if you see warnings related to the cluster on the server logs.

1. Open `http://localhost:8080/` in your browser to visit the application home page. If you have created more instances, you can access them by shifting the port number, for example `http://localhost:9080/`. The application will look similar to the following image:

    ![ToDo EAP demo Application](./media/howto-deploy-java-eap-app/todo-demo-application.png)

1. Check the application health enpoints. These endpoints will be used by OpenShift to verify when your pod is live and ready to receive user requests:

   ```bash  
    todo-list (bootable-jar) $ curl http://localhost:9990/health/live
    {"status":"UP","checks":[{"name":"SuccessfulCheck","status":"UP"}]}
    todo-list (bootable-jar) $ curl http://localhost:9990/health/ready
    {"status":"UP","checks":[{"name":"deployments-status","status":"UP","data":{"todo-list.war":"OK"}},{"name":"server-state","status":"UP","data":{"value":"running"}},{"name":"boot-errors","status":"UP"},{"name":"DBConnectionHealthCheck","status":"UP"}]}
    ```

1. Press **Control-C** to stop the application.
1. Execute the following command to stop the database server. Since we launched with `--rm` argument, the container will be automatically removed when it stops:

    ```bash
    docker stop mssqlserver
    ```
         
## Deploy to OpenShift

Similar to what we did in our local environment, before deploying the demo application on OpenShift we will deploy the database server as well. The database server will be deployed by using a [DeploymentConfig OpenShift API resource](https://docs.openshift.com/container-platform/4.8/applications/deployments/what-deployments-are.html#deployments-and-deploymentconfigs_what-deployments-are). The database server deployment configuration is available as a YALM file on the application source code. To deploy the application, we are going to use the JBoss EAP Helm Charts that are already available in ARO. We also need to supply the desired configuration, for example, the database password, the driver version we want to use, and the connection information used by the data source. This information will be supplied by config maps.

> [!NOTE]
> You can also use the JBoss EAP Operator to deploy this example, however, notice that the JBoss EAP Operator will deploy the application as stateful sets. Use the JBoss EAP Operator if your application requires one or more one of the following.
>
> - Stable, unique network identifiers.
> - Stable, persistent storage.
> - Ordered, graceful deployment and scaling.
> - Ordered, automated rolling updates.

Navigate to your demo application local repository and change the current branch to `bootable-jar-openshift`:

```bash
todo-list (bootable-jar) $ git checkout bootable-jar-openshift
Switched to branch 'bootable-jar-openshift'
todo-list (bootable-jar-openshift) $
```

Let's do a quick review about what we have changed:

1. We have added a new maven profile name `bootable-jar-openshift` that configures the Bootable JAR with a specific configuration for running the server on the cloud, for example, it enables the JGroup subsystem to use TCP requests to discover other pods by using the KUBE_PING protocol.
1. We have supplied a set of configuration files in the `todo-list/deployment` directory. In this directory, you will find the configuration files to deploy the database server and the application.

### Deploy the database server to OpenShift

The file to deploy the Microsoft SQL Server to OpenShift is `todo-list/deployment/mssqlserver/mssqlserver.yaml`. This file is composed by three configuration objects:
- A Service: To expose the SQL server port.
- A DeploymentConfig: To deploy the SQL server image.
- A PersistentVolumeClaim: To reclaim persistent disk space for the database. It uses the storage class named `managed-premium`. This storage class is the default disk provisioning method available on the ARO cluster.

This file expects the presence of a configmap object named `mssqlserver-cm` to supply the database administrator password. In the next steps, we will use the OpenShift CLI to create this config map, deploy the server and create the `todos_db`:

1. To create the config map, execute the following command on the `eap-demo` project created before at the pre-requisite steps section:

    ```bash
    todo-list (bootable-jar-openshift) $ oc create configmap mssqlserver-cm --from-literal 'sa-password=Passw0rd!'
    configmap/mssqlserver-cm created
    ```

1. Deploy the database server by executing the following:

    ```bash
    todo-list (bootable-jar-openshift) $ oc apply -f ./deployment/msqlserver/mssqlserver.yaml
    service/mssqlserver created
    deploymentconfig.apps.openshift.io/mssqlserver created
    persistentvolumeclaim/mssqlserver-pvc created
    ```

1. Monitor the status of the pods and wait until the database server is running:

    ```bash
    todo-list (bootable-jar-openshift *) $ oc get pods -w
    NAME                   READY   STATUS      RESTARTS   AGE
    mssqlserver-1-deploy   0/1     Completed   0          34s
    mssqlserver-1-gw7qw    1/1     Running     0          31s
    ```

1. Connect to the database pod and create the database `todos_db`:

    ```bash
    todo-list (bootable-jar-openshift) $ oc rsh mssqlserver-1-gw7qw
    sh-4.4$ /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P 'Passw0rd!'
    1> CREATE DATABASE todos_db
    2> GO
    1> exit
    sh-4.4$ exit
    exit
    ```

### Deploy the application to OpenShift

Now that we have the database server ready, we can deploy the demo application via EAP Helm Charts. The application Helm Chart configuration file is available at `todo-list/deployment/application/todo-list-helm-chart.yaml`. You could deploy this file via command line, however, to do so you would need to have Helm Charts installed locally. Instead of use the command line, the next steps explain how you can deploy this Helm Chart by using the OpenShift web console.

Before deploying the application, let's creating the expected config map that will hold the application configuration:

1. Execute the following to create the application config map:

    ```bash
    oc create configmap todo-list-cm \
    --from-literal mssqlserver-driver-version=7.4.1.jre11 \
    --from-literal mssqlserver-user=SA \
    --from-literal mssqlserver-password=Passw0rd! \
    --from-literal mssqlserver-database=todos_db \
    --from-literal mssqlserver-jndi=java:/comp/env/jdbc/mssqlds \
    --from-literal mssqlserver-cluster-password=mut2UTG6gDwNDcVW
    ```

    > [!NOTE]
    > You could have noticed from the above config map that we are not supplying the database Hostname and Port. That's not necessary. If you take a closer look at the application Helm Chart file, you will see that the database Hostname and Port are passed by using the following notations \$(MSSQLSERVER_SERVICE_HOST) and \$(MSSQLSERVER_SERVICE_PORT). This is a standard OpenShift notation that will make the application variables (MSSQLSERVER_HOST, MSSQLSERVER_PORT) get assigned to the values of pod environment variables (MSSQLSERVER_SERVICE_HOST, MSSQLSERVER_SERVICE_PORT) available at runtime, which are passed by OpenShift when the pod is launched. These variables are available to any pod because we have created a Service to expose the SQL server in the previous steps.

2. Open the OpenShift console and navigate to the developer view (in the `</> Developer` perspective in the left hand menu)

    ![OpenShift console developer view](./media/howto-deploy-java-eap-app/console_developer_view.png)

3. Once you are in the `</> Developer` perspective, ensure you have selected the `eap-demo` project at the projects combo box.

    ![OpenShift console project combo box](./media/howto-deploy-java-eap-app/console_project_combo_box.png)

4. Click on `+Add`, then Helm Chart. You will arrive at the Helm Chart catalog available on your ARO cluster. Write `eap` on the filter input box to filter all the Helm Charts and get the EAP ones. At this stage, you should see two options:

    ![OpenShift console EAP Helm Charts](./media/howto-deploy-java-eap-app/console_eap_helm_charts.png)

5. Since our application uses Microprofile capabilities, you have to select for this demo the one named EAP Xp (at the time of this writing, the exact version is EAP Xp 1.0.0). The `Xp` stands for Expansion Pack. With the JBoss Enterprise Application Platform expansion pack, developers can use Eclipse MicroProfile application programming interfaces (APIs) to build and deploy microservices-based applications.

6. Click on the `EAP Xp` Helm Chart and then `Install Helm Chart` button.

At this point, we need to configure the chart to be able to build and deploy the application:

1. Change the name of the release to `eap-todo-list-demo`. 
1. We can configure the Helm Chart either using a `Form View` or a `YAML View`. Check `YAML View` radio button in the `Configure via` box.
1. Then, change the YAML content to configure the Helm Chart by copying the content of the Helm Chart file available at `todo-list/deployment/application/todo-list-helm-chart.yaml` instead of the existing content:

   ![OpenShift console EAP Helm Chart YAML content](./media/howto-deploy-java-eap-app/console_eap_helm_charts_yalm_content.png)

1. Finally, click on the `Install` button to start the installation. This will open the `Topology view` with a graphical representation of the Helm release (named eap-todo-list-demo) and its associated resources.

    ![OpenShift console topology](./media/howto-deploy-java-eap-app/console_topology.png)

    The Helm Release (abbreviated HR) is named `eap-todo-list-demo`. It includes a Deployment resource (abbreviated D) also named `eap-todo-list-demo`. 

1. When build is finished (the bottom-left icon will display a green check) and the application is deployed (the circle outline is in dark blue), you can go to application the URL (using the top-right icon) from the route associated to the deployment.

    ![OpenShift console open application](./media/howto-deploy-java-eap-app/console_open_application.png)

1. The application is opened in your browser looking similar to the following image:

    ![OpenShift application running](./media/howto-deploy-java-eap-app/application_running_openshift.png)

1. The application shows you the name of the pod which has served the information. To verify the clustering capabilities you could add some Todos, change the filter to see all the records and mark some of them as completed. Then delete the pod name that appears on the application, and once deleted, create a new Todo on the same application window. You will see that the new Todo is added via an ajax request, and you will continue getting the same list. It means your view, which was stored on the HTTP Session, has not been reset to a new one when the response is served by a different pod, indeed you will see that the session ID has not changed. However, this time the response was served by a different pod. That verifies that your HTTP Session data is being replicated across all your pods.

## Clean up resources

### Delete the application

If you only want to delete your application, you can open the OpenShift console and, at the developer view, navigate to the `Helm` menu option. On this menu, you will see all the Helm Chart releases installed on your cluster. 

   ![OpenShift uninstall application](./media/howto-deploy-java-eap-app/console_uninstall_application.png)

Locate the `eap-todo-list-demo` Helm Chart and at the end of the row, click on the tree dots to open the action contextual menu entry.

Click on "Uninstal Helm Release" to remove the application. Notice the application config map is not part of the chart and you need to remove it separately if you no longer need it.

Execute the following command if you want to delete the configmap that hold the application configuration:

```bash
todo-list (bootable-jar-openshift) $  oc delete cm/todo-list-cm
configmap "todo-list-cm" deleted
```

### Delete the database

If you only want to delete the database and the related objects, execute the following command:

```bash
todo-list (bootable-jar-openshift) $ oc delete all -l app=mssql2019
replicationcontroller "mssqlserver-1" deleted
service "mssqlserver" deleted
deploymentconfig.apps.openshift.io "mssqlserver" deleted

todo-list (bootable-jar-openshift) $  oc delete cm/mssqlserver-cm
configmap "mssqlserver-cm" deleted
```

### Delete the OpenShift project

You can also delete all the configuration created for this demo by deleting the `eap-demo` project. To do so, execute the following:

```bash
todo-list (bootable-jar-openshift) $ oc delete project eap-demo
project.project.openshift.io "eap-demo" deleted
```

### Delete the ARO cluster

Delete the ARO cluster by following the steps in [Tutorial: Delete an Azure Red Hat OpenShift 4 cluster](./tutorial-delete-cluster.md)

## Next steps

In this guide, you learned how to:
> [!div class="checklist"]
>
> * Prepare an EAP application for OpenShift.
> * Run it locally
> * Run the containerized application and a Microsoft SQL Server on an ARO 4

You can learn more from references used in this guide:

* [Red Hat JBoss Enterprise Application Platform](https://www.redhat.com/en/technologies/jboss-middleware/application-platform)
* [EAP Helm Charts](https://jbossas.github.io/eap-charts/)
* [Azure Red Hat OpenShift](https://azure.microsoft.com/services/openshift/)
