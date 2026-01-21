# Exercise 2.2: Enable monitoring and end-to-end tracing

### Estimated Duration : 60 minutes

## Lab Scenario

You have created your Azure Container Apps environment, deployed your applications to it and exposed them through the api-gateway service. Now that everything is up and running, it would be nice to monitor the availability of your applications and to be able to see if any errors or exceptions occur in your applications. In this lab you will add monitoring and end-to-end tracing to your applications.

## Lab Objectives

After you complete this lab, you will be able to:

 - Inspect your Azure Container Apps in the Azure Portal
 - Configure Azure Container Apps environment monitoring
 - Configure Application Insights to receive monitoring information from your applications
 - Analyze application specific monitoring data

## Task 1: Inspect your Azure Container Apps in the Azure Portal

In this task, you will use Azure portal to inspect the customers-service container app. Once the replica is running, you will view live console logs through the Log Stream feature.

1. Navigate to the **Azure portal**, and to the resource group where you deployed your Azure Container Apps environment. Select the **api-gateway** container app. 

   ![](./media/javaimg1.png)

1. In the left menu, under **Application**, select **Revisions and replicas** and check the status of replica. It should be in **Running state**.

   ![](./media/javaimg2.png)

1. Once the replica is in running state, you can see the live console logs by selecting **Log Stream** from left menu.

   ![](./media/javaimg3.png)

## Task 2: Configure Azure Container Apps environment monitoring

In this task, you will configure monitoring for your Azure Container Apps environment using a Log Analytics Workspace. After setting up logging, you will verify the collected data by querying ContainerAppSystemLogs in the Azure portal.

1. Navigate back to the **Visual Studio Code** terminal, in which the **Git Bash** is open.

   >**LabTip:** If you face any errors while running these commands, just add `MSYS_NO_PATHCONV=1` before that command because Git Bash might incorrectly interpret parts of Azure Resource IDs (which contain slashes) as file paths, resulting in errors.

1. Run the following command, which will create a **Log Analytics Workspace**

   ```
   WORKSPACE=la-petclinic-<inject key="DeploymentID" enableCopy="false" />

   az monitor log-analytics workspace create \
    --resource-group $RESOURCE_GROUP \
    --workspace-name $WORKSPACE   
   ```

   >**&#128161;Tip:** A Log Analytics Workspace in Azure is a centralized platform for collecting, analyzing, and querying log and performance data from various resources and services.

1. Enable logging on your Azure Container Apps environment by running the below command block.

   ```
   ACA_ENVIRONMENT=acaenv-petclinic-<inject key="DeploymentID" enableCopy="false" />

   WORKSPACECID=$(az monitor log-analytics workspace show -n $WORKSPACE -g $RESOURCE_GROUP --query customerId -o tsv)

   WORKSPACEKEY=$(az monitor log-analytics workspace get-shared-keys -n $WORKSPACE -g $RESOURCE_GROUP --query primarySharedKey -o tsv)

   az containerapp env update \
        --name $ACA_ENVIRONMENT \
        --resource-group $RESOURCE_GROUP \
        --logs-destination log-analytics \
        --logs-workspace-id $WORKSPACECID \
        --logs-workspace-key $WORKSPACEKEY   
   ```

1. To verify that monitoring data is available in your Log Analytics workspace, in the Azure Portal, navigate back to the **Customer Service** container app page.

1. From the left menu, select **Logs** under monitoring and cancel the tabs by clicking **X**.

   ![](./media/ex2img5.png)

1. In the **New Query** page, select **ContainerAppSystemLogs_CL (1)** under Custom Logs,  verify it on the **query pane (2)** and click on **Run (3)**

   ![](./media/ex2img6.png)

1. After running the query, you will be able to see the data.

   ![](./media/ex2img7.png)

   >**&#128221;Note:** If you are not able to see any results, move with further tasks, comback and check after sometime.

> **Congratulations** on completing the task! Now, it's time to validate it. Here are the steps:
- If you receive a success message, you can proceed to the next task.
- If not, carefully read the error message and retry the step, following the instructions in the lab guide.
- If you need any assistance, please contact us at cloudlabs-support@spektrasystems.com. We are available 24/7 to help you out.
     
