# Introduction

In my previous Blog series I explained how to turn Kyma in to a native Runtime for Cloud Application Programming Model based solutions. In this blog I would like to show you now how you can add to a existing CAP application the required deployment artifacts. Therefore I will use the well known Partner Reference Application (PRA) which is a great example of a SaaS Mulititenant Solutions applicable for Partners and Customers.

# Prerequistes and preparation

* You have setup a Kyma cluster with the CAP Operator as described in my Blog Series [Kyma Evolution: Transforming SAP Kyma into a tailor-made SaaS Platform for sbs extensions](https://community.sap.com/t5/technology-blog-posts-by-sap/kyma-evolution-transforming-sap-kyma-into-a-tailor-made-saas-platform-for/ba-p/14317418)
* You have created a dedicated BTP Provider subaccount and corresponding Kyma namespace according to my Blog Post [Mastering Kyma Multi-Tenancy: Mapping namespaces to different BTP Subaccounts](https://community.sap.com/t5/technology-blog-posts-by-sap/mastering-kyma-multi-tenancy-mapping-namespaces-to-different-btp/ba-p/14316995) which establish the connectivity with the BTP Service Operator.
* You created the required entitlements for the BTP Service. For the example in BTP Trial all the required services are automatically entitled.
* You configured your HANA Cloud Instance Mapping to allow the Kyma registered BTP Service Operator to create for the Partner Reference Application the required tenant schemas manually following the [documentation](https://help.sap.com/docs/hana-cloud/sap-hana-cloud-administration-guide/map-sap-hana-database-to-another-environment-context) or you can use a [cloud native operator](https://community.sap.com/t5/technology-blog-posts-by-sap/hana-cloud-instance-mapping-operator-for-kyma-environment/ba-p/13711534). To get the required cluster id for this step use:

    ```
    kubectl get configmap sap-btp-operator-config -n kyma-system -o jsonpath='{.data.CLUSTER_ID}'
    ```
* Fork the [Partner Reference Application](https://github.com/SAP-samples/partner-reference-application/tree/11e8a52e3e7fbf9d21447b562c9aa4be8ffedb31) and create a dedicated branch like prakyma from the main-multi-tenant branch.

# Add the additional required components

## Add Docker build to the PRA

As Docker is meanwhile as well the recommended approach for Cloud Foundry deployment you might have already done this for you cf project, however we will need to add this to the PRA. Therefore you need to have Docker installed on your maschine and need to have access to a Docker Registry. 
We need to create Dockerfiles for the following parts:
1. The Application Approuter under folder app/router we add the [Dockerfile](./app/router/Dockerfile)
2. The CAP MTXS application which will be used for the provider and subscriber tenant lifecycle operations we add the [Dockerfile](./mtx/Dockerfile)
3. For the CAP Application Server we add the Dockerfile to the Root 
4. The poetryslams und vistitors app Content Deployment to the HTML5 Repository we add the [Dockerfile](./app/Dockerfile) to the app folder. You need as well to adjust the package.json files of the poetryslams and the visitors application for the Docker build:

```
    "build:copy": "npm run build && npm run copy",
    "build": "ui5 build preload --clean-dest --config ui5-deploy.yaml --include-task=generateCachebusterInfo",
    "copy": "shx mkdir -p ../html5-deployer/resources/ && shx cp -rf ./dist/*.zip ../html5-deployer/resources/"

```
    and adding
```
    npm install shx -D in both application folders.
```

This will trigger the HTML5 content build and will copy the dist output to a folder which will be used to upload the conent to the HTML5 Repository.
Adjust your package.json to add the commands for the dockerbuild 
```
npm install cross-env  -D
npm install cross-var  -D
```

Export the variables according to your setup to the environement before you run the docker scripts for build and push

Example windows powershell
```
$Env:IMAGE_PREFIX = "espchris"
$Env:IMAGE_TAG = "0.0.1"
```
Trigger the docker build and push and check if everything is running correctly.

## Add the CAP Operator Plugin

Now go to the root folder of the project and install the [CAP Operator Plugin](https://github.com/cap-js/cap-operator-plugin) as dev dependency

```
npm add @cap-js/cap-operator-plugin -D
```

Now you can use the CAP Operator plugins to create the necessary resources for the deployment.

## Use the CAP Operator Plugin to generate the HELM Chart for the deployment

### Create the HELM configuration files using the CAP Operator plugins 

We will use the the **--with-configurable-templates** option to utilize template functions in the CAP Operator resources. In this version of the chart, all the CAP Operator resource configurations are defined in templates/cap-operator-cros.yaml. If you choose this option, you can skip the cds build step since the chart already contains the templates folder.

Before you create the template you currently need to adjust the package.json with the information that the project is using xsuaa for authentication as the plugin will use information from the package.json to create a draft for the template files.

```
 "[production]": {
        "multitenancy": true,
        "auth": "xsuaa" <--- Add this line
  }
```

Execute cds add cap-operator --with-mta .\mta.yaml --with-templates from your root folder. This step will create a new folder chart in your project directory including the necessary files supporting templating containing serviceInstances, serviceBindings and workloads based on the existing mta which is still a experimental feature and has some gaps.

As a first step we will add the images for the different workloads into the generated values.yaml file and will need to do a few changes to make the deployment working.

Now we will need to created the input-value.yaml file with the paramters to create the runtime deployment configuration. 
```
appName: pra
capOperatorSubdomain: cap-op
clusterDomain: c-4a2c372.kyma.ondemand.com
globalAccountId: 8ec9195a-6924-45d2-94da-a5c798578808
providerSubdomain: prakyma
tenantId: 21bf2fa9-4e01-4c47-93f1-e4fc8726c3cc
imagePullSecret: regcred
```

Now will generate the runtime-values for specific parts of the values.yaml parts with are specific to the deployment using 
```
npx cap-op-plugin generate-runtime-values --with-input-yaml .\chart\input-values.yaml
```

### Deploy the application to your namespace

Once done we use to 
```
helm upgrade -i -n pra pra .\chart\ --set-file serviceInstances.xsuaaBroker.jsonParameters=xs-security.json -f .\chart\runtime-values.yaml
```
to create the required service instances, bindings and to deploy the application artificats. One successfully done the provider tenant will be automatically be created by the CAP Operator and the Application will appear in the "Instances / Subscription Create" Screen of a new subscriber subaccount in your Global Account.
<img src="/capoperator/subscription.png" alt="Subscription Create" width="400"/>
<img src="/capoperator/subscriptionlist.png" alt="Subscription List" width="800"/>


## How to uninstall your application

To uninstall your appliction use, however before doing this you need to delete all tenants and the cap application as the unistall will delete all service instance and will bring otherwise your deployment in a inconsistent state.
```
helm uninstall -n pra pra 
```




