# Administrator steps to prepare the cluster

## Configure image streams

The template used for deploying the stocktrader application references container images in the `openshift` namespace in the internal registry. To prepare the cluster before the lab is used by students, create ImageStream tags for each container in the application. After authenticating as a user with permissions to create images.

### Option 1 - set up tags to the container images in Docker Hub

```bash
oc project openshift
oc tag docker.io/clouddragons/tradr:latest tradr:latest
oc tag docker.io/clouddragons/trade-history:latest trade-history:latest
oc tag docker.io/clouddragons/stock-quote:latest stock-quote:latest
oc tag docker.io/clouddragons/event-streams-consumer:latest event-streams-consumer:latest
oc tag docker.io/clouddragons/portfolio:latest portfolio:latest
```

### Option 2 - copy images down and then push them to the internal registry

This option requires a docker engine where the script is being run

```bash
git clone https://github.com/IBMStockTraderLite/stocktrader-openshift.git
cd stocktrader-openshift/scripts
./importImages.sh
```
