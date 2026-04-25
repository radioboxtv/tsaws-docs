# AWS IAM Permissions

Minimum-privilege IAM policy for the tsaws Connector task role. All actions are read-only. The Connector never mutates customer AWS infrastructure.

Attach this policy to the ECS task role created by the Terraform module or CloudFormation template.

## Policy JSON

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Route53Read",
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets",
        "route53:GetHostedZone"
      ],
      "Resource": "*",
      "Comment": "Enumerate allowlisted private hosted zones and their records. ListHostedZones is required to validate zone IDs at startup. GetHostedZone is required by the bootstrap validator to confirm each zone ID resolves. ListResourceRecordSets pages through A, AAAA, CNAME, and SRV records for eligible service discovery."
    },
    {
      "Sid": "EC2Describe",
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*"
      ],
      "Resource": "*",
      "Comment": "Resolve VPC and subnet Name tags for Connector node tags (R40). Used to populate tag:aws-vpc-* and tag:aws-subnet-* on the Connector's tsnet node. All Describe* actions are read-only."
    },
    {
      "Sid": "ELBDescribe",
      "Effect": "Allow",
      "Action": [
        "elasticloadbalancing:Describe*"
      ],
      "Resource": "*",
      "Comment": "Resolve ALB and NLB target group listener ports when a Route 53 alias record points to a load balancer. Required for port derivation step 4 (spec §10.2)."
    },
    {
      "Sid": "RDSDescribe",
      "Effect": "Allow",
      "Action": [
        "rds:DescribeDBInstances",
        "rds:DescribeDBClusters"
      ],
      "Resource": "*",
      "Comment": "Derive engine type and port for RDS instances and Aurora clusters. DescribeDBInstances covers single-instance RDS. DescribeDBClusters covers Aurora writer, reader, and custom endpoints. Neptune clusters are also described via these RDS APIs."
    },
    {
      "Sid": "ElastiCacheDescribe",
      "Effect": "Allow",
      "Action": [
        "elasticache:Describe*"
      ],
      "Resource": "*",
      "Comment": "Derive cache engine (Redis vs Memcached) and port for ElastiCache clusters and replication groups."
    },
    {
      "Sid": "MemoryDBDescribe",
      "Effect": "Allow",
      "Action": [
        "memorydb:Describe*"
      ],
      "Resource": "*",
      "Comment": "Describe MemoryDB clusters to confirm engine and port (always 6379 for Redis-compatible MemoryDB)."
    },
    {
      "Sid": "MSKDescribe",
      "Effect": "Allow",
      "Action": [
        "kafka:Describe*",
        "kafka:List*"
      ],
      "Resource": "*",
      "Comment": "List and describe MSK clusters to determine broker endpoints and the active security protocol (plaintext 9092, TLS 9094, SASL/SCRAM 9096). List* is required because MSK describe APIs require a cluster ARN obtained from listing."
    },
    {
      "Sid": "DocumentDBDescribe",
      "Effect": "Allow",
      "Action": [
        "docdb:Describe*"
      ],
      "Resource": "*",
      "Comment": "Describe DocumentDB clusters and instances to confirm endpoint and port (always 27017)."
    },
    {
      "Sid": "OpenSearchDescribe",
      "Effect": "Allow",
      "Action": [
        "es:Describe*",
        "es:ListDomainNames"
      ],
      "Resource": "*",
      "Comment": "List and describe OpenSearch Service (formerly Elasticsearch Service) domains. ListDomainNames is required to enumerate domains before describing them. Port is 443 or 9200 depending on cluster configuration."
    },
    {
      "Sid": "NeptuneDescribe",
      "Effect": "Allow",
      "Action": [
        "neptune-db:GetEngineStatus"
      ],
      "Resource": "*",
      "Comment": "Check Neptune engine status. Neptune cluster and instance metadata is obtained via rds:DescribeDBClusters and rds:DescribeDBInstances (Neptune is built on Aurora). This action covers Neptune data-plane engine status checks."
    },
    {
      "Sid": "TagsRead",
      "Effect": "Allow",
      "Action": [
        "tag:GetResources"
      ],
      "Resource": "*",
      "Comment": "Fetch tailscale:* tags on underlying AWS resources across all service types in a single API call. Used to evaluate tailscale:register=true opt-in signal and read override tags (tailscale:ports, tailscale:service-name, etc.)."
    }
  ]
}
```

## Explicitly prohibited

The following action patterns are never granted, even for convenience:

- `route53:Change*` — no DNS mutations
- `ec2:Create*`, `ec2:Modify*`, `ec2:Delete*` — no VPC mutations
- `rds:Create*`, `rds:Modify*`, `rds:Delete*` — no RDS mutations
- `elasticache:Create*`, `elasticache:Modify*`, `elasticache:Delete*`
- `iam:*` — no IAM mutations of any kind

The execution role (separate from the task role) additionally requires:

- `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents` — CloudWatch Logs
- `secretsmanager:GetSecretValue` — scoped to the OAuth secret ARN only

## Notes

1. `ec2:Describe*` is broad but every action in the family is read-only. Scoping to individual Describe actions would require enumeration of ~80 actions and would break as AWS adds new ones.
2. `kafka:List*` is required because MSK's describe APIs take a cluster ARN as input, and the ARN is only obtainable by listing clusters first.
3. `es:ListDomainNames` is a list action required before any `es:Describe*` call can be made. It is added alongside the spec's `es:Describe*` for this reason.
4. `docdb:Describe*` and `neptune-db:GetEngineStatus` actions apply to both standalone clusters and those in global clusters.
