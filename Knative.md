---
title: Knative Workshop Guide
layout: page
---

# References

1. [Knative Slide Deck](https://docs.google.com/presentation/d/1wJ5HN7qeaUVZp_ZcLQMjDkP7p2CpejLzJhz3CG3DIwk/edit?usp=sharing)


## 1. Installing Knative on PKS

[Knative with PKS](https://github.com/knative/docs/blob/master/install/Knative-with-PKS.md)

### Install Istio

Either Install 0.4.1 (or 0.4.0) or install the latest available version.

```bash
kubectl apply --filename https://github.com/knative/serving/releases/download/v0.4.1/istio-crds.yaml && \
kubectl apply --filename https://github.com/knative/serving/releases/download/v0.4.1/istio.yaml
```

```bash
kubectl apply --filename https://storage.googleapis.com/knative-releases/serving/latest/istio-crds.yaml && \
kubectl apply --filename https://storage.googleapis.com/knative-releases/serving/latest/istio.yaml
```

*Note: You might have to deploy this twice, to get istio deployed.*

Label the default namespace with istio-injection=enabled:

```bash
kubectl label namespace default istio-injection=enabled
```

Watch if all the pods are created

```bash
kubectl get pods --namespace istio-system
```

### Install Knative

Either Install 0.4.1 (or 0.4.0) or install the latest available version.

```bash
kubectl apply --filename https://github.com/knative/serving/releases/download/v0.4.0/serving.yaml \
--filename https://github.com/knative/build/releases/download/v0.4.0/build.yaml \
--filename https://github.com/knative/eventing/releases/download/v0.4.0/release.yaml \
--filename https://github.com/knative/eventing-sources/releases/download/v0.4.0/release.yaml \
--filename https://github.com/knative/serving/releases/download/v0.4.0/monitoring.yaml \
--filename https://raw.githubusercontent.com/knative/serving/v0.4.0/third_party/config/build/clusterrole.yaml
```

For the latest version, use the following.

```bash
kubectl apply --filename https://storage.googleapis.com/knative-releases/serving/latest/serving.yaml
kubectl apply --filename https://storage.googleapis.com/knative-releases/build/latest/build.yaml
kubectl apply --filename https://storage.googleapis.com/knative-releases/eventing/latest/release.yaml
kubectl apply --filename https://storage.googleapis.com/knative-releases/eventing-sources/latest/release.yaml
kubectl apply --filename https://storage.googleapis.com/knative-releases/serving/latest/monitoring.yaml
kubectl apply --filename https://raw.githubusercontent.com/knative/serving/v0.4.1/third_party/config/build/clusterrole.yaml
```

### Apply BuildTemplates

Buildtemplate for the Build CRD, note the buildpack templates is not currently available. (_Need to update_)

```bash
kubectl apply -f https://raw.githubusercontent.com/knative/build-templates/master/kaniko/kaniko.yaml
kubectl apply -f https://raw.githubusercontent.com/knative/build-templates/master/buildpack/buildpack.yaml
```

### Reserve a static IP address

[Assigning Static IP](https://github.com/knative/docs/blob/master/serving/gke-assigning-static-ip-address.md)

Assigning a static IP address for Knative on Kubernetes Engine
If you are running Knative on Google Kubernetes Engine and want to use a custom domain with your apps, you need to configure a static IP address to ensure that your custom domain mapping doesn't break.

Knative configures an Istio Gateway CRD named knative-ingress-gateway under the knative-serving namespace to serve all incoming traffic within the Knative service mesh. The IP address to access the gateway is the external IP address of the "istio-ingressgateway" service under the istio-system namespace. Therefore, in order to set a static IP for the gateway you must to set the external IP address of the istio-ingressgateway service to a static IP.

```bash
gcloud beta compute addresses create knative-ip --region=us-east1
```

Update the external IP of istio-ingressgateway service

```bash
kubectl patch svc istio-ingressgateway --namespace istio-system --patch '{"spec": { "loadBalancerIP": "104.196.68.16" }}' service/istio-ingressgateway
```

You will have to do the above couple of times to make sure it propogates.

Alternatively, get the IP Address allocated by GCP

```bash
kubectl get svc istio-ingressgateway -ojson --namespace istio-system
```

Verify the static IP address of istio-ingressgateway service

```bash
kubectl get svc istio-ingressgateway  --namespace istio-system
```

## 2. Demo Knative Service

1. Create a hello world app using the `service.yaml`

   ```yaml
    apiVersion: serving.knative.dev/v1alpha1 # Current version of Knative
    kind: Service
    metadata:
    name: helloworld-go # The name of the app
    namespace: default # The namespace the app will use
    spec:
    runLatest:
        configuration:
        revisionTemplate:
            spec:
            container:
                image: gcr.io/knative-samples/helloworld-go # The URL to the image of the app
                env:
                - name: TARGET # The environment variable printed out by the sample app
                    value: "Go Sample v1"
   ```

2. Deploy the hello world knative app

   ```bash
   kubectl apply --filename service.yaml
   ```

   _output:_

        service.serving.knative.dev/helloworld-go created

3. Accesing your Knative Cluster

   ```bash
   kubectl get svc istio-ingressgateway --namespace istio-system
   export KNATIVE_INGRESS=$(kubectl get svc istio-ingressgateway --namespace istio-system --output 'jsonpath={.status.loadBalancer.ingress[0].ip}')
   echo $KNATIVE_INGRESS
   curl -H "Host: helloworld-go.default.example.com" http://$KNATIVE_INGRESS
   ```

   _output:_

        Hello Go Sample v1!

4. Create a Configuration with Revisions

    A Configuration is where you define your desired state for a deployment. At a minimum, this includes a Configuration name and a reference to the container image to deploy.

    Revisions represent immutable, point-in-time snapshots of code and configuration

    ```bash
        kubectl apply -f configuration.yaml
        kubectl get pods --all-namespaces
    ```

    _output:_

        default              knative-helloworld-xbqtm-deployment-cd674f574-dq7d2   3/3     Running     0          2m12s

5. Creating a Route and Sending Traffic

    A Route in Knative provides a mechanism for routing traffic to your running code. It maps a named, HTTP-addressable endpoint to one or more Revisions.

    ```bash
    kubectl apply -f route.yaml
    ```

    _output:_

        route.serving.knative.dev/knative-helloworld created`

    ```bash
    kubectl get routes -oname
    ```

    _output:_

        route.serving.knative.dev/helloworld-go
        route.serving.knative.dev/knative-helloworld

    ```bash
    kubectl get  route.serving.knative.dev/knative-helloworld -ojson
    ```

    _output:_

    ```yaml
        {
            "apiVersion": "serving.knative.dev/v1alpha1",
            "kind": "Route",
            "metadata": {
                "annotations": {
                    "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"serving.knative.dev/v1alpha1\",\"kind\":\"Route\",\"metadata\":{\"annotations\":{},\"name\":\"knative-helloworld\",\"namespace\":\"default\"},\"spec\":{\"traffic\":[{\"configurationName\":\"knative-helloworld\",\"percent\":100}]}}\n"
                },
                "creationTimestamp": "2019-03-19T14:11:47Z",
                "finalizers": [
                    "routes.serving.knative.dev"
                ],
                "generation": 1,
                "name": "knative-helloworld",
                "namespace": "default",
                "resourceVersion": "758179",
                "selfLink": "/apis/serving.knative.dev/v1alpha1/namespaces/default/routes/knative-helloworld",
                "uid": "eecb0096-4a50-11e9-b91c-4201c0a81413"
            },
            "spec": {
                "traffic": [
                    {
                        "configurationName": "knative-helloworld",
                        "percent": 100
                    }
                ]
            },
            "status": {
                "address": {
                    "hostname": "knative-helloworld.default.svc.cluster.local"
                },
                "conditions": [
                    {
                        "lastTransitionTime": "2019-03-19T14:11:48Z",
                        "status": "True",
                        "type": "AllTrafficAssigned"
                    },
                    {
                        "lastTransitionTime": "2019-03-19T14:11:51Z",
                        "status": "True",
                        "type": "IngressReady"
                    },
                    {
                        "lastTransitionTime": "2019-03-19T14:11:51Z",
                        "status": "True",
                        "type": "Ready"
                    }
                ],
                "domain": "knative-helloworld.default.example.com",
                "domainInternal": "knative-helloworld.default.svc.cluster.local",
                "observedGeneration": 1,
                "traffic": [
                    {
                        "percent": 100,
                        "revisionName": "knative-helloworld-ln5x8"
                    }
                ]
            }
        }
    ```

    This Route sends 100% of traffic to the latestReadyRevisionName of the Configuration specified in configurationName. You can test this Route and Configuration by issuing the following curl command:

    ```bash
    curl -H "Host: knative-helloworld.default.example.com" http://$KNATIVE_INGRESS
    ```

6. Kubernetes objects created by Knative

    ```bash
    kubectl get deployments -oname
    ```

    _output:_

        deployment.extensions/helloworld-go-x6hsz-deployment
        deployment.extensions/knative-helloworld-ln5x8-deployment`

    ```bash
    kubectl get replicasets -oname
    ```

    _output:_

        replicaset.extensions/helloworld-go-x6hsz-deployment-6577599f7f
        replicaset.extensions/knative-helloworld-ln5x8-deployment-76db6c76b4

    ```bash
    kubectl get pods -oname
    ```

    _output:_

        pod/knative-helloworld-00001-deployment-5f7b54c768-lrqt5`

    Now, watch the pods, configurations, revisions and routes in four separate terminal windows

    ```bash
    watch kubectl get pods --namespace=default
    watch kubectl get configurations
    watch kubectl get revisons
    watch kubectl get routes
    ```

    _output:_

    Configuration

        NAME                 LATESTCREATED              LATESTREADY                READY   REASON
        helloworld-go        helloworld-go-qw4sr        helloworld-go-qw4sr        True
        knative-helloworld   knative-helloworld-xbqtm   knative-helloworld-xbqtm   True

    Revision

        NAME                       SERVICE NAME                       AGE   READY   REASON
        helloworld-go-qw4sr        helloworld-go-qw4sr-service        20m   True
        knative-helloworld-xbqtm   knative-helloworld-xbqtm-service   11m   True

    Routes

        NAME                 DOMAIN                                   READY   REASON
        helloworld-go        helloworld-go.default.example.com        True
        knative-helloworld   knative-helloworld.default.example.com   True

    Pods

        NAME                                                  READY   STATUS        RESTARTS   AGE
        helloworld-go-qw4sr-deployment-5fc4785d96-2zs6k       3/3     Running       0          61s
        knative-helloworld-xbqtm-deployment-cd674f574-54f4j   2/3     Terminating   0          7m28s
        knative-helloworld-xbqtm-deployment-cd674f574-bbvjl   2/3     Terminating   0          3m49s
        knative-helloworld-xbqtm-deployment-cd674f574-zz8zs   3/3     Running       0          68s

    You will notice the pods will change the status from `Running` to `Terminating` if you don't have any traffic

7. Checking Autoscaler and Activator

   Serverless workloads should scale all the way down to zero. Knative uses two key components to achieve this functionality. It implements Autoscaler and Activator as Pods on the cluster.

   ```bash
    # Output of kubectl get pods on knative-serving

    knative-serving      activator-5f8c9678bd-89q5v                      2/2     Running     1          41m
    knative-serving      autoscaler-7486469d84-45vm8                     2/2     Running     1          41m
   ```

    The Autoscaler gathers information about the number of concurrent requests to a Revision. To do so, it runs a container called the queue-proxy inside the Revision’s Pod. The queue-proxy checks the observed concurrency for that Revi‐ sion. It then sends this data to the Autoscaler every one second. The Autoscaler evaluates these metrics every two seconds. Based on this evaluation, it increases or decreases the size of the Revision’s under‐ lying Deployment

    The Activator is a shared component that catches all traffic for Reserve Revisions. When it receives a request for a Reserve Revision, it transitions that Revision to Active. It then proxies the requests to the appropriate Pods.

    Check there are zero pods for both your deployments. 
    Now lets run some load test using `siege`. Install `siege` using `homebrew` if you don't have.

    ```bash
    siege -r 1 -c 50 -d 2 -v -H "Host: helloworld-go.default.example.com" http://$KNATIVE_INGRESS
    ```

    You will notice the activator and the autoscaler if now kicking in. 

    ```bash
    default              helloworld-go-qw4sr-deployment-5fc4785d96-jbwcv   3/3     Running     0          50s
    ```

    Tail the logs from the `user-container` in the pod

    ```bash
    kubectl logs -f helloworld-go-qw4sr-deployment-5fc4785d96-lqsn6 -c user-container

    # You will see a stream of logs as the siege kicks in the pods 
        2019/03/19 19:23:47 Hello world sample started.
        2019/03/19 19:23:49 Hello world received a request.
        2019/03/19 19:25:07 Hello world received a request.
        2019/03/19 19:25:07 Hello world received a request.

    ```

8. Blue-Green or Incremental Deployments (Zero Downtime) 

    [Routing and managing traffic with blue/green deployment](https://github.com/knative/docs/blob/master/docs/serving/samples/blue-green-deployment.md)

   ```bash
   kubectl apply -f blue-green-demo-config-v1.yaml
   ```

   This will deploy the blue version of an app by creating a revision and a configuration. 
   Get the revision name for this version

   ```bash
   kubectl get revisions
   ...
   blue-green-demo-76vs7      blue-green-demo-76vs7-service      17m   True
   ```

    Next, lets create a route for this blue version. 

    __*Note*__

    *Knative doesn't let you name revisions. Hence you have to modify this file and change the revision name.*  

    ```bash
    kubectl apply --filename blue-green-demo-route-v1.yaml
    ```

    This will create a route for the blue version.

    ```bash
    kubectl get routes
    ...
    blue-green-demo      blue-green-demo.default.example.com      True
    ```

    Now, lets curl it and get the response. 

    ```bash
    curl -H "Host: blue-green-demo.default.example.com" http://$KNATIVE_INGRESS
    ```

    ```html
    <!DOCTYPE html>
    <html lang="en">
    <head>
    <title>Knative Routing Demo</title>
    <link rel="stylesheet" type="text/css" href="/css/app.css" />
    </head>
    <body>
        
            <div class="blue">App v1</div>
        
    </div>
    </body>
    </html>
    ```

    Now, lets deploy the green version of the app. 

    ```bash
    kubectl apply -f blue-green-demo-config-v2.yaml
    ```

   This will deploy the green version of an app by creating a revision and a configuration. 
   Get the revision name for this version

   ```bash
   kubectl get revisions
   ...
   blue-green-demo-z68ng      blue-green-demo-z68ng-service      47m   True
   ```

    Next, lets create a route for this green version. 

    __*Note*__

    *Knative doesn't let you name revisions. Hence you have to modify this file and change the revision name.*  

    ```bash
    kubectl apply --filename blue-green-demo-route-v2.yaml
    ```

    This will create a route for the green version.

    ```bash
    kubectl get routes
    ...
    blue-green-demo      blue-green-demo.default.example.com      True
    ```

    Now, lets curl it and get the response. 

    ```bash
    curl -H "Host: blue-green-demo.default.example.com" http://$KNATIVE_INGRESS
    ```

    ```html
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <title>Knative Routing Demo</title>
        <link rel="stylesheet" type="text/css" href="/css/app.css" />
    </head>
    <body>
            
                <div class="green">App v2</div>
            
        </div>
    </body>
    </html>
    ```

9. Traffic Management

    We can manage traffic between different versions of the app, name the versions and do testing with the named version.

    ```yaml
    apiVersion: serving.knative.dev/v1alpha1
    kind: Route
    metadata:
    name: blue-green-demo # The name of our route; appears in the URL to access the app
    namespace: default # The namespace we're working in; also appears in the URL to access the app
    spec:
    traffic:
        - revisionName: blue-green-demo-76vs7
        percent: 50 # 50% traffic goes to this revision
        name: v2 # A named route

        - revisionName: blue-green-demo-z68ng
        percent: 50 # 50% traffic goes to this revision  
        name: v1 # A named route
    ```

    ```bash
    kubectl apply -f blue-green-demo-route-v3.yaml
    ```

    Now, lets curl it and get the response. 

    ```bash
    curl -H "Host: blue-green-demo.default.example.com" http://$KNATIVE_INGRESS
    ```

    ```html
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <title>Knative Routing Demo</title>
        <link rel="stylesheet" type="text/css" href="/css/app.css" />
    </head>
    <body>
            
                <div class="green">App v2</div>
            
        </div>
    </body>
    </html>
    ```

    Curl again, and you will see it load balances the traffic 50% of the time. 

    ```bash
    curl -H "Host: blue-green-demo.default.example.com" http://$KNATIVE_INGRESS
    ```

    ```html
    <!DOCTYPE html>
    <html lang="en">
    <head>
    <title>Knative Routing Demo</title>
    <link rel="stylesheet" type="text/css" href="/css/app.css" />
    </head>
    <body>
        
            <div class="blue">App v1</div>
        
    </div>
    </body>
    </html>
    ```

    You call call the named version, and it will directly invoke the specific named configuration/route.

    ```bash
    curl -H "Host: v1.blue-green-demo.default.example.com" http://$KNATIVE_INGRESS
    curl -H "Host: v2.blue-green-demo.default.example.com" http://$KNATIVE_INGRESS

    ```

10. Custom Domains

    [Setting up a custom domain](https://github.com/knative/docs/blob/master/docs/serving/using-a-custom-domain.md)

    By default, knative uses example.com domain. To configure the custom domain first update the config-domain

    ```bash
    kubectl apply --filename config-domain.yaml
    ```

    Now check if the changes to of the custom domain have propogated

    ```bash
    kubectl get route helloworld-go --output jsonpath="{.status.domain}"
    helloworld-go.default.gcp.rjainpcf.com
    ```

    Get the IP Address of the Istio ingress gateway and update the DNS 'A' record

    ```bash
    kubectl get svc istio-ingressgateway --namespace istio-system --output jsonpath="{.status.loadBalancer.ingress[*]['ip']}"
    ```

    Switch to Google Cloud DNS and add the A record

    ```html
    *.default.gcp.rjainpcf.com                 59     IN     A   35.196.142.161
    ```

    Now, curl the routes with the custom domain, and you will get the response.

    ```bash
    curl -H "Host: blue-green-demo.default.gcp.rjainpcf.com" http://$KNATIVE_INGRESS
    ```

11. Cleanup

    Cleanup all the knative services, routes, configurations, revisions, and deployment

    ```bash
    kubectl delete ksvc --all
    kubectl delete route -all
    kubectl delete configuration -all
    kubectl delete revision -all
    ```

## 3. Demo Knative Build

1. Create a simple Knative Build

    [Creating a simple Knative Build](https://github.com/knative/docs/blob/master/docs/build/creating-builds.md)

    In this build file, we are using a `busybox` to print "hello build"

    ```yaml
    apiVersion: build.knative.dev/v1alpha1
    kind: Build
    metadata:
    name: hello-build
    spec:
    steps:
    - name: hello
        image: busybox
        args: ["echo", "hello", "build"]
    ```

   ```bash
   kubectl apply -f hello-build.yaml

   watch kubectl get builds
   NAME          SUCCEEDED   REASON   STARTTIME   COMPLETIONTIME
   hello-build   Unknown     Pending   2s
   ...
   hello-build   True                 16s
   ```

   Get the details of the build

   ```bash
   kubectl get build hello-build --output yaml
   ```

   ```yaml
        apiVersion: build.knative.dev/v1alpha1
        kind: Build
        metadata:
        annotations:
            kubectl.kubernetes.io/last-applied-configuration: |
            {"apiVersion":"build.knative.dev/v1alpha1","kind":"Build","metadata":{"annotations":{},"name":"hello-build","namespace":"default"},"spec":{"steps":[{"args":["echo","hello","build"],"image":"busybox","name":"hello"}]}}
        creationTimestamp: "2019-03-20T02:14:58Z"
        generation: 1
        name: hello-build
        namespace: default
        resourceVersion: "71576"
        selfLink: /apis/build.knative.dev/v1alpha1/namespaces/default/builds/hello-build
        uid: f59c88bc-4ab5-11e9-80ba-4201c0a81415
        spec:
        Status: ""
        serviceAccountName: default
        steps:
        - args:
            - echo
            - hello
            - build
            image: busybox
            name: hello
            resources: {}
        timeout: 10m0s
        status:
        builder: Cluster
        cluster:
            namespace: default
            podName: hello-build-pod-483e6b
        conditions:
        - lastTransitionTime: "2019-03-20T02:15:07Z"
            status: "True"
            type: Succeeded
        startTime: "2019-03-20T02:14:59Z"
        stepStates:
        - terminated:
            containerID: docker://c5a32e9be79d998fb091a7f2240635ef1bdc0b41b1211b6ba920bbbbefab60e5
            exitCode: 0
            finishedAt: "2019-03-20T02:15:05Z"
            reason: Completed
            startedAt: "2019-03-20T02:15:05Z"
        stepsCompleted:
        - build-step-hello
   ```

   Verify if the build performed the single task

   ```bash
   kubectl logs $(kubectl get build hello-build --output jsonpath={.status.cluster.podName}) --container build-step-hello
   ```

   *Note* The docker build container is short lived. In case you don't get the response, delete the build, create it again and watch the container logs immediately

   ```bash
    kubectl delete build hello-build
    kubectl apply -f hello-build.yaml
    kubectl logs $(kubectl get build hello-build --output jsonpath={.status.cluster.podName}) --container build-step-hello
    ...
    hello build
   ```

2. Build, Build Template and Services

   [Build using GCR](https://github.com/GoogleCloudPlatform/knative-build-tutorials/tree/master/docker-build)
    Next, lets build a more complex build using kaniko build templates and publish the image to a image repo (gcr)

    Here is the build yaml file we are going to use, build-gcr.yaml

    ```yaml
        apiVersion: build.knative.dev/v1alpha1
        kind: Build
        metadata:
        name: docker-build
        spec:
        serviceAccountName: build-bot-gcr
        source:
            git:
            url: https://github.com/dgageot/hello.git
            revision: master
        steps:
        - name: build-and-push
            image: gcr.io/kaniko-project/executor:v0.1.0
            args:
            - --dockerfile=/workspace/Dockerfile
            - --destination=gcr.io/fe-rajain/hello-nginx
    ```

    First, we have to configure a service account on `gcloud`

    ```bash
       PROJECTID=$(gcloud config get-value project)
       #Create a Service Account
       gcloud iam service-accounts create knative-build --display-name "Knative Build"

       #Allow it to push to GCR
       gcloud projects add-iam-policy-binding $PROJECTID --member serviceAccount:knative-build@$PROJECTID.iam.gserviceaccount.com --role roles/storage.admin

       # This creates a knative-key.json file on your drive.
       gcloud iam service-accounts keys create knative-key.json --iam-account knative-build@$PROJECTID.iam.gserviceaccount.com

    ```

    Next, create a secret and a service account in `kubernetes`

    ```bash
    # Create a secret knative-build-auth
    kubectl create secret generic knative-build-auth --type="kubernetes.io/basic-auth" --from-literal=username="_json_key" --from-file=password=knative-key.json
    # Annotate the secret, to tell Build to use these parameters
    kubectl annotate secret knative-build-auth build.knative.dev/docker-0=https://gcr.io

    # Create a service account  build-bot-gcr
    kubectl apply -f build-bot-gcr.yaml

    ```

    Now we are ready to run the build

    ```bash
    kubectl apply -f build-gcr.yaml
    ```

    Check the build resources, and pod coming up. 

    ```bash
        Every 2.0s: kubectl get builds
        NAME           SUCCEEDED   REASON   STARTTIME   COMPLETIONTIME
        docker-build   True                 50m
    ```

    ```bash
        Every 2.0s: kubectl get pods --namespace=default
        NAME                      READY   STATUS      RESTARTS   AGE
        docker-build-pod-f1af81   0/1     Completed   0          50m
    ```

    Describe the build pod, to see the various steps and the containers it creates. 

    ```bash
      kubectl describe pod  docker-build-pod-f1af81
    ```

    Check on gcr.io image registry, and new hello-nginx folder and docker images are created in that. 

    ```bash
    gcloud beta container images describe gcr.io/fe-rajain/hello-nginx
    image_summary:
        digest: sha256:4d3a7ac7d0102e9a52eb9985b2889fd22fb2f9e46ab3a0d8ffcbd8adccdeaa2d
        fully_qualified_digest: gcr.io/fe-rajain/hello-nginx@sha256:4d3a7ac7d0102e9a52eb9985b2889fd22fb2f9e46ab3a0d8ffcbd8adccdeaa2d
        registry: gcr.io
        repository: fe-rajain/hello-nginx
    ```

    Next you can create a service using this image. 

    ```yaml
    apiVersion: serving.knative.dev/v1alpha1
    kind: Service
    metadata:
        name: knative-build-demo
        namespace: default
    spec:
        runLatest:
            configuration:
            build:
                serviceAccountName: build-bot-gcr
                source:
                git:
                    url: https://github.com/dgageot/hello.git
                    revision: master
                template:
                name: kaniko
                arguments:
                - name: IMAGE
                    value: gcr.io/fe-rajain/hello-nginx:latest
            revisionTemplate:
                spec:
                container:
                    image: gcr.io/fe-rajain/hello-nginx:latest
    ```

    ```bash
    kubectl apply -f service-build.yaml
    ```

    *Note* this is currrently failing, because of permission denied to pull from the private repo. Working on getting this configured and fixed

3. Cleanup

   ```bash
   kubectl delete builds --all
   kubectl delete ksvc --all
   
   ```


## 4. Demo Knative Eventing




