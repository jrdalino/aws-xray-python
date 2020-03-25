# myproject-aws-xray-eks-python

## Step 1: Modify IAM Role
- Using AWS CLI, add the following policy to the worker node's IAM role
```
$ aws iam attach-role-policy --role-name $ROLE_NAME \
--policy-arn arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess
```

## Step 2: Deploy X-Ray as a DaemonSet, Validate and View logs
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

## Step 3: Instrument Front End React Application

## Step 4: Instrument Python Flask Application using X-Ray SDK for Python

## Step 5: Deploy Application

## Step 6: Examine and Traces X-Ray Console
- View Service Map
- View Traces > Trace Overview
- View Traces > Trace Details

## Step 7: Instrument DynamoDB

## Step 8: API Gateway Integration

## Step 9: App Mesh Integration

## Step 10: CloudTrail Integration

## Step 11: Cloudwatch Integration

## Step 12: AWS Config Integration

## Step 13: Elastic Load Balancer Integration

## References
- https://eksworkshop.com/intermediate/245_x-ray/x-ray-daemon/
- https://docs.aws.amazon.com/xray/latest/devguide/aws-xray.html
- https://aws.amazon.com/blogs/compute/application-tracing-on-kubernetes-with-aws-x-ray/
