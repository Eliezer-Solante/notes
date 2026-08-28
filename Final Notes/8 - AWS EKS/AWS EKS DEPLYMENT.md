Deploying EKS Service: https://github.com/kodekloudhub/amazon-elastic-kubernetes-service-course/blob/main/docs/deploy.md

Setup Access and Join Nodes
https://github.com/kodekloudhub/amazon-elastic-kubernetes-service-course/blob/main/docs/nodes.md

Sample Output after deploying EKS service:
```bash
NodeAutoScalingGroup = "eks-cluster-stack-NodeGroup-w17MY0d9sY7L"
NodeInstanceRole = "arn:aws:iam::637423470040:role/eksWorkerNodeRole"
NodeSecurityGroup = "sg-0cddc9524b455b181" 
```

NodeAutoScalingGroup = "eks-cluster-stack-NodeGroup-zAhfkff1303C"
NodeInstanceRole = "arn:aws:iam::471112703240:role/eksWorkerNodeRole"
NodeSecurityGroup = "sg-041b46b2210aaf925"

{
    "User": {
        "Path": "/",
        "UserName": "iamuser-eksuser",
        "UserId": "AIDAW3MEBLUEIXVX2WF2P",
        "Arn": "arn:aws:iam::471112703240:user/iamuser-eksuser",
        "CreateDate": "2026-08-28T11:41:12Z"
    }
}
{
    "AccessKey": {
        "UserName": "iamuser-eksuser",
        "AccessKeyId": "AKIAW3MEBLUEIA3J5ZO2",
        "Status": "Active",
        "SecretAccessKey": "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",
        "CreateDate": "2026-08-28T11:41:39Z"
    }
}

![[Pasted image 20260828150516.png]]