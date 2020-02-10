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
