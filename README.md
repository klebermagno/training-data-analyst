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

## Knative


## Clud Run
 Cloud Run is based on open-source Knative Serving, which makes apps more portable.
 
●  Rapid deployment of serverless applications.
● Automatic scaling up to support millions of requests and down to zero instances to reduce consumption when it’s not needed.
● Routing and network programming for applications.
● Point-in-time snapshots of deployed code and configurations that simplify managing rollouts and rollbacks.
● Cloud Run offers a stable HTTPS endpoint with TLS termination handled for you, and the option to map your services to your own domains.
● Cloud Run abstracts away all infrastructure management by automatically scaling up and down from zero almost instantaneously, depending on traffic. Cloud Run only charges you for the exact resources you use.
● Local emulator available in gcloud, Cloud Code
● Use a single command to build, push, and deploy.  The command “gcloud run deploy” uses either your Dockerfile or buildpacks as the source to build your container
● Cloud Run is a regional resource, replicated across zones.
● Projects might contain multiple Services tied to different regions.
● Google Load Balancing is used to scale globally using a serverless network endpoint group.
● Google Load Balancer integrates with other services, such as Google Cloud Armor, Cloud DNS, and Cloud CDN.

### Cloud Run for Anthos offers the same serverless experience that you get with Cloud Run. 
 
```
apiVersion: serving.knative.dev/v1 kind: Service
metadata:
name: helloworld-go
namespace: default spec:
 template:
  spec:
containers:
- image: gcr.io/knative-samples/helloworld-go
env:
- name: TARGET
value: "Go Sample v1"
 
```
## Kubernets

## Architecture

* Control plane: main part of the server with:
  * etcd
  * kube-scheduler: responsible for scheduling 
  * kube-cloud-manager: controllers that interact with underlying cloud providers. bringing in Google Cloud features like load balancers and storage volumes
  * kube-controller-manager: monitor state of the cluster
* Node: Use kubelet agent   

Default configuration 1 control plane and 3 notes per zone. Multiples zones multiply this.

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

# Anthos

Anthos is a hybrid and multi-cloud application platform offered by Google Cloud Platform (GCP). Anthos provides a consistent platform for deploying and managing containerized applications across multiple clouds, including GCP, other public clouds, and on-premises data centers.

Anthos includes various tools and services that enable you to build, deploy, and manage your applications in a consistent manner, regardless of the underlying infrastructure. Some of the key components of Anthos include:

Google Kubernetes Engine (GKE): Anthos is built on top of GKE, which is a fully managed Kubernetes service provided by GCP. GKE provides a consistent, secure, and scalable platform for deploying and managing containerized applications.
Anthos Config Management: This tool enables you to manage and enforce policies across multiple Kubernetes clusters and across multiple clouds, ensuring consistency in your application environment.
Anthos Service Mesh: This tool provides traffic management, observability, and security for microservices-based applications running on Kubernetes.
Cloud Run for Anthos: This service allows you to deploy serverless workloads on Anthos, providing a fully managed platform for running stateless containers without worrying about the underlying infrastructure.
Anthos enables you to deploy and manage your applications in a consistent manner across multiple clouds, while also providing a high degree of security, scalability, and flexibility.

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

### Datastore

 Hierarchically structured space similar to the directory structure of a file system
 Transactopnmal
 
### Filestore

### Big Table
NoSQL  wide-column database optimized for heavy reads and writes. On the other hand,

### Big Query
 BigQuery is an enterprise data warehouse for large amounts of relational structured data
You can set a Time ti live Table expiration time.
Is ACID Atomic, Consistent, Isolated, Durable.

### Google Storage

You can't change after select regional you can't change to multiregional ou inverse.

### Efficiency in processing large amounts of data

#### Cache-Aside
Load data on demand into a cache from a data store. This can improve performance and also helps to maintain consistency between data held in the cache and data in the underlying data store.
If an application updates information, it can follow the write-through strategy by making the modification to the data store, and by invalidating the corresponding item in the cache.

#### Materialized View
Prepopulated views over the data in one or more tables when the data isn't ideally structured for required query operations. This can improve querying and data extraction performance.

#### Sharding
Split Tables  into a set of horizontal partitions or shards. This improves scalability.

### Signed URLK
 give time-limited access to a specific Cloud Storage resource
 
 ### Spinnaker
 Spinnaker is an open source, multi-cloud continuous delivery platform for releasing software changes.

* Automate deployment pipelines that run integration and system tests, spin up and down server groups, and monitor your rollouts via git events, Jenkins, Travis CI, Docker, CRON, or other Spinnaker pipelines.

* Deploy across multiple cloud providers including AWS EC2, Kubernetes, Google Compute Engine, Google Kubernetes Engine, Google App Engine, Microsoft Azure, Openstack, Cloud Foundry, and Oracle Cloud Infrastructure