<validation step="	5abe5e9e-0f11-4946-a49f-1d038c8f5c7a" />

## Task 3: Configure Application Insights to receive monitoring information from your applications

You now know how to set up monitoring for your overall Azure Container Apps environment. However, you would also like to get monitoring info on how your applications run in the cluster. To track Application specific monitoring data, you can use Application Insights.

In this task, you will configure Application Insights to receive monitoring data from your applications. This will enable you to track performance and diagnose issues by collecting telemetry data in real time.

1. As a first step, you will need to create an **Application Insights** resource. To do that, navigate back to the **Git Bash** terminal window inside your **Visual Studio Code** and run the following command.

    ```
     LOCATION=<inject key="Region" enableCopy="false" />

     WORKSPACEID=$(az monitor log-analytics workspace show -n $WORKSPACE -g $RESOURCE_GROUP --query id -o tsv)

     AINAME=ai-petclinic-<inject key="DeploymentID" enableCopy="false" />

     az extension add -n application-insights

     MSYS_NO_PATHCONV=1 az monitor app-insights component create \
         --app $AINAME \
         --location $LOCATION \
         --kind web \
         -g $RESOURCE_GROUP \
         --workspace $WORKSPACEID
    ```
    >**&#128161;Tip:** Azure Application Insights is an extensible service for monitoring live applications, providing real-time insights into performance, usage, and diagnostics.

1. Make note of the Application Insights connection string, you will need this later in this module.
 
    ```
    AI_CONNECTIONSTRING=$(az monitor app-insights component show --app $AINAME -g $RESOURCE_GROUP --query connectionString --output tsv)

    echo $AI_CONNECTIONSTRING
    ```

1. Since you will now be containerizing the different microservices, you will also need to create a new **Azure Container Registry (ACR)** instance for holding your container images.

    ```
    MYACR=acrpetclinics<inject key="DeploymentID" enableCopy="false" />

    az acr create \
     -n $MYACR \
     -g $RESOURCE_GROUP \
     --sku Basic \
     --admin-enabled true
    ```
   
    >**&#128161;Tip:** Azure Container Registry (ACR) is a fully managed service that allows you to store, build, and manage container images and artifacts in a private registry, seamlessly integrating with your Azure services and CI/CD pipelines.

1. Run the following command to  create an identity which can be used by your container apps to connect to the Container Registry. 

    ```
    ACA_IDENTITY=uid-petclinic-<inject key="DeploymentID" enableCopy="false" />

    MSYS_NO_PATHCONV=1 az identity create --resource-group $RESOURCE_GROUP --name $ACA_IDENTITY --output json
    USER_ID=$(az identity show --resource-group $RESOURCE_GROUP --name $ACA_IDENTITY --query id --output tsv)

    SP_ID=$(az identity show --resource-group $RESOURCE_GROUP --name $ACA_IDENTITY --query principalId --output tsv)
    echo $USER_ID
    echo $SP_ID
    ```
1. Assign user identity to the container apps environment by running the below command.

    ```
    MSYS_NO_PATHCONV=1 az containerapp env identity assign -g $RESOURCE_GROUP -n $ACA_ENVIRONMENT --user-assigned $USER_ID
    ```

1. Assign access for the container app identity to pull images from your container registry.

    ```
    ACR_ID=$(az acr show -n $MYACR -g $RESOURCE_GROUP --query id -o tsv)
    MSYS_NO_PATHCONV=1 az role assignment create --assignee $SP_ID --scope $ACR_ID --role acrpull
    ```

1. As you are in the `src` directory, create a **staging-acr** folder and navigate to it by running this command.

    ```
    mkdir staging-acr
    cd ./staging-acr
    ```

1. In this new folder download the latest application insights agent jar file. Also rename the jar file to **ai.jar**. You can do this by running the following command.

    ```
    AI_VERSION=3.5.4
    curl -L -o ai.jar "https://github.com/microsoft/ApplicationInsights-Java/releases/download/$AI_VERSION/applicationinsights-agent-$AI_VERSION.jar"
    ```
    ![](./media/ex-02-new1.png)

