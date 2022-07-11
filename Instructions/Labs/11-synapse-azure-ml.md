---
lab:
    title: 'Use Azure Machine Learning with Azure Synapse Analytics'
    module: 'Predictive Analytics with Microsoft Purview'
---

# Use Azure Machine Learning with Azure Synapse Analytics

Azure Machine Learning is a cloud-based platform for creating, deploying, and operating machine learning solutions. When combined with Azure Synapse Analytics, you can ingest and prepare data for machine learning model training, and then use Azure Machine learning to train and deploy a model. You can then use the model to support predictive analytics in Azure Synapse Analytics.

This lab will take approximately **60** minutes to complete.

## Before you start

You'll need an [Azure subscription](https://azure.microsoft.com/free) in which you have administrative-level access.

## Provision Azure resources

In this exercise, you'll integrate an Azure Machine Learning workspace and an Azure Synapse Analytics workspace. You'll start by using a script to provision these resources in your Azure subscription.

1. Sign into the [Azure portal](https://portal.azure.com) at `https://portal.azure.com`.
2. Use the **[\>_]** button to the right of the search bar at the top of the page to create a new Cloud Shell in the Azure portal, selecting a ***PowerShell*** environment and creating storage if prompted. The cloud shell provides a command line interface in a pane at the bottom of the Azure portal, as shown here:

    ![Azure portal with a cloud shell pane](./images/cloud-shell.png)

    > **Note**: If you have previously created a cloud shell that uses a *Bash* environment, use the the drop-down menu at the top left of the cloud shell pane to change it to ***PowerShell***.

3. Note that you can resize the cloud shell by dragging the separator bar at the top of the pane, or by using the **&#8212;**, **&#9723;**, and **X** icons at the top right of the pane to minimize, maximize, and close the pane. For more information about using the Azure Cloud Shell, see the [Azure Cloud Shell documentation](https://docs.microsoft.com/azure/cloud-shell/overview).

4. In the PowerShell pane, enter the following commands to clone this repo:

    ```
    rm -r dp-000 -f
    git clone https://github.com/MicrosoftLearning/mslearn-synapse dp-000
    ```

5. After the repo has been cloned, enter the following commands to change to the folder for this lab and run the **setup.ps1** script it contains:

    ```
    cd dp-000/Allfiles/Labs/11
    ./setup.ps1
    ```

6. If prompted, choose which subscription you want to use (this will only happen if you have access to multiple Azure subscriptions).
7. When prompted, enter a suitable password for your Azure SQL Database.

    > **Note**: Be sure to remember this password!

8. Wait for the script to complete - this typically takes around 15 minutes, but in some cases may take longer. While you are waiting, review the [Machine Learning capabilities in Azure Synapse Analytics](https://docs.microsoft.com/azure/synapse-analytics/machine-learning/what-is-machine-learning) article in the Azure Synapse Analytics documentation.

9. When the script has finished, in the Azure portal, view the **dp000-*xxxxxxx*** resource group that it created. This should include the following resources:
    - **aml*xxxxxxx*** - An Azure Machine Learning workspace.
    - **aml*xxxxxxx*insights*nnn*...** - An Application Insights instance.
    - **aml*xxxxxxx*keyvault*nnn*...** - A Key vault.
    - **aml*xxxxxxx*storage*nnn*...** - A storage account for Azure Machine Learning.
    - **datalake*xxxxxxx*** - A storage account for the data lake used in Azure Synapse Analytics
    - **spark*xxxxxxx* (synapse*xxxxxxx*/spark*xxxxxxx*)** - An Apache Spark pool.
    - **synapse*xxxxxxx*** - An Azure Synapse Analytics workspace.


> **Tip**: If, after running the setup script you decide not to complete the lab, be sure to delete the **dp000-*xxxxxxx*** resource group that was created in your Azure subscription to avoid unnecessary Azure costs.

## Explore data for model training

Before training a model, a data scientists generally explores the data with which the model will be trained, and prepares it by removing errors and outliers, identifying predictive features, and potentially using *feature engineering* techniques to augment the data with new derived values that will result in a more predictive model. In this example, we'll explore some existing data that relates to a bike rental scheme for which we'll later train a model to predict the expected number of rentals for a given day.

1. In the **dp000-*xxxxxxx*** resource group, select the **synapse*xxxxxxx*** Synapse workspace.
2. In the **Overview** page for your Synapse workspace, in the **Open Synapse Studio** card, select **Open** to open Synapse Studio in a new browser tab; signing in if prompted.
3. On the left side of Synapse Studio, use the **&rsaquo;&rsaquo;** icon to expand the menu - this reveals the different pages within Synapse Studio that you'll use to manage resources and perform data analytics tasks.
4. On the **Data** page, view the **Linked** tab. Then expand **Azure Data Lake Storage Gen2** and your **synapse*xxxxxxx* (Primary - datalake*xxxxxxx*)** data lake storage account, and select its **files** container.
5. In the **files** tab, open the **data** folder. Then select the **bike-rentals.csv** file it contains and in the **New notebook** menu, select **Load to DataFrame**. This creates a new **Notebook 1** tab for a notebook in which a single cell contains code similar to the following example:

    ```python
    %%pyspark
    df = spark.read.load('abfss://files@datalakexxxxxxx.dfs.core.windows.net/data/bike-rentals.csv', format='csv'
    ## If header exists uncomment line below
    ##, header=True
    )
    display(df.limit(10))
    ```

6. Modify the code as follows to uncomment the `, header=True` line and add a `, inferSchema=True` line:

    ```python
    %%pyspark
    df = spark.read.load('abfss://files@datalakexxxxxxx.dfs.core.windows.net/data/bike-rentals.csv', format='csv'
    ## If header exists uncomment line below
    , header=True
    , inferSchema=True
    )
    display(df.limit(10))
    ```

7. In the **Attach to** list, select your **spark*xxxxxxx*** Spark pool. Then use the **&#9655; Run all** button to run all of the cells in the notebook (there's currently only one!).

    Since this is the first time you've run any Spark code in this session, the Spark pool must be started. This means that the first run in the session can take a few minutes. Subsequent runs will be quicker.

8. When the code has finished running, view the output; which shows daily data for bike rentals. The data includes details for each day; including temporal, seasonal, and meteorological data) as well as the number of bikes rented.

    > ***Citation**: This data is derived from [Capital Bikeshare](https://ride.capitalbikeshare.com/system-data) and is used in accordance with the [published data license agreement](https://ride.capitalbikeshare.com/data-license-agreement).*

9. Use the **+ Code** button to add a new code cell to the notebook, and then enter the following code:

    ```python
    bike_df = df.select("season", 
                     "mnth", 
                     "holiday", 
                     "weekday", 
                     "workingday", 
                     "weathersit", 
                     "temp", 
                     "atemp", 
                     "hum", 
                     "windspeed", 
                     "rentals")
    bike_df.write.saveAsTable("bike_data")
    ```

10. Use the **&#9655;** button to the left of the code cell to run it. The code selects a subset of columns from the original dataset, and saves them as a table named **bike_data** in the default Spark database.

11. Add another new code cell, and use it to run the following code to query the table you just created:

    ```sql
    %%sql

    SELECT * FROM bike_data
    ```

12. Close the **Notebook 1** tab, ending the Spark session and discarding the notebook.

## Connect Azure Synapse Analytics to Azure Machine Learning

Your Azure Synapse Analytics workspace now includes a Spark database with a table of data that you want to use to train a machine learning model. To use Azure Machine Learning to train the model, you'll need to connect the two services.

### Configure Azure Machine Learning access for the Azure Synapse Analytics managed identity

Before you can connect to Azure Machine Learning from Azure Synapse Analytics, you need to ensure that the managed identity used by your Azure Synapse Analytics workspace has access to your Azure Machine Learning workspace.

1. Switch to the browser tab containing the Azure portal and view the **dp000-*xxxxxxx*** resource group.
2. Select the **aml*xxxxxxx*** Azure Machine Learning resource.
3. On the **Access control (IAM)** page for the **aml*xxxxxxx*** Azure Machine Learning resource, select **+ Add**, and select **Add role assignment**.
4. In the **Add role assignment** page, on the **Role** tab, select the **Contributor** role and then select **Next**.
5. On the **Members** tab, in the **Assign access to** options, select **Managed identity**. Then use the **+ Select members** link to select the **Synapse workspace** managed identity named **synapse*xxxxxxx*** (which is the managed identity used by your **synapse*xxxxxxx*** Azure Synapse Analytics workspace).
6. Use the **Review + Assign** button to assign the role membership, which adds the **synapse*xxxxxxx*** managed identity account to the **Contributor** role for the **aml*xxxxxxx*** Azure Machine Learning workspace.

### Create an Azure Machine Learning linked service

Now you can create a linked service to connect your Azure Machine Learning workspace to your Azure Synapse Analytics workspace.

1. In Synapse Studio, on the **Manage** page, in the **External connections** section, select the **Linked services** tab. The existing linked connections in your workspace are shown.
2. Select **+ New** and then in the **New linked service** pane, on the **Compute** tab, select **Azure Machine Learning**.
3. Continue adding the new linked service with the following settings:
    - **Name**: AzureML
    - **Description**: Azure Machine Learning service
    - **Connect via integration runtime**: AutoResolveIntegrationRuntime
    - **Authentication method**: System Assigned Managed Identity
    - **Azure Machine Learning workspace selection method**: From Azure subscription
    - **Azure subscription**: *Select your Azure subscription*
    - **Azure Machine Learning workspace name**: *Select your **amlxxxxxxx** Azure Machine Learning workspace*
4. Test the connection, and then create the linked service.
5. Use the **Publish all** button to save the new linked connection.

## Use automated machine learning to train a model

Now that you've connected your Azure Machine Learning workspace to your Azure Synapse Analytics workspace, you're ready to train a machine learning model to predict the number of rentals for a day based on its temporal, seasonal, and meteorological features. To accomplish this, you can use Azure Machine Learning's automated machine learning capability.

1. In Synapse Analytics, on the **Data** page, view the **Workspace** tab. If no databases are listed, you may need to use the **&#8635;** button at the top right of Synapse Studio to refresh the page.
2. Expand **Lake database** and the **default** database, and then expand the **Tables** folder to see the **bike_data** table you created previously.
3. Select the **bike_data** table, and in its **...** menu, select **Machine Learning** > **Train a new model**.
4. In the **Train a new model** pane, select **Regression** (you're going to predict the number of rentals, which is a numeric value; making this a *regression* model.)
5. Continue to configure a training job with the following details:
    - **Source data**: bike_data
    - **Azure Machine Learning workspace**: aml*xxxxxxx* (AzureML)
    - **Experiment name**: mslearn-bike-rentals
    - **Best model name**: rental-prediction-model
    - **Target column**: rentals (integer)
    - **Apache Spark pool**: spark*xxxxxxx*

    > **Note**: The automated machine learning job will run on your Spark pool compute. Your Spark pool must use the *Spark 2.4* runtime to support automated machine learning with Azure Machine Learning.

6. Continue to configure the regression model with the following settings:
    - **Primary metric**: Spearman correlation
    - **Maximum training job time (hours)**: 0.25
    - **Max concurrent iterations**: 2
    - **ONNX model compatibility**: Enable
7. Create the run, which will take a few minutes to be submitted and then run on your Spark pool. To view the run 's progress, on the **Monitor** page, select **Apache Spark applications** and look for the **Synapse_spark*xxxxxxx*_...** Spark session.

    The automated machine learning job may take 15 minutes or more to run, so now is a good time for a coffee break!

*To be continued...*