* Create and deploy immutable images

### Stackdriver Logs

 log sink to copy log.
 
* Stackdriver Profiler is a statistical, low-overhead profiler that continuously gathers CPU usage and memory-allocation information from your production applications. So it meets our requirements because it helps to identify the parts of the application consuming the most resources and the performance characteristics of the code.

* Stackdriver Trace is a tracing system that collects latency data and displays it in near real-time in the Google Cloud Platform Console.


### Anthos migration tool

Scaling
Patching
Upgrades
Flexibility
Virtualization
OS config
Physical operation
Hardware

Supported sources: VMware, AWS, Azure, Compute Engine
Supported Workloads OS types: Linux and Windows distributions

Application architectures:
● Web/application servers
● Stateless web frontends
● Business logic middleware (such as Tomcat, J2EE, COTS)
● Multi-VM, multi-tier stacks (such as LAMP, WordPress)
● Small/medium-sized databases (such as MySQL, PostgreSQL)

Badfit 
●  VMs with special kernel drivers (such as kernel mode NFS)
● Dependencies on specific hardware
● Software with licenses tied to certain hardware ID registration
● Always-on database VMs that don’t fit Cloud SQL
● HW-specific license constraints (such as per CPU)
● 32-bitOS
● File servers
● VM-based workloads that would require whole-node capacity, such as
high-performance, high-memory databases (SAP HANA)
● Single-instance workloads that rely on Compute Engine live migration to meet
high availability requirements


### StratoZone 

### Migration setup and planning

1. Discovery & assessment

● What applications do I have running?
● What are the best candidates for migration?

Stratozone
Fit report

2. Setup & planning
● Where will I run my migrated applications?
● How can I scale my migration process?

Migration plan (yaml)

3. Migrate & modernize
● How do I perform an actual migration?
● What if I need to modify or customize a migrated application?
 
 Docker file
 Deployment yaml
 Data volumes
 
4. Optimize
How do I handle the ongoing maintenance or upgrades of a migrated application?

 
Steps to assess a workload with the fit assessment tool
  1.
Use ssh to connect to the VM, and download the tool:
curl -O
https://anthos-migrate-release.storage.googleapis.com/v1.10.2/linux/amd64/mfit-linux- collect.sh
curl -O https://anthos-migrate-release.storage.googleapis.com/v1.10.2/linux/amd64/mfit
  2. ./mfit-linux-collect.sh
Collect machine information in a downloadable tar file:
  3. Create an assessment report in an HTML, JSON, or CSV file:
./mfit assess sample m4a-collect.tar --format json > monolith-mfit-report.json
 
 add migration source
 migctl source create local-vmware local-vmware-src \ --vc '1.2.3.4' \
--username 'admin'

create a migration plan
migctl migration create my-migration \ --source local-vmware-src \
--vm-id My_VMware_VM \
--intent Image
 
 customize plan
 migctl migration get my-migration
 
 generate container
 migctl migration generate- artifacts my-migration
 
 deploy GKE
  kubectl apply -f app.yaml
  
### CI/CD

Google Build can build and test code.

``` cloudbuild.yaml
steps:
- name: 'gcr.io/cloud-builders/npm'
args: ['install']
- name: 'gcr.io/cloud-builders/npm'
args: [test']
- name: 'gcr.io/cloud-builders/docker'
args: ['build', '-t', 'gcr.io/$PROJECT_ID/helloworld', '.'] - name: 'gcr.io/cloud-builders/docker'
args: ['push', 'gcr.io/$PROJECT_ID/helloworld']
```
 
 
 ### Google Cloud deply 
 When you use Google Cloud Deploy, you can deploy a container to any of the supported compute platforms that are compatible with your containerized application.

Google Cloud Deploy integrates with various compute platforms in GCP such as Google Kubernetes Engine (GKE), App Engine flexible environment, Compute Engine, and Cloud Run.
Fully managed service

```
steps:
- name: 'gcr.io/cloud-builders/npm'
args: ['install']
- name: 'gcr.io/cloud-builders/npm'
args: [test']
- name: 'gcr.io/cloud-builders/docker'
args: ['build', '-t', 'gcr.io/$PROJECT_ID/helloworld', '.'] - name: 'gcr.io/cloud-builders/docker'
args: ['push', 'gcr.io/$PROJECT_ID/helloworld'] - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
entrypoint: gcloud args: [
        "beta", "deploy", "releases", "create", "rel-\${SHORT_SHA}",
        "--delivery-pipeline", "helloworld-pipeline",
        "--region", "us-central1",
        "--annotations", "commitId=\${REVISION_ID}",
        "--images", "helloworld=gcr.io/$PROJECT_ID/helloworld"
        ]

 ```
 
