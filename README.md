# myproject-aws-xray-eks-python

## Step 1: Add Policy to the Worker Node IAM Role
- Using AWS CLI, add the following policy to the worker node's IAM role
```
$ aws iam attach-role-policy --role-name $ROLE_NAME \
--policy-arn arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess
```

## Step 2: Prepare the DemonSet
-  Create a folder and download the daemon
```
$ mkdir xray-daemon && cd xray-daemon
$ curl https://s3.dualstack.ap-southeast-2.amazonaws.com/aws-xray-assets.ap-southeast-2/xray-daemon/aws-xray-daemon-linux-2.x.zip -o ./aws-xray-daemon-linux-2.x.zip
$ unzip -o aws-xray-daemon-linux-2.x.zip -d .
```
- Create a Dockerfile with the following content.
```
FROM ubuntu:12.04
COPY xray /usr/bin/xray-daemon
CMD xray-daemon -f /var/log/xray-daemon.log &
```
-  Build the image and Push to new ECR with name: myproject-xray-daemon-ecr
```
$ $(aws ecr get-login --no-include-email --region ap-southeast-2)
$ docker build -t bp-xray-daemon-ecr .
$ docker tag myproject-xray-daemon-ecr:latest 222337787619.dkr.ecr.ap-southeast-2.amazonaws.com/myproject-xray-daemon-ecr:latest
$ docker push 222337787619.dkr.ecr.ap-southeast-2.amazonaws.com/myproject-xray-daemon-ecr:latest
```

## Step 3: Deploy X-Ray as a DaemonSet, Validate and View logs
- Deploy the X-Ray DaemonSet
```
$ kubectl create -f https://github.com/jrdalino/myproject-aws-xray-eks-python/blob/master/xray-k8s-daemonset.yaml
```
- To see the status of the X-Ray DaemonSet
```
$ kubectl describe daemonset xray-daemon
```
- View logs for all the X-Ray demon pods
```
$ kubectl logs -l app=xray-daemon
```

## Step 4: Instrument Front End React Application

## Step 5: Instrument Python Flask Application using X-Ray SDK for Python

## Step 6: Deploy Application

## Step 7: Examine and Traces X-Ray Console
- View Service Map
- View Traces > Trace Overview
- View Traces > Trace Details

## Step 8: Instrument DynamoDB

## Step 9: API Gateway Integration

## Step 10: App Mesh Integration

## Step 11: CloudTrail Integration

## Step 12: Cloudwatch Integration

## Step 13: AWS Config Integration

## Step 14: Elastic Load Balancer Integration

## References
- https://eksworkshop.com/intermediate/245_x-ray/x-ray-daemon/
- https://docs.aws.amazon.com/xray/latest/devguide/aws-xray.html
- https://aws.amazon.com/blogs/compute/application-tracing-on-kubernetes-with-aws-x-ray/
