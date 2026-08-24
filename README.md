# Auto Scaling Lab

Highly available, auto-scaling web tier on AWS: an internet-facing Application
Load Balancer fronting an Auto Scaling Group of EC2 instances in private
subnets, deployed entirely from a single CloudFormation template.

## What is included

- `template.yml` — the full CloudFormation stack:
  - VPC with 2 public subnets (ALB) and 2 private subnets (EC2), across two AZs
  - Internet Gateway + public route table for the public subnets
  - Single regional NAT Gateway + private route table so private instances can
    reach the internet for package installs/updates
  - Security groups: ALB SG allows inbound `80` from `0.0.0.0/0`; instance SG
    allows inbound `80` **only** from the ALB SG — no SSH ingress anywhere
  - Application Load Balancer, target group (health check on `/`), and listener
  - IAM role/instance profile with `AmazonSSMManagedInstanceCore` so instances
    are reachable via **SSM Session Manager** instead of SSH
  - Launch Template: latest Amazon Linux 2 AMI (resolved via SSM public
    parameter, works in any region), detailed (1-minute) monitoring enabled,
    User Data installs Apache and a demo page showing the instance's ID, AZ,
    and private IP
  - Auto Scaling Group: min 1 / desired 1 / max 4, `ELB` health checks
  - Target-tracking scaling policy on `ASGAverageCPUUtilization` at 30%

No separate `user-data.sh` file is needed — the User Data script is embedded
directly in the Launch Template resource in `template.yml`.

## Deploy

```bash
aws cloudformation deploy \
  --template-file template.yml \
  --stack-name auto-scaling-lab \
  --capabilities CAPABILITY_NAMED_IAM
```

Grab the ALB DNS name once the stack is `CREATE_COMPLETE`:

```bash
aws cloudformation describe-stacks \
  --stack-name auto-scaling-lab \
  --query "Stacks[0].Outputs"
```

## Deploying via CloudFormation GitSync (required deployment method)

1. Push this folder to a GitHub repository (see below).
2. In the CloudFormation console, choose **Stacks → Create stack → With new
   resources**, then pick **Sync from Git**.
3. Connect your GitHub account/repo (creates a CodeConnections connection the
   first time), select the branch, and set the template path to
   `template.yml`.
4. CloudFormation creates a stack and a deployment file (`*.stack.json`) in
   the repo; every subsequent push to the tracked branch automatically
   updates the stack — this satisfies the "repeatable, auditable IaC"
   requirement.

## Live demonstration script

**1. Single public endpoint, round-robin load balancing**

```bash
for i in $(seq 1 10); do curl -s http://<ALB_DNS_NAME>/ | grep "Instance ID"; done
```

With `DesiredCapacity` raised above 1 (or after a scale-out), you'll see
different instance IDs answering — proof traffic is spread across servers
without the client knowing how many exist.

**2. Prove there's no direct inbound access to instances**

Instances have no public IP and no SSH ingress rule. Connect instead via SSM
Session Manager (uses the instance's `AmazonSSMManagedInstanceCore` role):

```bash
aws ssm start-session --target <instance-id>
```

**3. Trigger a scale-out event with a CPU stress test**

From inside the session:

```bash
sudo stress --cpu $(nproc) --timeout 300s          # if epel/stress installed
# or, dependency-free fallback baked into every instance:
sudo /home/ec2-user/cpu-stress-fallback.sh 300
```

Or trigger it remotely on all instances without opening a session, via SSM
Run Command:

```bash
aws ssm send-command \
  --targets "Key=tag:Name,Values=AutoScalingLabInstance" \
  --document-name "AWS-RunShellScript" \
  --parameters commands="sudo bash /home/ec2-user/cpu-stress-fallback.sh 300"
```

**4. Watch the scale-out happen**

- CloudWatch → Alarms: the target-tracking alarm (`TargetTracking-...-AlarmHigh...`)
  goes into `ALARM` once average CPU exceeds 30% for the evaluation window.
- EC2 → Auto Scaling Groups → **Activity** tab: shows the new instance launch.
- EC2 Target Groups → **Targets** tab: the new instance registers and turns
  `healthy` automatically — no manual step required.
- Re-run the `curl` loop from step 1 to show the new instance's ID appearing
  in the responses.

**5. Scale-in (extra credit, automatic)**

Target-tracking scaling manages both directions: once CPU drops back below
30% (stop the stress script, or let the `timeout` expire), the same policy
scales the ASG back down toward the target after its cooldown — no separate
scale-in policy is needed, and this can be shown live by just waiting it out
after step 4.

## Scaling policy and thresholds explained

- **Policy type:** Target Tracking Scaling on the predefined metric
  `ASGAverageCPUUtilization`, target value `30.0`.
- **Why target tracking over simple/step scaling:** a single declarative
  target lets AWS manage both scale-out *and* scale-in with its own
  CloudWatch alarms, instead of hand-tuning separate high/low-threshold step
  policies — fewer moving parts, less risk of oscillation ("flapping").
- **`EstimatedInstanceWarmup: 120`**: tells the policy to ignore a new
  instance's CPU metrics for its first 2 minutes (Apache/OS startup), so a
  freshly launched instance's initial idle-then-spiking CPU doesn't skew the
  average or trigger a premature second scale-out.
- **1-minute detailed monitoring** (`Monitoring: Enabled: true` on the launch
  template) shortens the CloudWatch metric period from the default 5 minutes
  to 1 minute, so the demo's scale-out reaction is visible in a few minutes
  instead of up to ~15.
- **Min/Desired/Max = 1/1/4**: keeps steady-state cost minimal (a single
  instance) while allowing up to 4x capacity under load.

## Security and cost-optimization notes

- Instances sit in private subnets with no public IP and no SSH ingress;
  all administrative access is via SSM Session Manager (IAM-audited, no
  key pairs to manage or leak).
- A single NAT Gateway (not one per AZ) is used to keep cost down, since this
  is a lab/demo workload rather than a production multi-AZ-NAT design.
- The instance security group only accepts port 80 from the ALB security
  group (not from `0.0.0.0/0`), so instances are unreachable except through
  the load balancer.
- The AMI is resolved at deploy time from the public SSM parameter
  `/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2`, so the
  template is region-portable and always picks up the latest patched AMI.
