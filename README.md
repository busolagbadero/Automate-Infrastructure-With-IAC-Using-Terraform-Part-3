# Automate-Infrastructure-With-IAC-Using-Terraform-Part-3


In this project, we will make some changes to our existing project. Firstly, we will switch from using local storage to using S3 as our backend. This change is necessary to ensure that all engineers working on the project have access to the same terraform.tfstate file, regardless of their location.

We will also refactor our code to use modules, which will improve the reusability of our code. To do this, we will create a directory called "modules", and inside it, we will create subdirectories for different components such as VPC, Security, EFS, RDS, ALB, Autoscaling, and Compute. Each of these subdirectories will contain the necessary files for setting up that component.

We will also create a variables.tf file in each of the subdirectories to ensure that all arguments in the module are declared as variables, instead of being hardcoded. This will allow engineers to easily customize the module by tweaking the input parameters to suit their specific needs.

Overall, the directory structure of our modules directory will be organized and easy to navigate, making it simpler for engineers to use and reuse our code.

![56](https://user-images.githubusercontent.com/94229949/236188166-5e0e380c-8811-41dc-a7d3-a44fe8a6525d.png)


After creating new modules in our project, it is important to run "terraform init" to initialize and download the necessary modules and plugins. This ensures that our project has access to all the required resources and can successfully run the modules without any errors.


![bradford-3](https://user-images.githubusercontent.com/94229949/236188667-9b13a365-7089-4e07-8ce7-9a8a88f00edf.png)


![bradford-4](https://user-images.githubusercontent.com/94229949/236188702-9e4b2691-8e1e-4dbf-b441-56ee7d108307.png)

Once we launch "terraform apply" after making the necessary tweaks to our resources and modules, all the resources should work fine. This is because we have carefully designed and configured our modules to ensure that they are reusable and adaptable to different scenarios. By using variables and modules, we have created a flexible infrastructure that can be easily customized and scaled up or down as needed.


![bradford-5](https://user-images.githubusercontent.com/94229949/236189218-2a780c76-9d0e-44cc-a323-3a0bb5fa52f8.png)


![bradford-6](https://user-images.githubusercontent.com/94229949/236189239-70d7c4bc-31bd-459a-989d-a3dcb915be1b.png)


![bradford-7](https://user-images.githubusercontent.com/94229949/236189252-ea1bee80-772e-4bb6-9ac5-fed88ec5e8c2.png)


![bradford-8](https://user-images.githubusercontent.com/94229949/236189279-f9abdab0-21e8-4e86-baca-967f6985a80b.png)


![bradford-9](https://user-images.githubusercontent.com/94229949/236189293-1aca8bdb-f943-4bf6-85b2-e3738685fb45.png)


![bradford-10](https://user-images.githubusercontent.com/94229949/236189309-6aa06a7f-2f97-414a-bd9a-afe76040577a.png)


![bradford-11](https://user-images.githubusercontent.com/94229949/236189326-eb62aafa-e92b-4067-bb86-7d7d6698e6a6.png)


We will introduce a new backend on S3. Previously, we were using the local backend which stores the state file on the local machine. However, this approach is not reliable, especially when working with a team of DevOps engineers who need to access the state file remotely.

To solve this problem, we will configure an S3 bucket as our backend, which will allow us to store the state file remotely and make it accessible to other team members. The S3 backend is a more robust solution than the local backend and provides better reliability.

In addition to the S3 backend, we can also use the State Locking feature which is supported by S3 backend. This feature is used to lock the state file during write operations to prevent others from corrupting the file. However, to use this feature, we need to set up another AWS service - DynamoDB. State locking is an optional feature but provides an additional layer of security and reliability to our project.

To store our Terraform state file in an S3 bucket, we need to create a new bucket with a unique name and it's important to note that in order to use S3 as a backend for Terraform, versioning must be enabled on the S3 bucket. This ensures that previous versions of the state file can be retrieved in case of accidental deletion or corruption. 

```
resource "aws_s3_bucket" "terraform-state" {
  bucket = "pbl18abc"

  # force_destroy = true

}

resource "aws_s3_bucket_versioning" "version" {
  bucket = aws_s3_bucket.terraform-state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "first" {
  bucket = aws_s3_bucket.terraform-state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

```

To handle locks and ensure consistency when using S3 as our Terraform backend, we need to create a DynamoDB table. In the past, we used a local file to handle locks as shown in terraform.tfstate.lock.info. However, since we are now working with a team, it's necessary to use a cloud storage database like DynamoDB to manage locks centrally.

By using DynamoDB, anyone running Terraform against the same infrastructure can use a central location to control a situation where Terraform is running at the same time from multiple different people. This helps to avoid conflicts and ensure that only one person at a time is making changes to the infrastructure.

```
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
```


To configure our backend to use S3 and DynamoDB, we need to create a 'backend.tf' file and reference the S3 bucket name. We must ensure that both the S3 bucket and DynamoDB table resources have already been created before we configure the backend.

```
terraform {
  backend "s3" {
    bucket         = "pbl18abc"
    key            = "global/s3/terraform.tfstate"
    region         = "us-east-2"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

![bradford-12](https://user-images.githubusercontent.com/94229949/236192768-fccd6cf1-6f96-47d3-b81d-8767c21662cc.png)

![bradford-13](https://user-images.githubusercontent.com/94229949/236192596-652068e6-d7dd-48fb-9747-f64ad2e95cdc.png)



Before running 'terraform destroy', we should be aware that Terraform will destroy the S3 bucket if it is configured as the backend. If the backend is destroyed, Terraform will go into an undefined state and delete the files stored in the S3 bucket. To avoid this, we need to comment out the backend configuration in the 'backend.tf' file and run terraform init -migrate-state command.

This command will copy the state file back from S3 to our local storage, so that we can safely run terraform destroy without losing our state file. Once terraform destroy is complete, we can remove the S3 bucket and DynamoDB table resources.
