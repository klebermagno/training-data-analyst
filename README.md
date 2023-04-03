# training-data-analyst
Labs and demos for courses for GCP Training (http://cloud.google.com/training).


## Lab App Dev - Developing a Backend Service: Java
[training-data-analyst/courses/developingapps/v1.3/java/pubsub-languageapi-spanner/](https://github.com/klebermagno/training-data-analyst/tree/master/courses/developingapps/v1.3/java/pubsub-languageapi-spanner)


## OpenAPI

OpenAPI is a open patern to specify an API. You can use a YAML file to describle and documentate it.

(https://cloud.google.com/api-gateway/docs/openapi-overview) 


## Cloud Foundation Toolkit 
(https://github.com/terraform-google-modules/terraform-example-foundation)

## Google Cloud Toolkit CLI

From Cloud Shell, enable the Cloud Run API:
```
gcloud services enable run.googleapis.com
```

Set the compute region:

```
gcloud config set compute/region us-central1
```

Use the one-step gcloud command to deploy your container:
```
gcloud beta run deploy --source .
```

Enable API Services:
```
gcloud services list --available
gcloud services enable SERVICE_NAME
```

Create a service account (to run services):

```
gcloud iam service-accounts create pubsub-cloud-run-invoker \
   --display-name "Order Initiator"
```

Bind the permission to a service:
```
gcloud run services add-iam-policy-binding order-service \
  --member=serviceAccount:pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com \
  --role=roles/run.invoker --platform managed
```

Grant permission to create token
```
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
   --member=serviceAccount:service-$PROJECT_NUMBER@gcp-sa-pubsub.iam.gserviceaccount.com \
   --role=roles/iam.serviceAccountTokenCreator
```

## Cloud Pub/Sub


In Google Cloud Pub/Sub, push and pull are two different mechanisms for delivering messages to subscribers.

**Push** delivery is a mechanism where the Pub/Sub service delivers messages to an **HTTP/HTTPS endpoint** that is specified by the subscriber. The endpoint could be hosted on a Google Cloud service, such as Cloud Functions or App Engine, or on an external service outside of Google Cloud. With push delivery, subscribers do not need to actively poll the Pub/Sub service for new messages. Instead, the Pub/Sub service pushes the messages to the subscriber's endpoint as soon as they are available. This can lead to lower latency and more efficient use of resources.

**Pull** delivery is a mechanism where subscribers actively poll the Pub/Sub service for new messages. In pull delivery, the **subscriber must periodically send a request to the Pub/Sub service to check** if there are any new messages available. If there are new messages, the subscriber can then retrieve them from the service. Pull delivery is useful for scenarios where the subscriber cannot receive incoming traffic or when the subscriber needs more control over when and how to retrieve messages.

In summary, push delivery is more suitable for real-time or near real-time applications where low latency is important, while pull delivery is better for scenarios where a subscriber needs more control over message retrieval and processing.


Create a topic: 
```
gcloud pubsub topics create ORDER_PLACED
```

Create a subscription:
```
ORDER_SERVICE_URL=$(gcloud run services describe order-service \
   --region $LOCATION \
   --format="value(status.address.url)")
```
```
gcloud pubsub subscriptions create order-service-sub \
   --topic ORDER_PLACED \
   --push-endpoint=$ORDER_SERVICE_URL \
   --push-auth-service-account=pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```



## Buildpack

Buildpack build images to run in google envoriment or localy.

```
pack build --builder=gcr.io/buildpacks/builder sample-java-mvn
```

## Loadbalancer

Reserve a external IP
```
gcloud compute addresses create example-ip \
    --ip-version=IPV4 \
    --global
```

Display IP
```
gcloud compute addresses describe example-ip \
    --format="get(address)" \
    --global
```

Load balancers use a serverless Network Endpoint Group (NEG) backend to direct requests to a serverless Cloud Run service.
So, let's first create our serverless NEG for our serverless Python app created earlier in this lab.

To create a serverless NEG with a Cloud Run service, enter the following in Cloud Shell:
```
gcloud compute network-endpoint-groups create myneg \
   --region=$LOCATION \
   --network-endpoint-type=serverless  \
   --cloud-run-service=helloworld
```

Create a loadbalance service 
```
gcloud compute backend-services create mybackendservice \
    --global
```
And then add the serverless NEG as a backend to this backend service:
```
gcloud compute backend-services add-backend mybackendservice \
    --global \
    --network-endpoint-group=myneg \
    --network-endpoint-group-region=$LOCATION
```

Finally, create a URL map to route incoming requests to the backend service:
```
gcloud compute url-maps create myurlmap \
    --default-service mybackendservice
```

If you had more than one backend service, you could use host rules to direct requests to different services based on the host name, or you could set up path matchers to direct requests to different services based on the request path. None of this is necessary here because we have only created one backend service for this lab.

Now, create a target HTTP(S) proxy to route requests to your URL map:
```
gcloud compute target-http-proxies create mytargetproxy \
    --url-map=myurlmap
```
Create a global forwarding rule to route incoming requests to the proxy:
```
gcloud compute forwarding-rules create myforwardingrule \
    --address=example-ip \
    --target-http-proxy=mytargetproxy \
    --global \
    --ports=80
```    
### Configuring Engress from a Static Outbound IP Address [APPRUN]

Create a subnetwork called mysubnet with a range of 192.168.0.0/28 (CIDR format is required for this), in the region us-central1, to be used as the VPC for your Serverless VPC Access connector. Use the command below, which will create this subnetwork in the project's default network (note also that subnets used for VPC connectors must have a netmask of 28, or you will get an error later in the process):
```
gcloud compute networks subnets create mysubnet \
--range=192.168.0.0/28 --network=default --region=$LOCATION
``` 

Create a serverless VPC Access connector

To route your Cloud Run service's outbound traffic to a VPC network, you first need to set up a Serverless VPC Access connector.

Create a Serverless VPC Access connector named myconnector with your previously created subnetwork named mysubnet using the following command:
```
gcloud compute networks vpc-access connectors create myconnector \
  --region=$LOCATION \
  --subnet-project=$GOOGLE_CLOUD_PROJECT \
  --subnet=mysubnet
```
You may be asked to enable the vpcaccess.googleapis.com on your project. If so, enter y and wait for the process to complete before proceeding. 

Configure network address translation (NAT)

To route outbound requests to external endpoints through a static IP (which is the main goal of this lab), you must first configure a Cloud NAT gateway.

Create a new Cloud Router to program your NAT gateway:
```
gcloud compute routers create myrouter \
  --network=default \
  --region=$LOCATION
```
Next, reserve a static IP address using the command below, where myoriginip is the name being assigned to your IP address resource.
A reserved IP address resource retains the underlying IP address when the resource it is associated with is deleted and re-created. Using the same region as your Cloud NAT router will help to minimize latency and network costs.
```
gcloud compute addresses create myoriginip --region=$LOCATION
Copied!
content_copy
To route outbound requests to external endpoints through a static IP, you must configure a Cloud NAT gateway.
```
Bring all of the resources you've just made together to create a Cloud NAT gateway named mynat. Use the command below to configure your router to route the traffic originating from the VPC network:
```
gcloud compute routers nats create mynat \
  --router=myrouter \
  --region=$LOCATION \
  --nat-custom-subnet-ip-ranges=mysubnet \
  --nat-external-ip-pool=myoriginip
 ```
 
 Route Cloud Run traffic through the VPC network

After NAT has been configured, you will deploy your Cloud Run service with the Serverless VPC Access connector and set the VPC egress to route all traffic through the VPC network:
```
gcloud run deploy sample-go \
   --image=gcr.io/$GOOGLE_CLOUD_PROJECT/sample-go \
   --vpc-connector=myconnector \
   --vpc-egress=all-traffic
```
You may be asked to enable run.googleapis.com for your project. If so, respond y and wait for the API to enable.

When asked for a region, choose us-central1 as the location "us-central1" is created in the environment variable and also choose y to allow unauthenticated invocations. It will take a few minutes for the process to complete.

### Trafic migration cloud run. 25% each node

Traffic migration - deploy a new version

When splitting traffic between two or more revisions, a comma separated list can be used. The list represents the revisions deployed.

Deploy a new tagged revision (test3) with redirection of traffic:
```
gcloud run deploy product-service \
  --image gcr.io/qwiklabs-resources/product-status:0.0.3 \
  --no-traffic \
  --tag test3 \
  --region=$LOCATION \
  --allow-unauthenticated
 ```
Deploy a new tagged revision (test4) with redirection of traffic:
```
gcloud run deploy product-service \
  --image gcr.io/qwiklabs-resources/product-status:0.0.4 \
  --no-traffic \
  --tag test4 \
  --region=$LOCATION \
  --allow-unauthenticated
```

Output a list of the revisions deployed:
```
gcloud run services describe product-service \
  --region=$LOCATION \
  --format='value(status.traffic.revisionName)'
```
Create an environment variable for the available revisionNames:
```
LIST=$(gcloud run services describe product-service --platform=managed --region=$LOCATION --format='value[delimiter="=25,"](status.traffic.revisionName)')"=25"
```
Split traffic between the four services using the LIST environment variable:
``` 
gcloud run services update-traffic product-service \
  --to-revisions $LIST --region=$LOCATION
```
Observe the name of the Tagged Revision in the command output.

Test the endpoint is distributing traffic:
```
for i in {1..10}; do curl $TEST1_PRODUCT_SERVICE_URL/help -w "\n"; done
```

## Deployment strategies

* Blue/Green: Version B is released alongside version A, then the traffic is switched to version B.
* Canary: Version B is released to a subset of users, then proceed to a full rollout.
* Recreate: Version A is terminated then version B is rolled out.
* Ramped (also known as rolling-update or incremental): Version B is slowly rolled out and replacing version A.
* A/B testing: Version B is released to a subset of users under specific condition.
* Shadow: Version B receives real-world traffic alongside version A and doesn’t impact the response.

## Kubernets


## Security 
On Google Kubernetes Engine (GKE) is recommended to use *Workload Identity*. Use gcloud to bind the Google service accounts and the Kubernetes service accounts using roles/iam.workloadIdentityUser.

Not use service account attached to the GKE node. While it could work, all the services are using the same service account, there is no separation of permissions, and no detailed logging.

### Workload Identity
Workload Identity is a feature in Google Kubernetes Engine (GKE) that allows you to securely authenticate to Google Cloud services from within your GKE cluster without needing to manage and rotate service account keys.
With Workload Identity, you can configure your GKE cluster to use Kubernetes service account credentials to authenticate to Google Cloud services, instead of using service account keys. This provides an extra layer of security and simplifies the management of credentials.

### kubernets.ymal
``` (kubernets.ymal)
apiVersion: apps/v1 
kind: Pod 
metadata:
  name: nginx
  labels:
    app: nginx 
spec:
containers:
- name: nginx
    image: nginx:latest
```
### SSH


```
gcloud compute ssh ledgermonolith-service --tunnel-through-iap --zone=us-central1-b
```

### Commands

In Cloud Shell, configure access to your cluster for the kubectl command-line tool, using the following command:

```
gcloud container clusters get-credentials $my_cluster --zone $my_zone
```

Scale
```
kubectl scale --replicas=3 deployment nginx-deployment
```

To scalling canarian you can just scale to zero replicar the old container.
```
kubectl scale --replicas=0 deployment nginx-deployment
```

Get deploymenrts

```
kubectl get deployments
```
Get the history
```
kubectl rollout history deployment nginx-deployment
```

### deployment manifest

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

## Kind:

* Deployments: are usually used for stateless applications. you can save the state of deployment by attaching a Persistent Volume to it and make it stateful, but all the pods of a deployment will be sharing the same Volume and data across all of them will be same. 
* StatefulSets:  is a Kubernetes resource used to manage stateful applications. . It manages the deployment and scaling of a set of Pods, and provides guarantee about the ordering and uniqueness of these Pods. Every replica of a stateful set will have its own state, and each of the pods will be creating its own PVC(Persistent Volume Claim). So a statefulset with 3 replicas will create 3 pods, each having its own Volume, so total 3 PVCs.
* DaemonSets:  is a controller that ensures that the pod runs on all the nodes of the cluster. If a node is added/removed from a cluster, DaemonSet automatically adds/deletes the pod.
* Jobs: 
### To deploy your manifest, execute the following command:

```
kubectl apply -f ./nginx-deployment.yaml
```


### Rollout & Rollback

* Rollout:  A feature rollout enables you to safely deliver a new feature to your users by controlling who sees it, and when.
* Rollback: Back to the previous version.

### Rollout
You can set rollout by set command or update yaml

```
kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.9.1 --record
```

### Auto-scaling

* Horizontal scaling in cloud computing means adding additional instances instead of moving to a larger instance size.
* Vertical scaling refers to adding more or faster CPUs, memory, or I/O resources to an existing server, or replacing one server with a more powerful server.


### Storage
 
 [/ak8s/v1.1](https://github.com/klebermagno/training-data-analyst/tree/master/courses/ak8s/v1.1)

gsutil is a command-line tool used to manage Cloud Storage

bq is a command-line tool for BigQuery


# Cloud Endpoints 

Cloud Endpoints is a service offered by Google Cloud Platform (GCP) that allows you to create, deploy, protect, and monitor APIs for your applications. It provides a scalable and secure way to build APIs that can be accessed by external applications and developers.

With Cloud Endpoints, you can define and deploy APIs using the OpenAPI specification, also known as Swagger. You can use this specification to define the API's methods, parameters, request and response formats, authentication and authorization mechanisms, and other metadata.

Cloud Endpoints also provides various security features to protect your APIs, such as API key validation, JWT authentication, and OAuth 2.0 authorization. It also integrates with Google Cloud Identity and Access Management (IAM), allowing you to control access to your APIs at a granular level.

In addition, Cloud Endpoints provides a monitoring dashboard that allows you to track API usage, performance, and errors. You can use this dashboard to identify and troubleshoot issues in your APIs and to gain insights into the usage patterns of your API consumers.

Overall, Cloud Endpoints is a powerful and flexible service that enables you to create and manage APIs for your applications in a scalable and secure manner. It allows you to focus on building your applications' business logic while providing the infrastructure for your APIs.

# Identity-Aware Proxy

Provide only autentication.
HTTP Status Codes

### SRE
SRE is the management of availability, latency, performance, efficiency, change management, monitoring, emergency response, and capacity planning regarding the software development process. GCP has a set of tools called APPLICATION PERFORMANCE MANAGEMENT: Stackdriver Trace, Stackdriver Debugger, and Stackdriver Profiles.
