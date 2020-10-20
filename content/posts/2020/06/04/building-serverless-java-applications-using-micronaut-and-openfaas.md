---
title: "Building serverless Java applications using Micronaut and OpenfaaS"
date: 2020-06-04
authors: ["Paulien van Alst"]
summary: When working on microservices in the Java ecosystem, especially using Spring (Boot), you will notice the long start-up time that applications will have, let alone the high memory consumption they will have. The overhead of each single microservice will eventually take its costs on the system. Looking at a framework such as Micronaut could help out to reduce this overhead without loosing any developer's productivity. Not only "classic" applications can be built using Micronaut, but also serverless applications, functions, can be built and deployed on a cloud environment or on Kubernetes using OpenFaaS. Let's dive into it!
images:
  - posts/2020/06/04/images/cover-img.jpg
caption: Image by Tayeb MEZAHDIA from Pixabay
---
# Building serverless Java applications using Micronaut and OpenFaaS

## Introduction

When working on microservices in the Java ecosystem, especially using Spring (Boot), you will notice the long start-up time that applications will have, let alone the high memory consumption they will have. The overhead of each single microservice will eventually take its costs on the system. Looking at a framework such as Micronaut could help out to reduce this overhead without loosing any developer's productivity. Not only "classic" applications can be built using Micronaut, but also serverless applications, functions, can be built and deployed on a cloud environment or on Kubernetes using OpenFaaS. Let's dive into it!

Small side note: during this article some references and comparisons are made with Spring. Some knowledge about Spring can therefore be useful.

## Introducing Micronaut

Micronaut is a JVM based framework designed for building modular microservices. When you open a Micronaut project, you won't be surprised at first sight, it will even look and feel the same as the common Spring (Boot) projects in the Java world.

However, the differences are far to be small or subtle and should be very well understood before start taking such projects into production. On the other hand, starting discovering the Micronaut world will go very smoothly and the understanding will grow along as you are hacking your way through it.

### Annotations and Micronaut

Where Spring is known to use a lot of reflection during runtime, Micronaut is doing the same work but compile time. One example is the Spring data queries, that are generated runtime. Micronaut will generate those same queries compile time minimizing the used memory at runtime.

Each Bean created will be enriched compile time creating a so-called `BeanDefinition` class containing the bean's requirement and it's constructor. All these classes are processed using the `BeanDefinitionInjectProcessor`.

Also, Micronaut relies on the Java EE Dependency Injection, hence beans can be annotated with `@Singleton` and can be used with `@Inject`. The lifecycle of those beans will be managed by Micronaut itself.

## Building a Micronaut component

