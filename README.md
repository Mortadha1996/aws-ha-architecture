# Highly Available Web Architecture on AWS

A multi-AZ web architecture defined entirely in Terraform, built to be destroyed and rebuilt in minutes — and tested by actually breaking it.

The interesting part isn't the architecture diagram. It's what happens when an instance dies at 3am.

---

## What this builds

```
                        Internet
                            │
                            ▼
              ┌──────────────────────────┐
              │  Application Load Balancer│
              │  (public subnets, 2 AZs)  │
              │  health checks every 30s  │
              └────────────┬─────────────┘
                           │
         ┌─────────────────┴─────────────────┐
         ▼                                   ▼
┌──────────────────┐                ┌──────────────────┐
│  us-east-1a      │                │  us-east-1b      │
│  ┌────────────┐  │                │  ┌────────────┐  │
│  │  EC2 web   │  │                │  │  EC2 web   │  │
│  │  t3.micro  │  │                │  │  t3.micro  │  │
│  └────────────┘  │                │  └────────────┘  │
└──────────────────┘                └──────────────────┘
         └─────────────────┬─────────────────┘
                           │
              ┌────────────▼─────────────┐
              │   Auto Scaling Group      │
              │   min 2 · max 4           │
              │   target tracking @ 60%   │
              └───────────────────────────┘
```

**VPC** spanning two Availability Zones, with public subnets for the load-balanced tier and private subnets reserved for the data tier.

**Application Load Balancer** distributing across both AZs, with health checks that pull an unhealthy target out of rotation.

**Auto Scaling Group** maintaining a minimum of two instances, using ELB health checks rather than EC2 status checks, with target tracking on average CPU.

**Security groups chained by reference**, not by CIDR — the web tier accepts traffic from the ALB security group only, so instances are unreachable from the internet even though they sit in public subnets.

---

## Testing it by breaking it

An architecture is only highly available if you've watched it survive something. So I terminated an instance while a client was hitting the load balancer once per second.

```bash
# Terminal 1 — continuous requests
while true; do
  curl -s --max-time 2 http://$ALB_DNS | grep -o "us-east-1[ab]" || echo "FAILED"
  sleep 1
done

# Terminal 2 — kill an instance
aws ec2 terminate-instances --instance-ids i-02debbb9ce5803b50
```

### What happened

```
us-east-1a
us-east-1b
us-east-1a
us-east-1b
us-east-1a
FAILED          ← one request caught mid-shutdown
us-east-1a
us-east-1b
us-east-1a
us-east-1b
...             ← service continues uninterrupted
```

**One failed request out of roughly forty.** The ALB pulled the dying target out of rotation within seconds, and traffic carried on across both AZs.

Meanwhile the Auto Scaling Group did its own work:

```
+----------------------+--------------+------------+--------------+
|  i-02debbb9ce5803b50 |  Terminating |  Unhealthy |  us-east-1a  |
|  i-0bff8715f8e5f17d0 |  InService   |  Healthy   |  us-east-1b  |
|  i-0d5bb3ecdbe0d17e8 |  Pending     |  Healthy   |  us-east-1a  |  ← replacement
+----------------------+--------------+------------+--------------+
```

### Measured

| What | Observed |
|---|---|
| Failed requests during the event | 1 of ~40 |
| ALB removes unhealthy target | seconds |
| ASG launches replacement | ~2 minutes |
| AZs serving traffic throughout | 2 |

---

## The distinction that matters

The ALB and the ASG both do health checking, and they react on very different timescales.

**The ALB** checks every 30 seconds and needs 3 consecutive failures to mark a target unhealthy — but removing it from rotation is immediate once decided. That's why traffic recovered in seconds.

**The ASG** waits out its `health_check_grace_period` (120s here) before acting, because it has to distinguish "this instance is dead" from "this instance is still booting". That's why the replacement took minutes.

Setting `health_check_type = "ELB"` on the ASG is what connects the two: without it, the ASG only watches EC2 status checks, which stay green on an instance whose web server has crashed. The instance would keep receiving no traffic from the ALB while the ASG considered it perfectly healthy.

---

## Repository layout

```
.
├── main.tf              # provider, default tags, AZ data source
├── variables.tf         # region, environment, VPC CIDR
├── vpc.tf               # VPC, subnets, IGW, route tables
├── security-groups.tf   # ALB and web tier, chained by reference
├── compute.tf           # ALB, target group, launch template, ASG, scaling policy
├── outputs.tf           # VPC ID, subnet IDs, ALB DNS name
└── README.md
```

---

## Running it

```bash
git clone https://github.com/Mortadha1996/aws-ha-architecture
cd aws-ha-architecture

aws configure          # credentials for a non-root IAM user
terraform init
terraform plan
terraform apply
```

The ALB DNS name comes out as an output. Hit it a few times and watch the AZ change.

```bash
terraform destroy      # when you're done
```

---

## On cost

This runs on `t3.micro` instances and a single ALB — roughly **$0.04/hour** for the whole stack. Destroy it when you're not using it and the whole exercise costs a couple of dollars.

**There is deliberately no NAT Gateway.** At ~$32/month it's the most expensive thing in a small architecture like this, and it exists to give private subnets outbound internet access. For a demo where instances live in public subnets behind strict security groups, it buys nothing. In production, with the web tier in private subnets, it becomes necessary — but it's worth knowing what you're paying for.

---

## Design notes

**Security groups reference each other, not CIDR blocks.** `aws_security_group.web` allows port 80 from `aws_security_group.alb`, not from `10.0.0.0/16`. If the ALB moves, gets rebuilt, or changes IPs, the rule still holds. It's also the only reason instances in public subnets aren't directly reachable.

**The launch template pulls instance metadata via IMDSv2.** The token-based flow (`PUT` to get a token, then `GET` with the token header) is the current standard — IMDSv1's unauthenticated GET was exploitable through SSRF in application code.

**Target tracking rather than step scaling.** Telling AWS "keep average CPU at 60%" is simpler and less brittle than defining thresholds and cooldowns by hand.

**Private subnets exist but hold nothing yet.** They're where RDS goes next — a database in a public subnet is a mistake worth avoiding from the start rather than fixing later.

---

## Next

- RDS MySQL Multi-AZ in the private subnets, with automatic failover
- S3 and CloudFront for static content
- CloudWatch alarms on target health and ALB 5xx rates
- HTTPS with ACM and a redirect from port 80

---

## Author

**Mortadha Riahi** — Infrastructure & Platform Engineer

AWS Solutions Architect – Associate · RHCE · RHCSA · Red Hat Ansible (×2) · CCNA

[LinkedIn](https://www.linkedin.com/in/mortadha-riahi/) · [AIOps anomaly detection](https://github.com/Mortadha1996/aiops-anomaly-detection) · [LLM inference platform](https://github.com/Mortadha1996/llm-inference-platform)
