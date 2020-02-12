# Kubernetes yet another way...

After reading `Kuberenetes the hard way` I started trying out various different ways to deploy a Kuberenetes cluster on various different cloud providers. Durning my exploration into how to deploy Kuberentes cluster ive found tons and tons of different methods and tools that aid in doing just that. This post is going to cover how one might deploy an `RKE` Kuberentes cluster using `terraform` and `Gitlab`, and some of the things that I have found from doing so. 

## Terraform

Terraform is a hot topic these days as its been used and proven at scale by a countless number of _cloud services_ companys these days, often the ones that get big investments to further develop their service. So naturally we would want to use a tool that is quickly becoming the industry standard for deploying services in a _highly repeatble_ pattern via command line utility. That last part is really where the magic comes from for `terraform` being able to deploy it via a command line with some variable inputs form a `.tfvars` or `.tfvars.json` file, the latter being in `JSON` format. Along with easy variable inputs `terrafrom` also has a concept of `providers` of which these providers are really just interfaces for which and user can gather `data` or define `resources` in a standized format across _all_ supported providers. What this gives us is a single high level configuration language that allows us to define and deploy data or resouces to any supported provider. 

## Gitlab

As you might know Gitlab is version control, which naturally is something that we would want when coding but its really so much more for us in this project. For us its going to become our operator for performaing our `CRUD` operations upon an `RKE` cluster. The method for which it will allow us to do this is via its built-in CI/CD pipelines, all we have to do is define out the process that we want it to complete. 

## The MEAT!!!

So now that we have level set on what all we are going to us for these operations lets talk about how we are going to do all this. 

### RKE

RKE stands for Rancher Kubernetes Engine, but for us its really just an interface that was developed that allows us to deploy a kuberenetes cluster from a `yaml` definition file. The ability to do just is that is what makes it so perfect for something like terraform. As all we would really have to do is translate a terraform resource defintion into an object to pass along to the RKE binary for deployment. Lucky Rancher themselves have already written this for us, https://github.com/rancher/terraform-provider-rke. However as of writting this post it has _not_ been added to the terraform standard library of providers so we have to complie and add it to our terraform path. I wont cover this process here but the README.md for the provider covers the steps to build the provider and once built you need to add the plugin following the steps outlined here, https://www.terraform.io/docs/plugins/basics.html#installing-a-plugin. 

### Nodes

One of the things that the RKE provider doesnt and shouldnt do for you is deploy the nodes that will be used to run your kuberenetes cluster. Lucky though that is exactly what terraform was designed to do and there are plenty of examples out there of how deploy nodes to just about every cloud provider there is. For this post im just going to use AWS as its quick, dirty, and easy to do. 

Fist things first though you will want to clone down this repo, https://rebot. Now that we have done that lets take a look at what all it contains:

```bash
tree of repo
```

Since we were talking about deploying nodes in AWS lets look at the nodes folder:

```bash
tree of repo/nodes
```

Now there are several files here and since terraform is not the main subject of this post im not going to cover them all in detail but I will explain the general idea that is going on here. So first before we look at `nodes.tf` we need to look at `ami.tf` this is the file that tells terraform what AWS AMI image we are going to use for deploying our nodes. First we define some of the basic things that all AWS deployments _should_ have, availability zone and security group. Next is what we are really looking for the `aws_instance` this is what defines out our nodes for the cluster so lets take a closer look at it:

```terraform
resource "aws_instance" "rke-node" {
  count = 4

  ami                    = data.aws_ami.ubuntu.id
  instance_type          = var.instance_type
  key_name               = aws_key_pair.rke-node-key.id
  iam_instance_profile   = aws_iam_instance_profile.rke-aws.name
  vpc_security_group_ids = [aws_security_group.allow-all.id]
  tags                   = local.cluster_id_tag

  provisioner "remote-exec" {
    connection {
      host        = coalesce(self.public_ip, self.private_ip)
      type        = "ssh"
      user        = "ubuntu"
      private_key = tls_private_key.node-key.private_key_pem
    }

    inline = [
      "curl ${var.docker_install_url} | sh",
      "sudo usermod -a -G docker ubuntu",
    ]
  }
}
``` 

