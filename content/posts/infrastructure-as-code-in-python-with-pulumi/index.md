---
title: "Infrastructure as Code in Python: Why Engineering Teams Are Adopting Pulumi"
summary: "Moving beyond domain-specific languages: how using Python with Pulumi empowers DevOps teams with real programming loops, type checking, unit testing with pytest, and shared libraries."
categories: ["Python", "DevOps"]
tags: ["pulumi", "python", "iac", "devops", "cloud"]
date: 2026-07-05
draft: false
showTableOfContents: true
---

For years, Infrastructure as Code (IaC) was dominated by domain-specific declarative languages like HashiCorp Configuration Language (HCL) or YAML. While HCL is fantastic for straightforward architectures, complex enterprise requirements—such as dynamic loops over VPC subnets, parsing external JSON configurations, or applying complex encryption logic—often forced engineers into awkward templating workarounds.

**Pulumi** transforms this paradigm by allowing DevOps and software engineers to define cloud infrastructure using **general-purpose programming languages like Python**.

---

## 💡 Why Python for Infrastructure as Code?

Using Python for IaC provides immediate engineering superpowers:

1. **Full IDE Autocomplete & Type Safety:** Instant autocomplete for every cloud resource and property via Python type annotations.
2. **Real Programming Constructs:** Use standard `for` loops, list comprehensions, `if/else` conditionals, and classes without awkward template hacks.
3. **True Unit Testing:** Test your infrastructure logic with `pytest` without provisioning real cloud resources.
4. **Standard Package Ecosystem:** Distribute internal company infrastructure modules via standard Python packages on PyPI or internal Artifactory.

---

## 🏗️ Building an AWS Infrastructure Stack in Python

Here is a real-world Pulumi program written in Python that provisions a secure S3 bucket with lifecycle rules, server-side encryption, and public access blocks:

```python
import pulumi
import pulumi_aws as aws

# Retrieve current Pulumi stack configuration
config = pulumi.Config()
environment = pulumi.get_stack()
bucket_prefix = config.get("bucket_prefix") or f"fhmio-data-{environment}"

# 1. Provision KMS Key for custom encryption
kms_key = aws.kms.Key(
    f"{environment}-s3-kms-key",
    description=f"KMS key for {environment} S3 storage encryption",
    deletion_window_in_days=10,
    enable_key_rotation=True,
    tags={"Environment": environment, "ManagedBy": "Pulumi"},
)

# 2. Provision S3 Bucket with strict encryption
secure_bucket = aws.s3.Bucket(
    f"{environment}-secure-bucket",
    bucket_prefix=bucket_prefix,
    server_side_encryption_configuration=aws.s3.BucketServerSideEncryptionConfigurationArgs(
        rule=aws.s3.BucketServerSideEncryptionConfigurationRuleArgs(
            apply_server_side_encryption_by_default=aws.s3.BucketServerSideEncryptionConfigurationRuleApplyServerSideEncryptionByDefaultArgs(
                kms_master_key_id=kms_key.arn,
                sse_algorithm="aws:kms",
            ),
        ),
    ),
    tags={"Environment": environment, "Owner": "DevOps"},
)

# 3. Block all public access by default
public_access_block = aws.s3.BucketPublicAccessBlock(
    f"{environment}-public-block",
    bucket=secure_bucket.id,
    block_public_acls=True,
    block_public_policy=True,
    ignore_public_acls=True,
    restrict_public_buckets=True,
)

# 4. Export outputs for downstream CI/CD stacks
pulumi.export("bucket_name", secure_bucket.id)
pulumi.export("bucket_arn", secure_bucket.arn)
pulumi.export("kms_key_arn", kms_key.arn)
```

---

## 🧪 Unit Testing Infrastructure with `pytest`

One of Pulumi's most transformative advantages is the ability to write rapid, offline unit tests using `pytest` and mocks:

```python
import pytest
import pulumi

class MyMocks(pulumi.runtime.Mocks):
    def new_resource(self, args: pulumi.runtime.MockResourceArgs):
        return [args.name + "_id", args.inputs]

    def call(self, args: pulumi.runtime.MockCallArgs):
        return {}

pulumi.runtime.set_mocks(MyMocks())

# Import infrastructure modules after initializing mocks
import infrastructure

@pulumi.runtime.test
def test_s3_public_access_is_blocked():
    def check_public_block(args):
        block_public_acls, block_public_policy = args
        assert block_public_acls is True, "Public ACLs must be blocked!"
        assert block_public_policy is True, "Public policy must be blocked!"

    return pulumi.Output.all(
        infrastructure.public_access_block.block_public_acls,
        infrastructure.public_access_block.block_public_policy
    ).apply(check_public_block)
```

This test runs in milliseconds in your local terminal or GitHub Actions pipeline without paying a single cent in AWS API calls.

---

## 🚀 Conclusion

By bringing software engineering rigor—unit tests, linters, packaging, and object-oriented design—into infrastructure provisioning, Python with Pulumi empowers platform engineers to build truly scalable cloud foundations.
