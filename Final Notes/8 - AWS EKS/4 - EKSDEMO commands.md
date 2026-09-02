Here's the full `eksdemo` command reference, organized by category. All install commands follow the pattern `eksdemo install <app> -c <cluster-name> [flags]`, and get commands follow `eksdemo get <resource> [flags]`.

## Cluster Management

```
eksdemo create cluster -c <cluster-name>      # create an EKS cluster (Bottlerocket nodes by default)
eksdemo delete cluster -c <cluster-name>
eksdemo version                                # check installed version
eksdemo completion bash|zsh                    # shell completion
```

## Install — Autoscaling & Compute

```
eksdemo install karpenter -c <cluster>
eksdemo install autoscaling cluster-autoscaler -c <cluster>
eksdemo install autoscaling keda -c <cluster>
eksdemo install autoscaling vpa -c <cluster>
eksdemo install metrics-server -c <cluster>
```

## Install — Networking & Load Balancing

```
eksdemo install aws-lb-controller -c <cluster>
eksdemo install ingress nginx -c <cluster>
eksdemo install ingress contour -c <cluster>
eksdemo install ingress emissary -c <cluster>
eksdemo install external-dns -c <cluster>
eksdemo install vpc-lattice-controller -c <cluster>
eksdemo install cilium -c <cluster>
```

## Install — Storage

```
eksdemo install storage ebs-csi -c <cluster>
eksdemo install storage efs-csi -c <cluster>
eksdemo install storage fsx-lustre-csi -c <cluster>
eksdemo install storage openebs -c <cluster>
```

## Install — Secrets Management

```
eksdemo install secrets store-csi-driver -c <cluster>
eksdemo install secrets store-csi-driver-provider-aws -c <cluster>
eksdemo install vault -c <cluster>
```

## Install — Monitoring & Observability

```
eksdemo install kube-prometheus stack -c <cluster>
eksdemo install kube-prometheus stack-amp -c <cluster>          # using Amazon Managed Prometheus
eksdemo install kube-prometheus karpenter-dashboards -c <cluster>
eksdemo install kube-state-metrics -c <cluster>
eksdemo install prometheus-node-exporter -c <cluster>
eksdemo install container-insights cloudwatch-agent -c <cluster>
eksdemo install container-insights fluent-bit -c <cluster>
eksdemo install container-insights adot-collector -c <cluster>
eksdemo install container-insights prometheus -c <cluster>
eksdemo install adot-operator -c <cluster>
eksdemo install aws-fluent-bit -c <cluster>
eksdemo install kubecost eks -c <cluster>
eksdemo install kubecost eks-amp -c <cluster>
eksdemo install kubecost vendor -c <cluster>
eksdemo install goldilocks -c <cluster>
eksdemo install k8sgpt-operator -c <cluster>
eksdemo install headlamp -c <cluster>
```

## Install — Security & Policy

```
eksdemo install cert-manager -c <cluster>
eksdemo install falco -c <cluster>
eksdemo install policy kyverno -c <cluster>
eksdemo install policy opa-gatekeeper -c <cluster>
eksdemo install core-dump-handler -c <cluster>
```

## Install — GitOps & CI/CD

```
eksdemo install argo cd -c <cluster>
eksdemo install argo workflows -c <cluster>
eksdemo install argo workflows-cognito -c <cluster>
eksdemo install flux controllers -c <cluster>
eksdemo install flux sync -c <cluster>
eksdemo install velero -c <cluster>
```

## Install — Service Mesh

```
eksdemo install istio base -c <cluster>
eksdemo install istio istiod -c <cluster>
eksdemo install consul -c <cluster>
```

## Install — AWS Controllers for Kubernetes (ACK)

```
eksdemo install ack ec2-controller -c <cluster>
eksdemo install ack ecr-controller -c <cluster>
eksdemo install ack efs-controller -c <cluster>
eksdemo install ack eks-controller -c <cluster>
eksdemo install ack iam-controller -c <cluster>
eksdemo install ack rds-controller -c <cluster>
eksdemo install ack s3-controller -c <cluster>
eksdemo install ack apigatewayv2-controller -c <cluster>
eksdemo install ack prometheusservice-controller -c <cluster>
```

