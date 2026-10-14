# aws-vpc-network-foundation

A VPC in `ap-south-1` with public, private, and isolated subnets in 2 Availability Zones. Terraform keeps its state in S3 with a lock. GitHub Actions deploys every change to `infra/`.

## Problem

A new AWS account has no network for services. A Go service, a load balancer, or a database has no place to run until a VPC exists. Terraform state on a laptop is lost with the laptop, and two runs at the same time can corrupt it.

Constraints:

- Users: one developer and one CI pipeline.
- Traffic: none yet. Later projects put their services in this network.
- Budget: less than $50 per month while idle.
- Failures to prevent: a database that the internet can reach, a state file that two runs corrupt, and AWS keys stored in GitHub.

## Design

```
                       Internet
                          |
                   Internet gateway
                          |
+------------------ VPC 10.42.0.0/16 ------------------+
|        AZ ap-south-1a          AZ ap-south-1b        |
|   +------------------+    +------------------+       |
|   | public           |    | public           |       |
|   | 10.42.0.0/24     |    | 10.42.1.0/24     |       |
|   | NAT gateway      |    |                  |       |
|   +------------------+    +------------------+       |
|   | private          |    | private          |       |
|   | 10.42.16.0/20    |    | 10.42.32.0/20    |       |
|   +------------------+    +------------------+       |
|   | isolated         |    | isolated         |       |
|   | 10.42.64.0/24    |    | 10.42.65.0/24    |       |
|   +------------------+    +------------------+       |
|                                                      |
|   S3 gateway endpoint on all 3 route tables          |
+------------------------------------------------------+
```

The route table of a subnet sets its tier. AWS has only one kind of subnet.

| Tier | Default route | Holds |
|---|---|---|
| Public | `0.0.0.0/0` to the internet gateway | Load balancers, the NAT gateway |
| Private | `0.0.0.0/0` to the NAT gateway | Go services, workers |
| Isolated | No default route | Databases, caches |

The private subnets are `/20` (4,096 addresses) because containers and Lambda functions each use one IP address. The public and isolated subnets are `/24` (256 addresses). The gaps between the blocks leave space for new subnets. The VPC uses `10.42.0.0/16` and not `10.0.0.0/16`, because the common range is the one most likely to overlap a future peered network or VPN.

The repository has two Terraform folders:

- `bootstrap/` creates the S3 state bucket, the GitHub OIDC provider, and the IAM role for CI. It uses local state, because the bucket does not exist before it runs.
- `infra/` creates the VPC. It stores state in S3 at `network-foundation/terraform.tfstate` with `use_lockfile = true`.
- `.github/workflows/deploy.yml` runs `terraform plan` and `terraform apply` on each push to `main` that changes `infra/`.

CI gets AWS access through OIDC. GitHub signs a token for each run. AWS uses the public key of GitHub to prove the signature. Then AWS compares the `sub` claim with the trust policy of the role and gives credentials that expire after 1 hour.

### Options

Option A is one NAT gateway in `ap-south-1a`. Option B is one NAT gateway in each AZ. I selected Option A. I rejected Option B because it doubles the fixed cost, and no workload runs in the private subnets yet.

For the state lock, I used the S3 lock file and not a DynamoDB table. Terraform 1.10 added `use_lockfile`, and DynamoDB locking is now deprecated. This removes one resource.

## Trade-offs

The main trade-off is cost against availability of outbound traffic. One NAT gateway costs about $44.53 per month while idle. Two cost about $89.06. If `ap-south-1a` fails, both private subnets lose outbound internet access. Inbound traffic and traffic inside the VPC continue to work.

Other trade-offs:

- The isolated subnets cannot reach the internet. RDS and ElastiCache get patches from AWS through a separate path. A database that you run yourself on EC2 cannot get patches there.
- The CI role has `AdministratorAccess`. This is fast to set up, but it is too broad for a shared account.
- The `bootstrap/` state is local and is not in Git. If the laptop is lost, you must import the bucket and the role into a new state.

## Result

- 20 resources: 1 VPC, 1 internet gateway, 6 subnets, 3 route tables, 6 route table associations, 1 Elastic IP, 1 NAT gateway, and 1 S3 gateway endpoint.
- 2 min 50 s for one CI run from push to `Apply complete`. The NAT gateway takes most of the time.
- 0 long-lived AWS keys in GitHub. The only secret is the role ARN.
- $44.53 per month while idle in `ap-south-1`: $40.88 for NAT gateway hours ($0.056 per hour), and $3.65 for the public IPv4 address ($0.005 per hour). Data through the NAT gateway costs $0.056 per GB. The S3 gateway endpoint costs $0.

The first CI run failed with `Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity`. The repository uses the immutable subject format of GitHub. The `sub` claim contains the owner ID and the repository ID: `repo:veerbal1@55359171/aws-vpc-network-foundation@1418744444:ref:refs/heads/main`. The trust policy expected `repo:veerbal1/aws-vpc-network-foundation:*`. After the policy used the ID format, the run passed. The ID format is also safer. A new repository with the same name has a different ID, so it cannot assume the role.

To test the lock, start two `terraform plan` runs in `infra/` at the same time. While the first run holds the lock, S3 contains `network-foundation/terraform.tfstate.tflock`. The second run stops with `Error acquiring the state lock`.

## How to run

You must have an AWS SSO profile named `homestead` and the GitHub CLI `gh`.

1. Run `aws sso login --profile homestead`.
2. In `bootstrap/`, run `AWS_PROFILE=homestead terraform init` and then `AWS_PROFILE=homestead terraform apply`.
3. Copy the `github_actions_role_arn` output.
4. Run `gh secret set AWS_ROLE_ARN --body "<role ARN>"`.
5. Push a change in `infra/` to `main`. GitHub Actions applies it.

Do not run `terraform apply` in `infra/` from a laptop. CI is the only deployer.

## What I would do at 100x scale

- Put the network in its own account in AWS Organizations, and share subnets with workload accounts through AWS RAM.
- Use AWS IPAM to give each VPC a CIDR block that does not overlap with other networks.
- Add one NAT gateway in each AZ, and add interface endpoints for ECR, STS, and CloudWatch Logs to reduce data through the NAT gateway.
- Give the CI role only the permissions that the VPC needs.
- Run `terraform plan` on pull requests, and require an approval in a GitHub environment before `apply`.
- Turn on VPC Flow Logs to see rejected traffic.
