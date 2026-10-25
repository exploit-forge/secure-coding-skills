# Terraform

These are misconfiguration and secret-handling rules (A02, A03, A04). A public website bucket that is supposed to serve static files can stay public; say that in the review and do not file it. A bucket of customer data, an admin port, or an app role cannot.

`sensitive = true` only redacts plan output. The value is still in state. Remote state, encryption at rest, and access control are the actual control. Do not commit `terraform.tfstate` or a `terraform.tfvars` that holds a password.

## Storage is private (A02)

Turn on the account-level public access block. Do not set an ACL of `public-read` on a data bucket.

```hcl
resource "aws_s3_bucket_public_access_block" "data" {
  bucket                  = aws_s3_bucket.data.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

```hcl
resource "aws_s3_bucket_acl" "data" {
  bucket = aws_s3_bucket.data.id
  acl    = "public-read"
}
```

What goes wrong: the ACL or a bucket policy with principal `*` makes every object world-readable. The block stops that even if a later policy tries to open the bucket.

## Security groups name a source (A02)

Allow application ports from the load balancer or the VPC CIDR you actually use. Do not open SSH, RDP, or database ports to `0.0.0.0/0` or `::/0`.

```hcl
ingress {
  description = "app from vpc"
  from_port   = 443
  to_port     = 443
  protocol    = "tcp"
  cidr_blocks = [var.vpc_cidr]
}
```

```hcl
ingress {
  from_port   = 22
  to_port     = 22
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}
```

What goes wrong: the instance accepts administration traffic from the whole internet. A bastion or SSM path is the supported way in; the security group does not need a world route.

## IAM is a specific action and resource (A01, A02)

Name the actions and the resource ARNs. `Action = "*"` or `Resource = "*"` on an application role is not least privilege.

```hcl
statement {
  effect    = "Allow"
  actions   = ["s3:GetObject"]
  resources = ["${aws_s3_bucket.data.arn}/uploads/*"]
}
```

```hcl
statement {
  effect    = "Allow"
  actions   = ["*"]
  resources = ["*"]
}
```

What goes wrong: the role can call every API in the account. A task role should match the calls the service makes.

## Secrets stay out of the repo and out of casual state (A03, A04)

Read a database password from a secret manager at apply time, and prefer a write-only argument when the resource has one, so the password is not stored in state. Mark variables `sensitive` so plans do not print them. Do not put the value in `terraform.tfvars` that is committed, in a `default`, or in an output that is not `sensitive`.

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}
```

```hcl
variable "db_password" {
  type    = string
  default = "change-me"
}
```

What goes wrong: `default` and a committed tfvars file put the secret in git history. State still holds ordinary sensitive attributes, so the backend must be private and encrypted. A local `terraform.tfstate` on a laptop is a copy of those values.
