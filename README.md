# ECS on EC2 Cluster into an Existing VPC (Terraform)

Terraform code (originally "ecs-cluster-sin-vpc", i.e. "ECS cluster without VPC") that deploys an EC2-backed ECS cluster with
an ALB, CPU/memory-based Auto Scaling and a sample NGINX service into an **existing** VPC in any region.

> **Status:** legacy reference (2020), written for Terraform 0.11-era syntax (`"${...}"` interpolation, `type = "string"`, `tags {}`/`dimensions {}` blocks), not actively maintained. Review and update providers/syntax before using.

## Architecture

- ECS cluster named `var.ecs_cluster`; `aws_key_pair` `<cluster>-key` from an embedded public key.
- One IAM role trusted by both `ecs.amazonaws.com` and `ec2.amazonaws.com`, with `AmazonEC2ContainerServiceRole`,
  `AmazonEC2ContainerServiceforEC2Role`, `AmazonSSMFullAccess` and `CloudWatchFullAccess`; instance profile on top.
- ALB in `subnet_1..3` with SG allowing 80/443 from `0.0.0.0/0`, HTTP:80 listener and a default target group (health check `/`).
- Cluster SG allowing 22/80/443 from `10.0.0.0/16` (hardcoded CIDR).
- Launch configuration (`ami_id`, `instance_type`, 30 GB gp2) whose user data joins the cluster, installs Docker, the SSM agent
  and the CloudWatch monitoring scripts (memory/disk metrics every minute via cron).
- Auto Scaling group in `subnet_4..6` with min/desired/max from variables.
- Simple scaling policies + alarms: CPU (`AWS/EC2`) and memory (`System/Linux`), scale up at >= 80%, scale down at <= 80%.
- ECS service `nginx` (`nginx:latest`, 1 task, host port 80) behind the ALB, logs to CloudWatch group `<cluster>/ecs/nginx`.

Output: `alb_hostname` (see the bug note below).

## Inputs

| Name | Description | Default |
|------|-------------|---------|
| `access_key` / `secret_key` | AWS credentials for the provider | none (required) |
| `region` | AWS region | none (required) |
| `ami_id` | ECS-optimized Amazon Linux 2 AMI ID for the region | none (required) |
| `ecs_cluster` | Cluster name (also used as prefix for most resources) | none (required) |
| `vpc_id` | Existing VPC ID | none (required) |
| `subnet_1`, `subnet_2`, `subnet_3` | Subnets for the ALB (public) | none (required) |
| `subnet_4`, `subnet_5`, `subnet_6` | Subnets for the ECS instances (private) | none (required) |
| `min_instance_size` / `desired_capacity` / `max_instance_size` | ASG sizing | none (required) |
| `instance_type` | EC2 instance type for the nodes | none (required) |

## Usage

```bash
terraform init
terraform plan  -var-file=my.tfvars     # region, ami_id, vpc_id, subnet_1..6, sizes, credentials
terraform apply -var-file=my.tfvars
```

Keep `my.tfvars` out of git (it would contain `<ACCESS_KEY_ID>` / `<SECRET_ACCESS_KEY>`); better, drop the key variables and use an
AWS profile. No backend is configured; state is local.

## Security notes

- `outputs.tf` references `aws_alb.jalcalaroot`, which does not exist (the ALB is `aws_alb.ecs-load-balancer`): plan fails until fixed.
- Instance role has `AmazonSSMFullAccess` and `CloudWatchFullAccess` — far broader than needed (use `AmazonSSMManagedInstanceCore` and scoped CloudWatch permissions).
- The ALB SG (80/443 from `0.0.0.0/0`) is also attached to the instances; ALB is HTTP only.
- CloudWatch monitoring scripts downloaded over plain HTTP; the cron job omits `--auto-scaling`, so the ASG-dimension memory alarms may never receive data.
- Scale-up and scale-down thresholds are both 80%, so policies can flap.
- AWS credentials as plain variables, local state, root volumes not encrypted, `nginx:latest` mutable tag, deprecated `aws_launch_configuration` and legacy ECS service role.

## Cleanup

```bash
terraform destroy -var-file=my.tfvars
```

## License

MIT — see [LICENSE](LICENSE).