1. Now select the **staging-acr (1)** directory from explorer. Click on **New File Icon (2)** at the top to create a new file inside staging-acr directory.

    ![](./media/ex2img1.png)

1. Provide the name of the file as `Dockerfile` and hit enter to open that file.

    >**&#128221;Note:** Provide the name carefully, as the Dockerfile should not have any file extension and also it is case sensitive.

1. Paste the following data inside the `Dockerfile`. This Dockerfile copies the application jar file and ai.jar file into the container. It also adds the ai.jar file as a javaagent to your app, so it can start logging.

    ```
    # Build stage
    FROM mcr.microsoft.com/openjdk/jdk:17-mariner
    COPY spring-petclinic-my-service-3.2.5.jar app.jar
    COPY ai.jar ai.jar
    EXPOSE 8080

    # Run the jar file
    ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-javaagent:/ai.jar","-jar","/app.jar"]

    ```

1. Once the changes are done, make sure to save the file by using the **Save** option from file menu.

    ![](./media/ex1img53.png)

1. Run the below command blocks one by one, which deletes the previous container app and creates a new container app from `Dockerfile`.

    ```
    export APP_NAME="api-gateway"
    cp ../spring-petclinic-$APP_NAME/target/spring-petclinic-$APP_NAME-$VERSION.jar  spring-petclinic-$APP_NAME-$VERSION.jar
    sed -i "s|my-service|$APP_NAME|g" Dockerfile
    ```

    ```
    az containerapp delete --name $APP_NAME --resource-group $RESOURCE_GROUP --yes
    ```

    ```
    MSYS_NO_PATHCONV=1 az containerapp create \
       --name $APP_NAME \
       --resource-group $RESOURCE_GROUP \
       --source .  \
       --env-vars APPLICATIONINSIGHTS_CONNECTION_STRING=$AI_CONNECTIONSTRING APPLICATIONINSIGHTS_CONFIGURATION_CONTENT='{"role": {"name": "api-gateway"}}' InstrumentationKey=$AI_CONNECTIONSTRING \
       --registry-server $MYACR.azurecr.io \
       --registry-identity $USER_ID \
       --environment $ACA_ENVIRONMENT \
       --user-assigned $USER_ID \
       --ingress external \
       --target-port 8080 \
       --min-replicas 1 \
       --runtime java
    ```
   
    ```
    sed -i "s|$APP_NAME|my-service|g" Dockerfile
    rm spring-petclinic-$APP_NAME-$VERSION.jar
    ```