For everything that has to do with Micronaut, a CLI (Command Line Interface) tool can be used. This can be installed by simply following the steps in the official [documentation](https://docs.micronaut.io/latest/guide/index.html#buildCLI).

The CLI tool is a good starting point for any project using Micronaut and will generate the backbone structure of the project and some useful files for development. It does the entire scaffolding for you. For each application type a list of features can be set up to generate all the necessary configuration needed for those features.

The CLI tool supports three JVM languages: Java, Kotlin and Groovy and two build tools: Maven and Gradle. In case of a Maven project a Maven wrapper will be generated as well as a pom.xml.

Next to that, the CLI tool will generate a `Micronaut-cli.yml`, this will be the input for the CLI tool for any further operations and will contain the project's name and profile.

Coming from the Spring world, you won't be surprised to find the following in the main/resources directory: an `application.yml`. This file contains, just as it will in a Spring application, all the configuration settings for the application.

## Let's start

During this article, we will be building the following application:

![](../images/bookstore-arch.png#image-center-50)

A simple application will serve a book given an author. To do so, it will call a function that returns all the books of an author.

### Building an application

In this example, the application will contain a REST endpoint to ask for a book of a particular author. This endpoint will have to a call a function to retrieve this information. This application will therefore be a http-server as well as a http-client and will be deployed on Kubernetes.

Therefore the command to generate the basis of the application will be:

```bash

mn create-app bookstore-service --build=Maven --lang=java --features=Kubernetes,netflix-hystrix,http-server,http-client

```

Because the service calls a function, netflix-hystrix is needed. A function needs some time to start-up (warming up time). This does not mean it will take a lot of time, but the HTTP call will always take some time for sure. To avoid the function returning a HTTP response with status 500 straightaway, a retry mechanism is needed to make sure an answer is retrieved as soon as it becomes available.

As the application will be deployed on Kubernetes as indicated with the feature `Kubernetes`, the `create-app` command will also generate a `k8s.yml`, which will serve as a basis for the deployment. Of course this will have to be adjusted to the requirements of the environment where it will be deployed to. This Kubernetes configuration will have a deployment and a service.

By default, such an application will be running on port 8080. To change that the following property in the application.yml can be set to the preferred value:

```yaml

micronaut:

  server:

    port: 8081

```

Now it's time to add an endpoint to retrieve all the books given a certain author. For that, the HTTP functionality of Micronaut will be used.

```java

package bookstore.service.store;

 

import io.Micronaut.http.annotation.Controller;

import io.Micronaut.http.annotation.Get;

import io.Micronaut.http.annotation.PathVariable;

 

@Controller("books")

public class BookstoreController {

 

  @Get("/{author}")

  public Book retrieveBooksByAuthor() {

    return new Book("1000 new things", "John Doe");

  }

 

}

```

This will look a lot like a Spring Controller, right? Except that the annotations are used from the Micronaut package.

Remarkable here is the implementation of the `Book` DTO:

```java

@Introspected

public class Book implements Serializable {

 

  private String title;

  private String author;

 

  public Book(String title, String author) {

    this.title  = title;

    this.author = author;

  }

 

  // some getters

}

```

The `@Introspected` annotation which is needed for reflection free DTO's. At compile time, a check will be performed to see if all the properties can be initialized for the DTO.

It is interesting to have a look at the compiled files for this object.

### Building a function

A function will be built to serve data that needs to be served quickly to our application without using a lot of logic. In our example, the function will be returning a list of books given an author's name. Let's start with the most simple case:

```bash

mn create-function get-books-by-author --build=Maven --features=openfaas

```

Note: to be able to see all the features available for a function use: `mn create-function --help`

To make the function buildable, one AWS dependency has to be removed (in this example no AWS functionality will be used and it will return class path errors):

At the time of writing this was still generated for a function running on OpenFaaS.

```xml

    <dependency>

      <groupId>com.amazonaws</groupId>

      <artifactId>aws-lambda-java-log4j2</artifactId>

      <version>1.0.0</version>

      <scope>runtime</scope>

    </dependency>

```

At time of writing, the Dockerfile will still have a bug. The Dockerfile is based on the OpenJDK 8 image, but it should be based on th e OpenJDK 13 image to not encounter any runtime/compilation problems.

In the following line of code in the Dockerfile, two flags have to be removed:

`ENV fprocess="java -Dcom.sun.management.jmxremote -noverify -XX:TieredStopAtLevel=1 -XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -jar Handler.jar"`

`-noverify` and `-XX:+UseCGroupMemoryLimitForHeap` are deprecated in JDK 13 and are therefore not needed. The line of code will then become:

`ENV fprocess="java -Dcom.sun.management.jmxremote -XX:TieredStopAtLevel=1 -XX:+UnlockExperimentalVMOptions -jar Handler.jar"

Next we need to add `log4j2.xml` as configuration of the log4 logging.

Also the following must be added for the logging in the shade jar:

```xml

<transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">

  <mainClass>${exec.mainClass}</mainClass>

  <manifestEntries>

    <Multi-Release>true</Multi-Release>

  </manifestEntries>

</transformer>

```

Now, for convenience, the logging level will only be set to ERROR, by changing the fprocess command in the Dockerfile a bit more:

`ENV fprocess="java -Dorg.apache.logging.log4j.simplelog.StatusLogger.level=ERROR -Dcom.sun.management.jmxremote -XX:TieredStopAtLevel=1 -XX:+UnlockExperimentalVMOptions -jar Handler.jar"`

To keep the function as simple as possible, the function class won't be extending the `FunctionInitializer`

## Deploying Micronaut

Now an environment is needed to deploy this. OpenFaaS is a good option for deploying an application or function on Kubernetes. For a Micronaut function, OpenFaaS will deploy a pod. On this pod nothing will be running until the endpoint is called. At that point an application will start running and return the endpoint call.

### Installing OpenFaaS

Installing OpenFaaS is quite easy and should not take too much time. When working on my project, I wanted to explore the Micronaut functions. However, deploying to AWS was not an option and my curiosity was triggered by OpenFaaS. Installing OpenFaaS locally is quite easy.

OpenFaaS makes it easy to deploy functions and applications on an existing Kubernetes cluster.

Instructions for the installation are listed [here](https://docs.openfaas.com/deployment/Kubernetes/).

In a nutshell, start with installing the CLI:

```bash

brew install faas-cli

```

Then before installing OpenFaaS, install Arkade (which will be fastest option to install OpenFaaS later on):

```bash

curl -SLsf https://dl.get-arkade.dev/ | sudo sh

```

Now you can finally install OpenFaaS on your local Kubernetes cluster:

```bash

arkade install openfaas

```

Now the trick is not to ignore all the lines of logging that will come out of the installation, there are some useful instructions there that will help you finish the installation. To start with checking whether all the deployments where successful:

```bash

kubectl -n openfaas get deployments -l "release=openfaas, app=openfaas"

```

The successful deployments should be:

```

NAME                READY   UP-TO-DATE   AVAILABLE   AGE

alertmanager        1/1     1            1           6d21h

basic-auth-plugin   1/1     1            1           6d21h

faas-idler          1/1     1            1           6d21h

gateway             1/1     1            1           6d21h

nats                1/1     1            1           6d21h

prometheus          1/1     1            1           6d21h

queue-worker        1/1     1            1           6d21h

```

Now one final step has to be taken. The Gateway used by OpenFaaS should be forwarded:

```bash

kubectl -n openfaas rollout status deploy/gateway

kubectl -n openfaas port-foward svc/gateway 8080:8080

```

Do not forget setup the login:

```bash

PASSWORD=$(kubectl get secret -n openfaas basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode; echo)

echo -n $PASSWORD | faas-cli login --username admin --password-stdin

```

OpenFaaS should now be all up and running!

### Deploying a Micronaut function

Testing the function on OpenFaaS:

OpenFaaS requires a registry to push and deploy the function. For local development, the registry could be started within a docker container:

```bash

sudo docker run -d -p 5000:5000 --name registry registry:2

```

Looking more closely to the `function.yml`, after the provider description (which will only specify the endpoint of the gateway), the function is described as a docker image that would have to be ran:

```yaml

provider:

  name: openfaas

  gateway: http://127.0.0.1:8080

functions:

  get-books-by-author:

    lang: dockerfile

    handler: .

    image: localhost:5000/get-books-by-author:latest

```

The image in this example is prefixed with `localhost:5000/` which is the local registry that was launched by the previous bash command.

Now let's use the magical commands of the `faas-cli` to deploy and run the function:

```bash

faas-cli build -f function.yml

faas-cli push -f function.yml

faas-cli deploy -f function.yml

```

Now call the function through the OpenFaaS Gateway:

```bash

curl -X GET http://127.0.0.1:8080/function/get-books-by-author -H 'Content-Type: application/json' -d $'{"name":"Piet"}'

```

To verify that the function is running:

```bash

kubectl -n openfaas-fn get pods

```

Now this function has a running pod, but it is not running as such. The pod will be put up as a placeholder for the function. Once a REST call is made to the endpoint, the application will be started. This is done with [Watchdog](https://docs.openfaas.com/architecture/watchdog/).

# Resources

Everything in this article was developed with the following environment:

<div class="list-margin">

- Mac OSX  Catalina
- Docker Desktop (Engine v.19.03.5, Kubernetes v1.15.5)
- OpenJDK 13.0.2

</div>

References:

<div class="list-margin">

- [Official Micronaut documentation](http://www.micronaut.io)
- [Official OpenFaaS documentation](https://www.openfaas.com/)
- [Java Documentation on Micronaut's BeanDefinition class](https://docs.micronaut.io/latest/api/io/micronaut/inject/BeanDefinition.html)

</div>