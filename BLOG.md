In my previous Blog series i explained how to turn Kyma in to a native Runtime for Cloud Application Programming Model based solutions. In this blog I would like to show you now how you can add to a existing CAP application the required deployment artifacts. Therefore I wil use  Partner Reference Application (PRA) which is a great example of a SaaS Mulititenant Solutions applicable for Partners and Customers.

1. Preparation

You have setup a Kyma cluster with the CAP Operator as described in my Blog Series ...
You have created a dedicated BTP Provider subaccount and corresponding Kyma namespace according to my Blog Post ....


2. Steps to add the additional components to for the deplyoments

2.1 Checkout the Partner Reference Application

Create a dedicated branch prakyma 

Go to the project root folder and install the CAP Operator Plugin as dev dependency

npm add @cap-js/cap-operator-plugin -D

Now you can use the CAP Operator plugins to create the necessary resources for the deployment.


2.1. Add Docker build to your project. 

As Docker is meanwhile as well the recommended approach for Cloud Foundry deployment you might have already done this for you application. Therefore you need to have Docker installed on your Laptop and need to have access to a Docker Registry. 

Here