1. Once the api-gateway deployment has succeeded, execute the same statements for the other microservices.

    * customers-service

    ```
    export APP_NAME="customers-service"
    cp ../spring-petclinic-$APP_NAME/target/spring-petclinic-$APP_NAME-$VERSION.jar spring-petclinic-$APP_NAME-$VERSION.jar
    sed -i "s|my-service|$APP_NAME|g" Dockerfile
    ```

    ```
    az containerapp delete --name $APP_NAME --resource-group $RESOURCE_GROUP --yes
    ```
 
    ```
    MSYS_NO_PATHCONV=1 az containerapp create \
       --name $APP_NAME \
       --resource-group $RESOURCE_GROUP \
       --source .  \
       --env-vars APPLICATIONINSIGHTS_CONNECTION_STRING=$AI_CONNECTIONSTRING APPLICATIONINSIGHTS_CONFIGURATION_CONTENT='{"role": {"name": "customers-service"}}' InstrumentationKey=$AI_CONNECTIONSTRING \
       --registry-server $MYACR.azurecr.io \
       --registry-identity $USER_ID \
       --environment $ACA_ENVIRONMENT \
       --user-assigned $USER_ID \
       --ingress internal \
       --target-port 8080 \
       --min-replicas 1 \
       --runtime java
    ``` 

    ```
    sed -i "s|$APP_NAME|my-service|g" Dockerfile
    rm spring-petclinic-$APP_NAME-$VERSION.jar
    ```

    * vets-service

    ```
    export APP_NAME="vets-service"
    cp ../spring-petclinic-$APP_NAME/target/spring-petclinic-$APP_NAME-$VERSION.jar spring-petclinic-$APP_NAME-$VERSION.jar
    sed -i "s|my-service|$APP_NAME|g" Dockerfile
    ```

    ```
    az containerapp delete --name $APP_NAME --resource-group $RESOURCE_GROUP --yes
    ```

    ```
    MSYS_NO_PATHCONV=1 az containerapp create \
       --name $APP_NAME \
       --resource-group $RESOURCE_GROUP \
       --source .  \
       --env-vars APPLICATIONINSIGHTS_CONNECTION_STRING=$AI_CONNECTIONSTRING APPLICATIONINSIGHTS_CONFIGURATION_CONTENT='{"role": {"name": "vets-service"}}' InstrumentationKey=$AI_CONNECTIONSTRING \
       --registry-server $MYACR.azurecr.io \
       --registry-identity $USER_ID \
       --environment $ACA_ENVIRONMENT \
       --user-assigned $USER_ID \
       --ingress internal \
       --target-port 8080 \
       --min-replicas 1 \
       --runtime java
    ```

    ```
    sed -i "s|$APP_NAME|my-service|g" Dockerfile
    rm spring-petclinic-$APP_NAME-$VERSION.jar
    ```

    * visits-service

    ```
    export APP_NAME="visits-service"
    cp ../spring-petclinic-$APP_NAME/target/spring-petclinic-$APP_NAME-$VERSION.jar spring-petclinic-$APP_NAME-$VERSION.jar
    sed -i "s|my-service|$APP_NAME|g" Dockerfile
    ```

    ```
    az containerapp delete --name $APP_NAME --resource-group $RESOURCE_GROUP --yes
    ```

    ```
    MSYS_NO_PATHCONV=1 az containerapp create \
       --name $APP_NAME \
       --resource-group $RESOURCE_GROUP \
       --source .  \
       --env-vars APPLICATIONINSIGHTS_CONNECTION_STRING=$AI_CONNECTIONSTRING APPLICATIONINSIGHTS_CONFIGURATION_CONTENT='{"role": {"name": "visits-service"}}' InstrumentationKey=$AI_CONNECTIONSTRING \
       --registry-server $MYACR.azurecr.io \
       --registry-identity $USER_ID \
       --environment $ACA_ENVIRONMENT \
       --user-assigned $USER_ID \
       --ingress internal \
       --target-port 8080 \
       --min-replicas 1 \
       --runtime java
    ```

    ```
    sed -i "s|$APP_NAME|my-service|g" Dockerfile
    rm spring-petclinic-$APP_NAME-$VERSION.jar
    ```

    * admin-server

    ```
    export APP_NAME="admin-server"
    cp ../spring-petclinic-$APP_NAME/target/spring-petclinic-$APP_NAME-$VERSION.jar spring-petclinic-$APP_NAME-$VERSION.jar
    sed -i "s|my-service|$APP_NAME|g" Dockerfile
    ```

    ```
    az containerapp delete --name $APP_NAME --resource-group $RESOURCE_GROUP --yes
    ```

    ```
    MSYS_NO_PATHCONV=1 az containerapp create \
       --name $APP_NAME \
       --resource-group $RESOURCE_GROUP \
       --source .  \
       --env-vars APPLICATIONINSIGHTS_CONNECTION_STRING=$AI_CONNECTIONSTRING APPLICATIONINSIGHTS_CONFIGURATION_CONTENT='{"role": {"name": "admin-server"}}' InstrumentationKey=$AI_CONNECTIONSTRING \
       --registry-server $MYACR.azurecr.io \
       --registry-identity $USER_ID \
       --environment $ACA_ENVIRONMENT \
       --user-assigned $USER_ID \
       --ingress external \
       --target-port 8080 \
       --min-replicas 1 \
       --runtime java
    ```
  
    ```
    sed -i "s|$APP_NAME|my-service|g" Dockerfile
    rm spring-petclinic-$APP_NAME-$VERSION.jar 
    ```

1. Once all the deployment succeeds, navigate to your **Container App Environment** and select **Services (1)** from left menu and select **myconfigserver (2)**.

    ![](./media/ex2img12.png)

1. On the **myconfiserver** pane, select all Container Apps under **Bindings (1)** and click on **Next (2)**.

    ![](./media/ex2img13.png)

