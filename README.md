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
### Configuring Egress from a Static Outbound IP Address [APPRUN]

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


## Deployment strategies

* Blue/Green: Version B is released alongside version A, then the traffic is switched to version B.
* Canary: Version B is released to a subset of users, then proceed to a full rollout.
* Recreate: Version A is terminated then version B is rolled out.
* Ramped (also known as rolling-update or incremental): Version B is slowly rolled out and replacing version A.
* A/B testing: Version B is released to a subset of users under specific condition.
* Shadow: Version B receives real-world traffic alongside version A and doesn’t impact the response.

## Kubernets

# Commands

```
$
```

# deployment manifest

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


