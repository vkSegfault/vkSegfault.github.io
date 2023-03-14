---
title: Terraform remote backend
categories: [cloud]
tags: [terraform, aws]     # TAG names should always be lowercase
---

It is usually a case that for very small (and sometimes even for medium-sized projects) we store our Terraform state file(s) as a part of our repo - in other words we use **local state**. While this might work quite well if infrastructure is not big enough and not many people will try to change it, there is still possibility to introduce problems that may negatively impact our infrastructure.

> But bigger the state file gets, longer it is updated (and imagine that after long wait at the end connection fails and we need to apply this once more).

## The Problem

Why do we even need state, can't we just use what we got in `.tf` files? Well, theretically this would work. But how do we know what has been already provisioned, `.tf` files only tells what we want, not what is already running. Knowing what is already provisioned means we can simply destroy - in reverse order - all resources previously created. Having state also means we can cache things up. This will result faster execution times.

> Terraform must have state to operate correctly. It uses state to:
> 1. map 1 cloud resource to 1 local counterpart
> 2. to manage dependencies and thus creation/deletion order
> 3. to cache those and speed things up

So we *need* a state. What about using it locally and just check-in into git with all other code that our programmers write? Once more, not a great idea. Terraform state is just a file, a big json file. Imagine 2 or more peoply trying to check out code from repository, applying different changes and checking it back to repo. Bad things may happen just when both poeple apply different changes to same infrastructure and then when both peaople will push changed Terraform state into repositotory. This file may easily get broken.

> Although you should definitely store your Terraform code in version control, storing Terraform state in version control is a _bad idea_ because we can (and will do sooner or later) forget to pull latest changes of state file from github and apply conflicting one (basically **race condition**), we also don't have locking, and we can't use secretes because repos are not encrypted (usually .tf files don't have secrets but when we use them as input to tf commands then all those secrets are inserted into tf state).

To protect againts mentioned problems we need 3 things. First is **versioning**. We actually get this one for free when using same repository for IaC files and our app files but it's not particulary good idea. Another is machinism called **locking**. If you are familiar with multithreaded code then idea is basically the same: when 1 user checkouts code that containts state we lock it so no other users can use it at the same time. Once all operations are applied by first user, updated state is merged in and lock is removed. Now others can make theirs changes. This is simple yet crucial to have state in good condition all the time. The last one is **encryption** of state, especially when it includes sensitive data.

> Terraform state is fragile.

## Local vs Remote

By default when we create new state it's called local. Meaning it's just file that. But there is also notion of remotely stored state - stored somewhere outside, with possibility to automate versionig, locking and ecryption. The best part is Terraform has implemented easy way to create it so get to work.

## Remote backend

> **Remote backend is just some cloud storage on e.g.: AWS S3, Azure Storage or Google Cloud Storage with few additional features enabled.**

To implement all above protective machanisms we first need to create another repository for separate infrastructure.

We will use AWS for no other reason than it's most populat cloud provider. Region is up to you:
```json
provider "aws" {
	region = "us-east-1"
}
```

Now we need place to store actual state file somewhere, obvious choice is S3 so let's use it:
```json
resource "aws_s3_bucket" "tf_state" {
	bucket = "tf-state-bucket-totally-unique-name"
	
	lifecycle {
		prevent_destroy = true   # blocks `terraform destroy`, will result in error
	}
}

# enable versioning
resource "aws_s3_bucket_versioning" "tf_state_versioning" {
	bucket = aws_s3.bucket.tf_state.id
	versioning_configuration {
		versionig = "Enabled"
	}
}

# enable server side encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "tf_state_encryption" {
  bucket = aws_s3_bucket.tf_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# block accidental enable to public
resource "aws_s3_bucket_public_access_block" "tf_state_public_access" {
  bucket                  = aws_s3_bucket.tf_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# create DynamoDB table for locking
resource "aws_dynamodb_table" "tf_state_locks" {
  name         = "tf_state_locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

now just run `terraform init` and `terraform apply` in this directory. Once S3 bucket and DynamoDB table have been created we need to go back to our original project (the one that will use above remote backend to store it's own state) and tell Terraform to use them as *remote backend*:

```json
# this is within original tf project files, not one that created remote backend
terraform {
  backend "s3" {
    bucket         = "tf_state"   # our bucket name
    key            = "global/s3/terraform.tfstate"   # place within bucket where it should be stored, this must be unique for every module (project) we want to store state
    region         = "us-east-1"

    dynamodb_table = "tf_state_locks"   # our DynamoDB table name
    encrypt        = true   # 2nd layer of encryption
  }
}
```
and now we just call `terraform init` to upload local backend to remote backend. From now on every change to tf state uses locking, is encrypted and is versioned. 

> **We can use this particular remote backend setup (S3 + DynamonDB) as remote repo for all our projects. At least those within one account.**

There is small caveat though: backend block in Terraform can't use variables so any sensitive value must be hardcoded or... we can kinda sort it out with using separate file. We paste there all needed values and feed then to Terraform with `terraform init -backend-config=values.txt`.

One last thing: if someone else is already running `apply`,  you will have to wait. You can run `apply` with the `-lock-timeout=<TIME>` parameter to instruct Terraform to wait up to TIME for a lock to be released (e.g., `-lock-timeout=5m` will wait for 5 minutes).