1. Review the configurations and click on **Configure**.

1. Once the configuration is completed for myconfigserver, navigate back to **Services** and select **eureka**.

    ![](./media/ex2img14.png)

1. On the **eureka** pane, select all Container Apps under **Bindings (1)** and click on **Next (2)**.

    ![](./media/ex2img13.png)

1. Review the configurations and click on **Configure**.

1. Please make sure, that the **Connected Apps** property for both java components are populated to **5** before proceeding further.

    ![](./media/ex-01-new2.png)

> **Congratulations** on completing the task! Now, it's time to validate it. Here are the steps:
- If you receive a success message, you can proceed to the next task.
- If not, carefully read the error message and retry the step, following the instructions in the lab guide.
- If you need any assistance, please contact us at cloudlabs-support@spektrasystems.com. We are available 24/7 to help you out.
     
<validation step="5f64a11d-c413-4d26-a1d8-720a3c22f147" />

 ## Task 4: Analyze application specific monitoring data

 Now that Application Insights is properly configured, you can use this service to monitor what is going on in your application. You can follow the below guidance to do so.

 In this task, you will analyze the results those are retrived on Application Insights resource. You will go through multiple features available in Application Insights to inspect your application.

1. Navigate to the Azure Portal from your browser. Select **petclinic-<inject key="DeploymentID" enableCopy="false" />** resource group from the list.
  
    ![](./media/ex1img9.png)

1. From the resource list of your resource group, select **Application Insights** resource.

    ![](./media/ex2img15.png) 

1. On the overview page of **Application Insights**, you can already see data about `Failed requests`, `Server response time`, `server requests` and `Availability`.

    ![](./media/ex2img16.png)

1. Select **Application map** from the left menu. This will show you information about the different applications running in your Spring Cloud Service and their dependencies.

    ![](./media/ex-02-new3.png)

    ![](./media/ex2img17.png)

1. Select the **api-gateway** service. This will show you details about this application, like slowest requests and failed dependencies.

    ![](./media/ex2img18.png)

1. Select **Investigate performance**. This will show you more data on performance.

    ![](./media/ex2img19.png)

    ![](./media/ex2img20.png)

    >**&#128161;Tip:** You can also drag your mouse on the graph to select a specific time period, and it will update the view.

1. Select **Live Metrics** from left menu, to see live metrics of your application. This will show you near real time performance of your application, as well as the logs and traces coming in.
   
    ![](./media/ex2img21.png)

1. Select **Availability (1)** from left menu, click on **Add Standard test (2)**, to configure an availability test for your application.

    ![](./media/ex2img22.png)
   
1. On **Create Standerd test** pane, provide the following details:

    - **Test name** : `demotest`**(1)**.
    - **URL** : Provide the **Application URL** of `api-gateway` **(2)** service.
    - Click on **Create (3)**.

      ![](./media/ex2img23.png)

      >Once created every 5 minutes your application will now be pinged for availability from 5 test locations.

1. Select the **three dots** on the right of your newly created availability test and select **Open Rules (Alerts) page**.

    ![](./media/ex2img24.png)

    >By default there are no action groups associated with this alert rule. We will not configure them in this lab, but just for your information, with action groups you can send email or SMS notifications to specific people or groups.

1. Navigate back to your Application Insights resource.

1. Select **Failures** from left menu, to see information on all failures in your applications.

    ![](./media/ex2img25.png)

1. Select **Performance** from left menu, to see performance data of your applications’ operations.

    ![](./media/ex2img26.png)

1. Select **Logs (1)** from left menu, select **Queries (2)** from the top and double click on **Operations performance (4)** by expanding **Performance (3)**.

    ![](./media/ex2img27.png)

1. Once you double click on Operations performance, you can see it on query pane. Click on **Run** to run the query.

    ![](./media/ex2img28.png)

1. After running the query, you can see the results in the results pane.

    ![](./media/ex2img29.png)

## Summary

In this exercise, you have inspected your Azure Container Apps, set up environment monitoring using Log Analytics, and configured Application Insights to collect telemetry data. You also analyzed specific monitoring data to better understand your application's performance.

### You successfully completed this Lab!
