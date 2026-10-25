# CloudFormation

Template misconfiguration and secret handling (A02, A03, A04). A bucket meant to host a public website can be public; a data bucket cannot. `NoEcho` hides a parameter in some console views. It does not keep a `Default` out of the template or out of stack events if you also output it.

## S3 data is private (A02)

Block public ACLs and public policies. Do not set `AccessControl` to `PublicRead` on a data bucket.

```yaml
PublicAccessBlockConfiguration:
  BlockPublicAcls: true
  BlockPublicPolicy: true
  IgnorePublicAcls: true
  RestrictPublicBuckets: true
```

```yaml
AccessControl: PublicRead
```

What goes wrong: objects are readable without authentication. The public access block is the control that stays on when a later policy is edited.

## Security groups name a source (A02)

Open the app port to the VPC CIDR or the load balancer security group. Do not use `0.0.0.0/0` on SSH, RDP, or database ports, and do not allow all ports from the world.

```yaml
SecurityGroupIngress:
  - IpProtocol: tcp
    FromPort: 443
    ToPort: 443
    CidrIp: 10.0.0.0/16
```

```yaml
SecurityGroupIngress:
  - IpProtocol: -1
    CidrIp: 0.0.0.0/0
```

What goes wrong: every address can reach every port on the instance. Use the security group of the caller when you can, so the rule tracks that tier instead of a wide CIDR.

## IAM names the call (A01, A02)

List the actions and the resource ARNs. `Action: '*'` with `Resource: '*'` is not an application role.

```yaml
Policies:
  - PolicyName: read-uploads
    PolicyDocument:
      Version: "2012-10-17"
      Statement:
        - Effect: Allow
          Action:
            - s3:GetObject
          Resource: !Sub "${DataBucket.Arn}/uploads/*"
```

```yaml
Statement:
  - Effect: Allow
    Action: "*"
    Resource: "*"
```

What goes wrong: the instance profile or the Lambda role can do anything the account allows. Match the policy to the API calls in the code.

## Secrets are references, not defaults (A03, A04)

Point `MasterUserPassword` and similar fields at a dynamic reference to Secrets Manager or SSM. Do not put the password in `Parameters` with a `Default`. If you must accept a parameter, set `NoEcho: true` and do not give it a default, and do not `Output` it. Prefer an instance role or a task role over a long-lived key embedded in the template.

```yaml
MasterUserPassword: "{{resolve:secretsmanager:app/db:SecretString:password}}"
```

```yaml
Parameters:
  DbPassword:
    Type: String
    Default: "change-me"
    NoEcho: true
```

What goes wrong: `Default` is in the template in git. `NoEcho` does not remove that default. A dynamic reference keeps the secret in the secret store; the template only names it. Stack outputs and processed templates still need to be treated as sensitive if a secret was passed through.
