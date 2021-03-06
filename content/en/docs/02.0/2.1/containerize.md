---
title: "2.1 Containerize an application"
linkTitle: "2.1 Containerize an application"
weight: 21
sectionnumber: 2.1
description: >
  Containerize an application.
---

The main goal of this lab is to show you how to containerize a Java application. Including deployment on OpenShift and exposing the service with a route. In this example we want to build a microservice based on [Quarkus](https://quarkus.io/), which produces random data when it’s REST interface is called. Another Quarkus microservice consumes then the data and exposes it on it’s own endpoint.

```
+----------+                    +----------+
| producer +<-------------------+ consumer +
+----------+                    +----------+
```

Some words about Quarkus:

{{% alert  color="primary" %}}
“Quarkus is a Kubernetes Native Java stack tailored for GraalVM & OpenJDK HotSpot, crafted from the best of breed Java libraries and standards. Also focused on developer experience, making things just work with little to no configuration and allowing to do live coding.” - [quarkus.io](https://quarkus.io/)
{{% /alert %}}

In short, Quarkus brings a framework built upon JakartaEE standards to build microservices in the Java environment. Per default Quarkus comes with full CDI integration, RESTeasy-JAX-RS, dev mode and many more features.

If you wanna know more about quarkus, you can checkout our [Puzzle Quarkus Techlab](https://puzzle.github.io/quarkus-techlab/)


## Task {{% param sectionnumber %}}.1: Setup Project

Prepare a new OpenShift project

```bash
oc new-project producer-consumer-userXY
```


## Task {{% param sectionnumber %}}.2: Inspect Dockerfile

First we need a Dockerfile. You can find the `Dockerfile` in the root directory of the example Java application
[Git Repository](https://gitea.techlab.openshift.ch/APPUiO-AMM-Techlab/example-spring-boot-helloworld).
The base image is a `uay.io/quarkus/centos-quarkus-maven:20.1.0-java11` which is pre configured for Quarkus Maven builds.

Because the build needs a huge amount of memory (>8GB) and takes a lot of time (+5min) we refrain building the app from source. Instead we take the pre built Quarkus app from the Github release page.

```Dockerfile
FROM registry.access.redhat.com/ubi8/ubi
WORKDIR /work/

#Install wget
RUN yum install wget -y

#Fetch the latest binary release from the GutHub release page
RUN wget https://github.com/puzzle/amm-techlab/releases/download/1.0.0/application

RUN chmod -R 775 /work
EXPOSE 8080

#Run the application
CMD ["./application", "-Dquarkus.http.host=0.0.0.0"]

```

[source](https://gitea.techlab.openshift.ch/APPUiO-AMM-Techlab/example-spring-boot-helloworld/raw/branch/master/Dockerfile)


Here is the full example of the  Dockerfile for building the application from source.
In this example we make use of the Docker Multistage builds. In the first stage we use the centOS Quarkus image and perform a Quarkus nativ build. The resulting binary will be used in the second build stage. For the second stage we use the UBI minimal image.

```Dockerfile
## Stage 1 : build with maven builder image with native capabilities
FROM quay.io/quarkus/centos-quarkus-maven:20.1.0-java11 AS build
COPY pom.xml /usr/src/app/
RUN mvn -f /usr/src/app/pom.xml -B de.qaware.maven:go-offline-maven-plugin:1.2.5:resolve-dependencies
COPY src /usr/src/app/src
USER root
RUN chown -R quarkus /usr/src/app
USER quarkus
RUN mvn -f /usr/src/app/pom.xml -Pnative clean package

## Stage 2 : create the docker final image
FROM registry.access.redhat.com/ubi8/ubi-minimal
WORKDIR /work/
COPY --from=build /usr/src/app/target/*-runner /work/application

# set up permissions for user `1001`
RUN chmod 775 /work /work/application \
  && chown -R 1001 /work \
  && chmod -R "g+rwX" /work \
  && chown -R 1001:root /work

EXPOSE 8080
USER 1001

CMD ["./application", "-Dquarkus.http.host=0.0.0.0"]
```

[source](https://gitea.techlab.openshift.ch/APPUiO-AMM-Techlab/example-spring-boot-helloworld/raw/branch/master/Dockerfile.complete)


## Task {{% param sectionnumber %}}.3: Create BuildConfig

The [BuildConfig](https://docs.openshift.com/container-platform/4.5/builds/understanding-buildconfigs.html) describes how a single build task is performed. The BuildConfig is primary characterized by the Build strategy and its resources. For our build we use the Docker strategy which invokes the Docker build command. Furthermore it expects a `Dockerfile` in the source repository. If the Dockerfile is not in the root directory, then you can specify the location with the `dockerfilePath`.
Beside we configure the source and the triggers as well. For the source we can specify any Git repository. This is where the application sources resides. The triggers describe how to trigger the build. In this example we provide four different triggers. (Generic webhook, GitHub webhook, ConfigMap change, Image change)

Prepare a file inside your workspace `<workspace>/buildConfig.yaml` and add the following resource configuration:

```YAML
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  annotations:
    openshift.io/generated-by: OpenShiftNewBuild
  labels:
    build: data-producer
    application: amm-techlab
  name: data-producer
spec:
  output:
    to:
      kind: ImageStreamTag
      name: data-producer:rest
  postCommit: {}
  resources:
    limits:
      cpu: "500m"
      memory: "512Mi"
    requests:
      cpu: "250m"
      memory: "512Mi"
  source:
    git:
    # Non Puzzle Link
      uri: https://github.com/schlapzz/quarkus-techlab-data-producer.git
      ref: rest
    type: Git
  strategy:
    dockerStrategy:
      dockerfilePath: src/main/docker/Dockerfile.binary
    type: Docker
  triggers:
  - github:
      secret: PPMUkybOXqfoY_bJd-ou
    type: GitHub
  - generic:
      secret: f31PWzHXBGI9iYw-fTli
    type: Generic
  - type: ConfigChange
  - imageChange: {}
    type: ImageChange
```

[source](https://raw.githubusercontent.com/puzzle/amm-techlab/master/manifests/02.0/2.1/buildConfig.yaml)

Create the build config.

```BASH
oc apply -f buildConfig.yaml
```

```
buildconfig.build.openshift.io/data-producer created
```

## Task {{% param sectionnumber %}}.4: Create ImageStreams

Next we need to configure an [ImageStream](https://docs.openshift.com/container-platform/4.5/openshift_images/image-streams-manage.html) for the output Image. The ImageStream is an abstraction for referencing images from within OpenShift Container Platform. Simplified the ImageStream tracks changes for the defined images and reacts by triggering a new Build.


```YAML
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  annotations:
    openshift.io/generated-by: OpenShiftNewBuild
  labels:
    build: data-producer
    application: amm-techlab
  name: data-producer
spec:
  lookupPolicy:
    local: false
status:
  dockerImageRepository: ""
```

[source](https://raw.githubusercontent.com/puzzle/amm-techlab/master/manifests/02.0/2.1/imageStream.yaml)

Let's create the ImageStream

<!-- Wiso wird hier keine Datei erstellt sondern GithubLink -->
<!-- Hier wird create verwendet und nicht apply wie zuvor?-->
```BASH
oc create -f https://raw.githubusercontent.com/puzzle/amm-techlab/master/manifests/02.0/2.1/imageStream.yaml
```

```
imagestream.image.openshift.io/data-producer created
```


## Task {{% param sectionnumber %}}.5: Deploy Application

After the ImageStream definition we can setup our Deployment.

```YAML
apiVersion: v1
kind: DeploymentConfig
metadata:
  annotations:
    image.openshift.io/triggers: '[{"from":{"kind":"ImageStreamTag","name":"data-producer:rest"},"fieldPath":"spec.template.spec.containers[?(@.name==\"data-producer\")].image"}]'
  labels:
    application: amm-techlab
  name: data-producer
spec:
  replicas: 1
  selector:
    deploymentConfig: data-producer
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        application: amm-techlab
        deploymentConfig: data-producer
    spec:
      containers:
        - image: data-producer
          imagePullPolicy: Always
          livenessProbe:
            failureThreshold: 5
            httpGet:
              path: /health
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 3
            periodSeconds: 20
            successThreshhold: 1
            timeoutSeconds: 15
          readinessProbe:
            failureThreshold: 5
            httpGet:
              path: /health
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 3
            periodSeconds: 20
            successThreshold: 1
            timeoutSeconds: 15
          name: data-producer
          ports:
          - containerPort: 8080
            name: http
            protocol: TCP
          resources:
            limits:
              cpu: "1"
              memory: 200Mi
            requests:
              cpu: 50m
              memory: 100Mi
  triggers:
    - imageChangeParams:
        automatic: true
        containerNames:
          - data-producer
        from:
          kind: ImageStreamTag
          name: data-producer:rest
      type: ImageChange
    - type: ConfigChange
```

[source](https://raw.githubusercontent.com/puzzle/amm-techlab/master/manifests/02.0/2.1/deploymentConfig.yaml)

Let's create the deployment with following command

```BASH
oc create -f https://raw.githubusercontent.com/puzzle/amm-techlab/master/manifests/02.0/2.1/deploymentConfig.yaml
```

```
deployment/data-producer created
```

When you check your project in the web console (Developer view) the example app is visible.
The pod will be deployed successfully when the build finishes and the application image is pushed to the image stream. Please note this might take several minutes.


## Task {{% param sectionnumber %}}.6: Create Service

Expose the container ports to the to the cluster with a Service. For the Service we configure the port `8080` for the Web API. We set the Service type to ClusterIP to expose the Service cluster internal only.

```YAML
apiVersion: v1
kind: Service
metadata:
  name: data-producer
  labels:
    application: amm-techlab
spec:
  ports:
  - name: 8080-tcp
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    deploymentConfig: data-producer
  sessionAffinity: None
  type: ClusterIP
```

[source](https://raw.githubusercontent.com/puzzle/amm-techlab/master/manifests/02.0/2.1/svc.yaml)

Create the Service in OpenShift

```bash
oc create -f https://raw.githubusercontent.com/puzzle/amm-techlab/master/manifests/02.0/2.1/svc.yaml
```

```
service/data-producer created
```


## Task {{% param sectionnumber %}}.7: Create Route

Create a Route to expose the service at a host name. This will make the application available outside of the cluster.

The TLS type is set to Edge. That will configure the router to terminate the SSL connection and forward to the service with HTTP.

```YAML
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  labels:
     application: amm-techlab
  name: data-producer
spec:
  port:
    targetPort: 8080-tcp
  to:
    kind: Service
    name: data-producer
    weight: 100
  tls:
    termination: edge
  wildcardPolicy: None
```

[source](https://raw.githubusercontent.com/puzzle/amm-techlab/master/manifests/02.0/2.1/route.yaml)


Create the Route in OpenShift

```bash
oc create -f https://raw.githubusercontent.com/puzzle/amm-techlab/master/manifests/02.0/2.1/route.yaml
```

```
route.route.openshift.io/data-consumer created
```


## Task {{% param sectionnumber %}}.8: Verify deployed resources

Now we can list all resources in our project to double check if everything is up und running.
Use the following command to display all resources within our project.

```BASH
oc get all
```

```
{{< highlight text "hl_lines=" >}}
NAME                         READY   STATUS      RESTARTS   AGE
pod/data-producer-1-build    0/1     Completed   0          4m4s
pod/data-producer-1-deploy   0/1     Completed   0          2m44s
pod/data-producer-1-h4bwj    1/1     Running     0          2m41s

NAME                                    DESIRED   CURRENT   READY   AGE
replicationcontroller/data-producer-1   1         1         1       2m44s

NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/data-producer   ClusterIP   172.30.211.87   <none>        8080/TCP   3m54s

NAME                                               REVISION   DESIRED   CURRENT   TRIGGERED BY
deploymentconfig.apps.openshift.io/data-producer   1          1         1         config,image(data-producer:rest)

NAME                                           TYPE     FROM       LATEST
buildconfig.build.openshift.io/data-producer   Docker   Git@rest   1

NAME                                       TYPE     FROM          STATUS     STARTED         DURATION
build.build.openshift.io/data-producer-1   Docker   Git@838be5c   Complete   4 minutes ago   1m21s

NAME                                           IMAGE REPOSITORY                                                                              TAGS   UPDATED
imagestream.image.openshift.io/data-producer   image-registry.openshift-image-registry.svc:5000/producer-consumer-hanelore15/data-producer   rest   2 minutes ago

NAME                                     HOST/PORT                                                         PATH   SERVICES        PORT       TERMINATION   WILDCARD
route.route.openshift.io/data-producer   data-producer-producer-consumer-hanelore15.techlab.openshift.ch          data-producer   8080-tcp   edge          None

{{< / highlight >}}
```


## Task {{% param sectionnumber %}}.9: Access application by browser

Finally you can visit your application with the URL provided from the Route: <https://data-producer-producer-consumer-userXY.techlab.openshift.ch/data>

{{% alert  color="primary" %}}Replace **userXY** with your username or get the url from your route.{{% /alert %}}

When you open the url you should see the producers data

```json
{"data":0.6681209742895893}
```

If you just see `Your new Cloud-Native application is ready!`, then you forget to append the `/data`path to the url


## Task {{% param sectionnumber %}}.10: Deploy consumer application

Now it's time to deploy the counterpart. Use following command to deploy the data-consumer application. The consumer application consists of three resource definitions. First we create a deployment, pointing to the consumer Docker Image on docker Hub. Next the service which exposes the application inside our cluster and last the route to access the app from outside the cluster.

<!-- The Producer is a DeploymentConfig and the Consumer is a Deloyment -->
<!-- Example explanation:
Consumer and Producer are different Kind of Templates. The Producer is a DeploymentConfig and the Consumer a Deployment. The Producer needs to be a DeploymentConfig to make full use of the build process with imagestreams. Because the Consumer uses a external Image and doesn't need these features we can follow the best pracices and use the Kubernetes compatible Deployment
 -->
```BASH
oc create -f  https://raw.githubusercontent.com/puzzle/amm-techlab/master/manifests/02.0/2.1/consumer.yaml
```

```
deployment.apps/data-consumer created
service/data-consumer created
route.route.openshift.io/data-consumer created
```

Let's verify if everything was deployed and is up running.

```s
oc get all
```


```
NAME                                 READY   STATUS             RESTARTS   AGE
pod/data-consumer-7f44cc5647-q2hqc   1/1     Running            0          4m38s
pod/data-producer-1-build            0/1     Completed          0          10m
pod/data-producer-1-deploy           0/1     Completed          0          9m33s
pod/data-producer-1-h4bwj            1/1     Running            0          9m30s

NAME                                    DESIRED   CURRENT   READY   AGE
replicationcontroller/data-producer-1   1         1         1       9m33s

NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/data-consumer   ClusterIP   172.30.22.100   <none>        8080/TCP   4m38s
service/data-producer   ClusterIP   172.30.211.87   <none>        8080/TCP   10m

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/data-consumer   1/1     1            1           4m38s

NAME                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/data-consumer-7f44cc5647   1         1         0       4m38s

NAME                                               REVISION   DESIRED   CURRENT   TRIGGERED BY
deploymentconfig.apps.openshift.io/data-producer   1          1         1         config,image(data-producer:rest)

NAME                                           TYPE     FROM       LATEST
buildconfig.build.openshift.io/data-producer   Docker   Git@rest   1

NAME                                       TYPE     FROM          STATUS     STARTED          DURATION
build.build.openshift.io/data-producer-1   Docker   Git@838be5c   Complete   10 minutes ago   1m21s

NAME                                           IMAGE REPOSITORY                                                                              TAGS   UPDATED
imagestream.image.openshift.io/data-producer   image-registry.openshift-image-registry.svc:5000/producer-consumer-hanelore15/data-producer   rest   9 minutes ago

NAME                                     HOST/PORT                                                         PATH   SERVICES        PORT       TERMINATION   WILDCARD
route.route.openshift.io/data-consumer   data-consumer-producer-consumer-hanelore15.techlab.openshift.ch   /      data-consumer   <all>      edge/Allow    None
route.route.openshift.io/data-producer   data-producer-producer-consumer-hanelore15.techlab.openshift.ch          data-producer   8080-tcp   edge          None
```

## Solution
<!-- The Folder isn't present -->
When you were not successful, you can update your project with the solution.
The needed resource files are available inside the folder *manifests/02.0/2.1/*.

To apply the Solution execute this command:

```s
oc apply -f manifests/02.0/2.1/
```
