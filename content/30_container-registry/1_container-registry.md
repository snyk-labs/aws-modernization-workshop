+++
title = "Container Registry"
chapter = false
weight = 1
+++

## Container Registry

### Expected Outcome:
* Push your containers to Amazon Elastic Container Registry

**Average Lab Time:** 10-15 minutes

### Introduction
In this module, we're going to take our already containerized applications and push them to [Amazon Elastic Container Registry (ECR)](https://aws.amazon.com/ecr/)

#### Deploy ECR Repositories
We will first deploy our CloudFormation template which configures our ECR Repositories

**Step 1:** Change to this modules directory by running:
```
cd ~/environment/aws-modernization-workshop/modules/container-registry
```

**Step 2:** Deploy the CloudFormation template using the `aws cli` tool. 
```
aws cloudformation create-stack \
  --stack-name "petstore-ecr" \
  --template-body file://petstore-ecr-cf-resources.yaml \
  --capabilities CAPABILITY_NAMED_IAM
```

**Step 3:** Wait for the Template to finish deploying
```
until [[ `aws cloudformation describe-stacks --stack-name "petstore-ecr" --query "Stacks[0].[StackStatus]" --output text` == "CREATE_COMPLETE" ]]; do  echo "The stack is NOT in a state of CREATE_COMPLETE at `date`";   sleep 30; done && echo "The Stack is built at `date` - Please proceed"
```

##### Pushing the Petstore images to Amazon Elastic Container Registry (ECR)
Before we can deploy our Petstore application to an orchestrator, we need to push our Petstore images to https://aws.amazon.com/ecr/[Amazon Elastic Container Registry (ECR)]. 

Amazon Elastic Container Registry (ECR) is a fully-managed Docker container registry that makes it easy for developers to store, manage, and deploy Docker container images. Amazon ECR is integrated with Amazon Elastic Container Service (ECS), simplifying your development to production workflow.

**Step 1:** Log into your Amazon ECR registry using the helper provided by the AWS CLI.
```
eval $(aws ecr get-login --no-include-email)
```

***Step 2:** Use the AWS CLI to get information about the two Amazon ECR repositories that were created for you ahead of time. One repository will be for the Petstore PostgreSQL backend and the other will be for the Petstore web frontend.
```
aws ecr describe-repositories --repository-name petstore_postgres petstore_frontend
```

```
{
    "repositories": [
        {
            "registryId": "123456789012",
            "repositoryName": "petstore_postgres",
            "repositoryArn": "arn:aws:ecr:us-west-2:123456789012:repository/petstore_postgres",
            "createdAt": 1533757748.0,
            "repositoryUri": "123456789012.dkr.ecr.us-west-2.amazonaws.com/petstore_postgres"
        },
        {
            "registryId": "123456789012",
            "repositoryName": "petstore_frontend",
            "repositoryArn": "arn:aws:ecr:us-west-2:123456789012:repository/petstore_frontend",
            "createdAt": 1533757751.0,
            "repositoryUri": "123456789012.dkr.ecr.us-west-2.amazonaws.com/petstore_frontend"
        }
    ]
}
```

**Step 3:** Verify that your Docker images exist by running the docker images command.

```
docker images
```

```
REPOSITORY                          TAG                 IMAGE ID            CREATED             SIZE
containerize-application_petstore   latest              22f901adffb3        32 minutes ago      626MB
<none>                              <none>              9b961113e5bf        33 minutes ago      564MB
postgres                            9.6                 506063568b80        2 days ago          237MB
maven                               3.5-jdk-7           84a20ec9fab9        3 weeks ago         483MB
lambci/lambda                       nodejs4.3           6c30c5c1b1e0        6 weeks ago         969MB
lambci/lambda                       python2.7           377732dd7a1f        6 weeks ago         974MB
lambci/lambda                       python3.6           acf16b1d5297        6 weeks ago         1.1GB
lambci/lambda                       nodejs6.10          da301bf4fe34        6 weeks ago         1.02GB
jboss/wildfly                       11.0.0.Final        6926d48f2e5b        4 months ago        617MB
```

**Step 4:** Tag the local docker images with the locations of the remote ECR repositories we created using our `CloudFormation` template.
```
docker tag postgres:9.6 $(aws ecr describe-repositories --repository-name petstore_postgres --query=repositories[0].repositoryUri --output=text):latest
docker tag containerize-application_petstore:latest $(aws ecr describe-repositories --repository-name petstore_frontend --query=repositories[0].repositoryUri --output=text):latest
```

**Step 5:** Once the images have been tagged, push them to the remote repository.
```
docker push $(aws ecr describe-repositories --repository-name petstore_postgres --query=repositories[0].repositoryUri --output=text):latest
docker push $(aws ecr describe-repositories --repository-name petstore_frontend --query=repositories[0].repositoryUri --output=text):latest
```

You should see the Docker images being pushed with an output similar to this:
```
The push refers to repository [123456789012.dkr.ecr.us-west-2.amazonaws.com/petstore_postgres]
7856d1f55b98: Pushed
a125032aca95: Pushed
fcfc309521a9: Pushed
4c4e9f97ac56: Pushed
109402c6a817: Pushed
6663c6c0d308: Pushed
ed4da41a79a9: Layer already exists
7c050956ab95: Layer already exists
c6fcee3b341c: Layer already exists
998e6abcfae7: Layer already exists
df9515382700: Layer already exists
0fae9a7d0574: Layer already exists
add4404d0b51: Layer already exists
cdb3f9544e4c: Layer already exists
latest: digest: sha256:ca39b6107978303706aac0f53120879afcd0d4b040ead7f19e8581b81c19ecea size: 3243
```

With the images pushed to Amazon ECR we are ready to deploy them to our orchestrator.
