# Terraform State 

This document explains how I setup an S3 bucket for State store that can be used by multiple alleaffengaffen project.

## Facts

The bucket was created manually:

name: terraform-state-alleaffengaffen
region: eu-west-1
public access: blocked
versioning: enabled
object lock: disabled

### IAM

Policy named `terraform-state-alleaffengaffen`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::terraform-state-alleaffengaffen"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::terraform-state-alleaffengaffen/*"
    }
  ]
}
```

The policy is attached to an IAM user whoes name is `terraform-state-alleaffengaffen`. A key-pair for the user can be found in akeyless.

## State config

See https://developer.hashicorp.com/terraform/language/settings/backends/s3