One of the key things there is `count` it defines how many nodes we are going to deploy for the cluster. Next is `ami` this defines what image we are going to use for the nodes. Lastly the most important that I see is the `key_name` this defines what ssh key is going to be able to access these nodes by default. Something that is required for us to be able to run our `RKE` provisioner and deploy the cluster. Its also worth noting that in this example we are using a barebones ubuntu install and so we have to install docker, to run our kubernetes containers, and then allow the ubuntu user to run docker commands without sudo. The way that we achive that is via the `remote-exec` terraform provider. This is part of the standard terraform library and allows us to ssh directly to a deployed node and run shell commands once the node had been deployed.

One last thing I would like to note is how we store the generated ssh key that we will use to access the cluster. This is done via the `local_file` terraform resource, this allows us to create a local file with the content of any terraform variable. Here is an example:

```terraform
resource "local_file" "rke_cluster_key" {
  filename = format("%s/%s" , path.root, "rke_cluster_key")
  content = tls_private_key.node-key.private_key_pem
}
```

## Cluster!!!

Now that we have talked about the cluster nodes and how deploy them its time to talk about what probably drew you to this post, the RKE cluster. As we have already seen with terraform we would define the cluster as a resource. In this resource we need to define at minimum the nodes that will be used for the cluster, this is done like so:

```terraform
locals {
	cluster_nodes = {
		for node in module.nodes.addresses:
			node => module.nodes.[*].internal_ips
	}
}

resource "rke_cluster" "cluster" {
  cloud_provider {
    name = "aws"
  }

  dynamic nodes {
	for_each = local.cluster_nodes
    content {
		address = nodes.key
		internal_address = nodes.value
		user = module.nodes.username
		ssh_key = module.nodes.private_key
		role = ["controlplane", "etcd", "worker"]
	}
  }
 
  services {
	etcd {
      backup_config {
      	interval_hours = "6"
      	retention = "24"
      }
	}
  }
}

```

So now that you have seen that you probably have some questions about what all is going on here. Starting from the top we define a local map called `nodes` this isnt something that is required however it allows us to loop over all nodes that were created without having to edit the cluster terraform if we want to grow or shrink the cluster size. This map has a key of the public dns entry created for the node by AWS with the value of its internal address within your VPC network. 

Next up is the big boy, the cluster resource here we first define the cloud provider that we are using for the cluster in this case AWS. RKE does support other cloud providers however I will not cover those in this post you can look it up here, https://github.com/rancher/terraform-provider-rke/blob/master/website/docs/r/cluster.html.markdown. Next is where we get kinda fancy with our terraform by using a dynamic nested block. We use the local `cluster_nodes` variable that we created as the variable we are looping over and fill in the basic content that is required to define a node. Things to note here are the `role` section here we are defining _all_ rules for all of the nodes, in other deployments you probably dont want to have your control plane nodes also be your worker nodes. Otherwise you are going to have to bring workloads down when you need to upgrade which you should expect to have to happen at some point. Next we define something that you might not always need a `Services` block, the block above defines an etcd snapshot configuration so that we can quickly restore the cluster from a backup in the event of a diaster.

## Conclusion

Well that is it, the only thing left is to fill out the terraform.tfvars file with the correct data for your environment and you will have a cluster. I showed you how I have deployed my RKE clusters in production and some of the configuration changes that I made however there is so much more that you can do with this. In addition to that you should now have some good examples of terraform code that you can modify slightly to fit with just about anything you are doing. So what is next? Well give it a try and let me know where my shit examples in this post are broken since I wrote this mostly from memory im sure there are issues if you were to copy pasta it all somewhere.  