## Install — Crossplane

```
eksdemo install crossplane core -c <cluster>
eksdemo install crossplane ec2-provider -c <cluster>
eksdemo install crossplane iam-provider -c <cluster>
eksdemo install crossplane s3-provider -c <cluster>
```

## Install — Registries & Misc Platform

```
eksdemo install harbor -c <cluster>
eksdemo install keycloak-amg -c <cluster>
eksdemo install spark-operator -c <cluster>
eksdemo install neuron neuron-device-plugin -c <cluster>
```

## Install — Example / Demo Apps

```
eksdemo install example game-2048 -c <cluster>
eksdemo install example inflate -c <cluster>          # triggers autoscaling events
eksdemo install example podinfo -c <cluster>
eksdemo install example wordpress -c <cluster>
eksdemo install example ghost -c <cluster>
eksdemo install example eks-workshop -c <cluster>
eksdemo install example kube-ops-view -c <cluster>
eksdemo install example spark-pi -c <cluster>
eksdemo install example ascp -c <cluster>              # Secrets Manager + Config Provider demo
```

## Get — Cluster & Kubernetes Resources

```
eksdemo get cluster
eksdemo get nodegroup
eksdemo get fargate-profile
eksdemo get node
eksdemo get addon
eksdemo get addon-versions
eksdemo get application                 # installed eksdemo applications
eksdemo get access-entry
```

## Get — Compute (EC2/ASG)

```
eksdemo get ec2-instance      # alias: ec2
eksdemo get auto-scaling-group
eksdemo get ami
eksdemo get elastic-ip
```

## Get — Networking (VPC)

```
eksdemo get vpc
eksdemo get vpc-summary
eksdemo get subnet
eksdemo get security-group
eksdemo get security-group-rule
eksdemo get network-interface
eksdemo get network-acl
eksdemo get network-acl-rule
eksdemo get route-table
eksdemo get internet-gateway
eksdemo get nat-gateway
eksdemo get vpc-endpoint
eksdemo get prefix-list
eksdemo get availability-zone
```

## Get — Load Balancing

```
eksdemo get load-balancer
eksdemo get listener
eksdemo get listener-rule
eksdemo get target-group
eksdemo get target-health
```

## Get — VPC Lattice

```
eksdemo get vpc-lattice service
eksdemo get vpc-lattice service-network
eksdemo get vpc-lattice target-group
```

## Get — Storage

```
eksdemo get volume              # EBS volumes
eksdemo get efs-filesystem
eksdemo get s3-bucket
```

## Get — IAM & Identity

```
eksdemo get iam-role
eksdemo get iam-policy
eksdemo get iam-oidc
```

## Get — DNS & Certificates

```
eksdemo get hosted-zone
eksdemo get dns-record
eksdemo get acm-certificate
```

## Get — Monitoring & Logs

```
eksdemo get log-group
eksdemo get log-stream
eksdemo get log-event
eksdemo get logs-insights query
eksdemo get logs-insights results
eksdemo get logs-insights stats
eksdemo get metric
eksdemo get alarm
eksdemo get amp-workspace
eksdemo get amp-rule
eksdemo get amg-workspace
```

## Get — Other AWS Resources

```
eksdemo get cloudformation-stack
eksdemo get cloudtrail-trail
eksdemo get cloudtrail-event
eksdemo get kms-key
eksdemo get sqs-queue
eksdemo get event-rule
eksdemo get ecr-repository
eksdemo get ssm-node
eksdemo get ssm-parameter
eksdemo get ssm-session
eksdemo get organization
eksdemo get sagemaker domain
eksdemo get sagemaker user-profile
eksdemo get cognito user-pool
eksdemo get cognito app-client
eksdemo get cognito domain
```

**Useful universal flags:** `--dry-run` (preview steps without applying), `-c <cluster>` (target cluster), `-o yaml` / `-o json` (raw output on `get` commands), `-h` (per-command help — the catalog is large enough that `-h` on any subcommand is worth using liberally).

Given your playground's node/instance limits, `install`s that provision node groups (like Karpenter's default NodePool) are the ones to double-check with `--dry-run` first.