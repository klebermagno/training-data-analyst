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

## Buildpack

Buildpack build images to run in google envoriment or localy.

```
pack build --builder=gcr.io/buildpacks/builder sample-java-mvn
```

## Deployment strategies

* Recreate: Version A is terminated then version B is rolled out.
* Ramped (also known as rolling-update or incremental): Version B is slowly rolled out and replacing version A.
* Blue/Green: Version B is released alongside version A, then the traffic is switched to version B.
* Canary: Version B is released to a subset of users, then proceed to a full rollout.
* A/B testing: Version B is released to a subset of users under specific condition.
* Shadow: Version B receives real-world traffic alongside version A and doesn’t impact the response.
