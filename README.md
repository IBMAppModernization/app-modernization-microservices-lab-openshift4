
# IBM Client Developer Advocacy App Modernization Series

## Lab - Deploying Microservices

### Deploying and testing the IBM Stock Trader sample application in  OpenShift


## Overview

In this lab you will deploy and test the *IBM Stock Trader application* on Red Hat OpenShift.

The *IBM Stock Trader*  application is a simple stock trading sample, where you can create various stock portfolios and add shares of stock to each for a commission. It keeps track of each porfolio’s total value and its loyalty level, which affect the commission charged per transaction. It also lets you submit feedback on the application, which can result in earning free (zero commission) trades, based on the tone of the feedback. (Tone is determined by calling the Watson Tone Analyzer).

The architecture of the  app is shown below:

![Architecture diagram](images/microservices-architecture.png)

The **portfolio** microservice sits at the center of the application. This microservice;
* persists trade data  using JDBC to a MariaDB database
* invokes the **stock-quote** service that invokes an API defined in API Connect in the public IBM Cloud to get stock quotes
* invokes the Tone Analyzer service in the public IBM Cloud to analyze the tone of submitted feedback
* sends trades to Kafka so that they can be recorded in Mongo by the **event-consumer** microservice
* calls the **trade-history** service to get aggregated historical trade  data.

**Tradr** is a Node.js UI for the porfolio service

The **event-consumer** service serves as a Kafka consumer and stores trade data published by the **portfolio** service in the Mongo database.

The **trade-history** service exposes an API to query the historical data in Mongo and is  called by the **portfolio** to get aggregated historical data.

The **stock-quote** service queries an external service to get real time stock quotes via an API Connect proxy.

## Setup

1. Login into the OpenShift web console using the user credentials provided to you

2. From the OpenShift web console click on your username in the upper right and select **Copy Login Command**

![Copy Login Command](images/ss0.png)

