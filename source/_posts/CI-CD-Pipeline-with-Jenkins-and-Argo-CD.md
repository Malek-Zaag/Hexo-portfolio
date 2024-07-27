---
title: CI/CD Pipeline with Jenkins and Argo CD
date: 2022-12-11 18:27:09
tags:
categories: [Blogs]
---

# CI/CD pipeline with Jenkins and Argo CD

Overview :

Hello, in this article we are going to build a CI/CD pipeline from scratch and we are going to talk about each component in it .

Here is our pipeline :

![](https://cdn-images-1.medium.com/max/2000/1*Azvr5MKmYd8DzrghPLWjPw.png)

### Jenkins :

![Jenkins](https://cdn-images-1.medium.com/max/2000/0*BUk9Yc1Dhs7DrRxk)_Jenkins_

What is Jenkins and what it is used for ?

Jenkins is an open source continuous integration/continuous delivery and deployment (CI/CD) automation software DevOps tool written in the Java programming language. It is used **to implement CI/CD workflows, called pipelines**.

### Pipelines :

Let's talk about what is a CI/CD pipeline .

A continuous integration and continuous deployment (CI/CD) pipeline is a series of steps that must be performed in order to deliver a new version of software. CI/CD pipelines are a practice focused on improving software delivery throughout the software development life cycle via automation.

**Installing Jenkins on Windows :**

To install Jenkins, we go straight ahead to its official website and download the .msi file and we install it :

![](https://cdn-images-1.medium.com/max/2000/1*eT0GJ20ItAoLSYH7B9ojmA.png)

Now it is time to set up credentials .

**Credentials :**

To let Jenkins access my repository I need to configure the credentials for my github account :

![](https://cdn-images-1.medium.com/max/2000/1*ryrFHxldBE8-gRtt4L9YZg.png)

Also, i must configure the azure credentials to my azure container registry so it can push the image there :

![](https://cdn-images-1.medium.com/max/2000/1*5fRM8slInyFNvK7TeJaJ3g.png)

**My Jenkinsfile :**

    ```groovy
    pipeline{
      agent any
      environment{
        AZURE_REPO='myprivaterepo.azurecr.io'
      }
      stages{
        stage("build image"){
          steps{
            sh "docker build -t myprivaterepo.azurecr.io/library-management-api ./api/"
            echo "image built successfully"
          }
        }
        stage("push image to ACR"){
          steps{
            echo "pushing to ACR stage"
            withCredentials([usernamePassword(credentialsId: 'azure-credentials', passwordVariable: 'PASS', usernameVariable: 'USER')]){
              sh "echo $PASS | docker login ${AZURE_REPO} --username $USER --password-stdin"
              sh "nslookup myprivaterepo.azurecr.io"
              sh "docker push myprivaterepo.azurecr.io/library-management-api"
            }
            echo "image pushed successfully to azure container registry"
          }
        }
      }
    }
    ```

Then i got this error :

![](https://cdn-images-1.medium.com/max/2000/1*77hZT545K2J4PDsai_1QGQ.png)

This error was solved by adding sh.exe to my PATH .

Now our pipeline is running successfully, it is triggered by branch commits, rebuilds docker image every time and pushes it to Azure Container Registry :

![](https://cdn-images-1.medium.com/max/2000/1*n8KbgOCWOZZPsh3np6ofuw.png)

### Azure MySQL Database :

All of our application data is held in Azure Database for MySQL Databases, Azure gives us this service for free under certain Conditions :

![](https://cdn-images-1.medium.com/max/2000/1*Dnjz_7or09GMK_dSd0x3fg.png)

We create flexible server :

![](https://cdn-images-1.medium.com/max/2000/1*3uyxdG352qac0kbvwIaxaQ.png)

Create a new database :

![](https://cdn-images-1.medium.com/max/2000/1*ud_IGz7hDDAZWW21_3upjw.png)

And it is all set up . One more thing to talk about is the continuous Deployment .

### Argo CD :

![Argo CD](https://cdn-images-1.medium.com/max/2000/0*PydwW6ZxkZ63DrHS.png)_Argo CD_

Argo CD is a Kubernetes-native continuous deployment (CD) tool. Unlike external CD tools that only enable push-based deployments, Argo CD can pull updated code from Git repositories and deploy it directly to Kubernetes resources. It enables developers to manage both infrastructure configuration and application updates in one system.

### GitOps with Argo CD :

GitOps is a software engineering practice that uses a Git repository as its single source of truth. Teams commit declarative configurations into Git, and these configurations are used to create environments needed for the continuous delivery process. There is no manual setup of environments and no use of standalone scripts ÔÇö everything is defined through the Git repository.

A basic part of the GitOps process is a pull request. New versions of a configuration are introduced via pull request, merged with the main branch in the Git repository, and then the new version is automatically deployed. The Git repository contains a full record of all changes, including all details of the environment at every stage of the process.

We set up a new project and we link it to our git repository and specify where the k8s manifests are located :

![](https://cdn-images-1.medium.com/max/2000/1*jEfhuusB3c5TheioAe4eIA.png)

Now we can see our resources deployed successfully to Azure Kubernetes Cluster :

![](https://cdn-images-1.medium.com/max/2000/1*yRANH4ZifzrGU-wrcuResg.png)

> Just a reminder, ArgoCD is installed on the cluster itself and is running as a LoadBalancer type service

### Testing :

Testing our external api :

![](https://cdn-images-1.medium.com/max/2000/1*cMScL5YIau6wjZuI7_FFFg.png)

We test if the books are set :

![](https://cdn-images-1.medium.com/max/2000/1*Rq9JjkuhITWC7eZbu_VxPQ.png)

And finally we check the users :

![](https://cdn-images-1.medium.com/max/2000/1*E_5-iqf6fd4OO3hQdLMuUA.png)

Surely, we can't find them because we haven't added yet .

To conclude, we learnt in this article about pipelines and GitOps, the way to deploy one and the tools needed for that .

Was this helpful ? Confusing ? If you have any questions, feel free to comment below ! Make sure to follow on Linkedin : [https://www.linkedin.com/in/malekzaag/](https://www.linkedin.com/in/malekzaag/) and github : [https://github.com/Malek-Zaag](https://github.com/Malek-Zaag) if youÔÇÖre interested in similar content and want to keep learning alongside me!
