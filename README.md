# Practical Introduction to Terraform

This is a short, practical session designed to introduce key concepts of Terraform.

## Install Terraform

Terraform comes as a Go binary. See the documentation for downloading the right
binary for your architecture: https://www.terraform.io/downloads.html

I recommend using `tfenv`, which is a Terraform version manager.

On MacOS, install using Homebrew: `brew install tfenv`

To install the current version, run: `tfenv install latest`

## Set up credentials

It is easiest to use admin credentials to create resources. Ensure you have
access to admin credentials.

## Let's write some code!

Let's create an EC2 instance in eu-central-1 (Frankfurt).

The documentation gives us a good example of how to create a single instance in
AWS: https://www.terraform.io/docs/providers/aws/r/instance.html

### Provider block

We need to use the `aws` provider to give us access to APIs. Terraform
automatically includes access to many third-party providers, and fetches them
as required:

```
provider "aws" {
  region = "eu-central-1"
}
```

The provider block tells Terraform to fetch that provider so we can use their
resources.

> Note: HCL is like JSON, so quotes should always be double-quotes!

### Data block

We should find a recent AMI (Amazon Machine Image) to create our server with.
Let's used the very latest Ubuntu distribution (Disco Dingo):

```
data "aws_ami" "ubuntu" {
  most_recent = true

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-trusty-19.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["099720109477"] # Canonical
}
```

Data blocks allow you to search or produce data. This is extremely useful when
you need to programatically configure something based on variables, rather
than hardcoding things.

See the documentation for `aws_ami`: https://www.terraform.io/docs/providers/aws/d/ami.html

Check the bottom of the documentation for "Attributes Reference". These are
objects produced by the data source that we can reference in our code.

### Resource block

Now let's add the block to actually create our resource. This is completed
using the `resource` block. A `resource` is anything that executes any CRUD
function.

```
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"

  tags = {
    Name = "HelloWorld"
  }
}
```

This will create a server, but we won't be able to SSH into it since it will
not have our SSH key installed.

Let's create another resource block to create a key pair in AWS:

```
resource "aws_key_pair" "my-ssh-key" {
  key_name   = "lauras-ssh-key"
  public_key = "<some key data>"
}
```

We can either pass in a string for our public key, or we may be able to use a
built-in function to output the key. Terraform has lots of built-in functions
to enable us to write better code: https://www.terraform.io/docs/configuration/functions.html

We could use the `file` function here, along with the `pathexpand` function to
to ensure our `~` is respected:

```
resource "aws_key_pair" "my-ssh-key" {
  key_name   = "lauras-ssh-key"
  public_key = file(pathexpand("~/.ssh/id_rsa.pub"))
}
```

We can now reference this key pair to attach to our EC2 instance:

```
resource "aws_instance" "web" {
  ami           = "${data.aws_ami.ubuntu.id}"
  instance_type = "t2.micro"
  key_name      = aws_key_pair.my-ssh-key.key_name

  tags = {
    Name = "HelloWorld"
  }
}
```

When referencing resources, it is assumed you want a `resource`, so you reference
them simply by:

<resource_type>.<resource name>.<object>

For example, to reference the `instance_type` from the resource above, we can
use:


```
aws_instance.web.instance_type
```

Other block types, such as `data` or `local`, must be specified as those types:

`data.aws_ami.ubuntu.id`

### Outputs

Ideally we need to know the public IP of the server we create to be able to SSH
to it.

Let's create an output block:

```
output "public_ip" {
  value = aws_instance.web.public_ip
}
```

### Running Terraform

Now we have some configuration ready, let's run Terraform.

First we use `init` to download any providers we have configured.

```
terraform init
```

Then we can view the plan:

```
terraform plan
```

To apply the plan, we must write it to a file:

```
terraform plan -out plan.terraform
```

And then apply:

```
terraform apply plan.terraform
```

### Connecting

After applying, we should see the `output` we created above.

We should be able to connect using `ssh ubuntu@<ip>`

### Deleting everything

To delete everything, we can delete all the `data`, `resource`, and `output`
blocks, and run `terraform plan` and `terraform apply` again.

The absense of these blocks will tell Terraform these resources we've created
are no longer needed.