3. Paste the login command in a terminal window and run it (Note: leave the web console browser tab open as you'll need it later on in the lab)

4. Create a new OpenShift project for this lab (Note: your project name must be unique. We suggest you use msvcs-usernnn where usernnn is your student id for the project name e.g. user012)

```
  oc new-project [YOUR PROJECT NAME]
```

###  Step 1: Prepare for installation

Like a typical  Kubernetes app, Stock Trader use secrets and ConfigMaps to store information needed by one  or more microservices to access external  services and other microservices. We've  provided that info in a file hosted in Cloud Storage and there is a script that you'll use to retrieve it.


1.1 From a terminal window clone the Github repo that has everything needed to deploy the aggregated Stock Trader app.
```
   git clone https://github.com/IBMStockTraderLite/stocktrader-openshift.git
   cd stocktrader-openshift

   ```

1.2 Retrieve credentials and other details needed to create secrets and/or ConfigMaps. Ask you instructor for the **SETUPURL** and **STUDENTID** values needed as parameters in the command below.

   ```bash
   # Note you must be in the scripts sub folder or this command will fail
   cd scripts   

   # Your instructor will provide your with values for SETUPURL adn  STUDENID
   ./setupLab.sh SETUPURL STUDENTID
   ```

1.3 Verify that the output looks something like the following:

   ```
    Script being run from correct folder
    Validating URL to setup files ...
    Validating student id  ...
    Retrieving setup files ...
    Getting application  subdomain for cluster  ...
    Updating OpenShift template with shared host for all routes: stocktrader-microservices.apps.ocp.kubernetes-workshops.com
    Using stocktrader-user001 as Kafka topic name ...
    Updating variables.sh with Kafka topic : stocktrader-user001
    Setup completed successfully
   ```

1.4 Also verify that there is now a file called **variables.sh** in the current folder

###  Step 2: Install all the prereq

In this part  you'll install the prereqs step by step before installing the Stock Trader application.

2.1 Install MariaDB by running the following command. Verify that no errors are displayed by the installation script.

   ```
   ./setupMariaDB.sh

   ```

2.2 Install Mongo by running the following command. Verify that no errors are displayed by the installation script.

   ```
   ./setupMongo.sh

   ```

2.3 Create the DNS Proxy and store all the  access information as secrets  for the  external Kafka installation. Verify that no errors are displayed by the script.

   ```
   ./setupKafka.sh

   ```

2.4 Store all the  access information as secrets for the API Connect proxy to the external  realtime stock quote . Verify that no errors are displayed by the script.

   ```
   ./setupAPIConnect.sh

   ```

2.5 Store all the  access information as secrets for the  external  Watson Tone Analyzer service . Verify that no errors are displayed by the script.

   ```
   ./setupWatson.sh

   ```

2.6 Initialize the MariaDB transactional database with some data. Verify that no errors are displayed by the script.

   ```
   ./initDatabase.sh

   ```

2.7 Verify your progress so far. Run the following to see the pods you have so far

   ```
   oc get pods
   ```

   The output should show pods for MariaDB and Mongo and they both should be running and in the READY state

   ```
       NAME              READY     STATUS    RESTARTS   AGE
     mariadb-1-shzjl   1/1       Running   0          2m
     mongodb-1-gqpln   1/1       Running   0          2m
   ```

2.8 Next look at your services

   ```
   oc get svc
   ```

2.9 Verify that the output shows services for Mongo, MariaDB and your DNS proxy to Kafka

  ```  
  NAME              TYPE           CLUSTER-IP      EXTERNAL-IP                                                                 
  kafka-dns-proxy   ExternalName   <none>          broker-0-0mqz41lc21pr467x.kafka.svc01.us-south.eventstreams.cloud.ibm.com   
  mariadb           ClusterIP      172.30.103.15   <none>                                                                      
  mongodb           ClusterIP      172.30.235.11   <none>                                                                      
  ```

###  Step 3: Install the StockTrader app

In this part  you'll install all the Stock Trader microservices using a template  for all the microservices. Note that all the  microservices require some of the information stored via secrets in the scripts you ran in the previous section.

3.1 Go back to the top level folder of the  cloned repo

   ```
   cd ..
   ```

3.2 Install the microservices chart. Verify that no errors are displayed

   ```
   oc process -f templates/stock-trader.yaml | oc create -f -
   ```

3.3 Verify that all the pods are running and are in the READY state. Note you may have to run this command multiple times before all the pods become READY.

   ```
   oc get pods
   ```

3.4 Keep running the command  until the output looks something like this:

   ```
     NAME                             READY     STATUS    RESTARTS   AGE
   event-streams-consumer-1-455pj   1/1       Running   0          2m
   mariadb-1-shzjl                  1/1       Running   0          2d
   mongodb-1-gqpln                  1/1       Running   0          2d
   portfolio-1-vkxnp                1/1       Running   0          2m
   stockquote-1-zck9n               1/1       Running   0          2m
   trade-history-1-5pngp            1/1       Running   0          2m
   tradr-1-qdps9                    1/1       Running   0          2m
   ```

3.5 The app uses OpenShift routes to provide access outside of the cluster. Use the following command to get the external hostnames you'll need to access Stock Trader.

   ```
   oc  get routes
   ```

3.6 Verify the output looks something like the following. The value in the  HOST/PORT column is the common hostname used for all the  microservices. The value in the PATH column is the unique path for each microservice.

   ```
   NAME            HOST/PORT                                                     PATH             SERVICES           
   portfolio       stocktrader-microservices.apps.ocp.kubernetes-workshops.com   /portfolio       portfolio       
   stockquote      stocktrader-microservices.apps.ocp.kubernetes-workshops.com   /stock-quote     stockquote      
   trade-history   stocktrader-microservices.apps.ocp.kubernetes-workshops.com   /trade-history   trade-history   
   tradr           stocktrader-microservices.apps.ocp.kubernetes-workshops.com   /tradr           tradr           
   ```
In this example the URL for the **tradr** UI is http://stocktrader-microservices.apps.ocp.kubernetes-workshops.com/tradr (the common hostname plus the PATH for **tradr**).

## Step 4: Test the app

In this part you'll verify that the various microservices are working as designed.

4.1 Bring up the **tradr** web application using the  hostname you noted at the end of the  previous  section

![Login page](images/ss1.png)

4.2 Log in with the following credentials (note these are the only values that will work)

   ```
   username:  stock
   password: trader
   ```

![Dashboard](images/ss2.png)

4.3 Click **Add Client** and name the client `Client2`. Click **OK**

![Dashboard](images/ss3.png)

4.4 Click on the link in the **Name** column to see the  details of Client2

4.5 Do 3 or 4 "Buy" operations with different stock tickers (e.g. STT, T, GOOG, IBM).

![Dashboard](images/ss4.png)

4.6 Sell part of one of the holdings you just bought and verify that the table is updated appropriately

4.7 Click on **Feedback** and submit some client feedback. If the client sounds really angry they'll get 3 free trades otherwise they'll get one free trade.

![Feedback](images/ss5.png)

4.8 Verify that the data flow of `portfolio->Kafka->event-consumer->trade-history-Mongo` works by querying the **trade-history** service via an endpoint  that makes it do a Mongo query.  Add the path `/trades/Client2` to the  route for the **trade-history** microservice. For example `http://stocktrader-microservices.apps.ocp.kubernetes-workshops.com/trade-history/trades/Client2` for the route used in the example above.

4.9 Enter the URL in another browser tab and verify that the history has captured  all the  trades you did while testing. A partial screen shot of what you should get back is shown below:

![Trade History](images/ss6.png)


## Cleanup

Free up resources for subsequent labs by deleting the Stock Trader app.

1. Run the following commands to cleanup (note: you can copy all the commands at once and post then into you command window)

   ```
   cd scripts
   oc delete all,routes --selector app=stock-trader
   ./cleanupWatson.sh
   ./cleanupAPIConnect.sh
   ./cleanupKafka.sh
   ./cleanupMongo.sh
   ./cleanupMariaDB.sh
   cd -
   ```


## Summary
You installed and then tested the  Stock Trader microservices sample application and got some insight into the challenges of deploying microservices apps in an OpenShift cluster.
