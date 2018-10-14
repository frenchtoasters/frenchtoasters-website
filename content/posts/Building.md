+++
categories = ["ci/cd","gitlab","etcd","kubernetes","rancher"]
date = "2018-10-05T09:32:37-04:00"
tags = ["gitlab","rancher","kubernetes"]
title = "buildingthething"

+++


# Building something in CI/CD

In this post im am going to start exploring what all can be done with CI/CD pipelines in reguards to Kubernetes. So most people have been using CI/CD to deploy out their applications or modify their application that is running to fix a bug or address some feature request. But what can us infrastructure guys do with this? Well everyone seems to preach these days the infrastructure as code right? I dont really know where im going to stop with this post cause I could just cram in all everything into one or break it up but let me tell you there is a lot of stuff we can do.

## Building repeatable deployments

The first kinda key aspect of CI/CD pipelines is that through these pipelines you can script out basically any kind of deployment task that would be needed to deploy whatever application, infrastruce, or website that you want. What does that really mean though? Well lets start with an example of how to deploy out a blank kubernetes cluster. 

First we need to think about "Ok what do we need to do to manually build out this cluster?" so lets break it down into some steps:

* Setup deployment environment with the tools that are needed for deployment
* Configure the deployment
* Execute the deployment

### Setup Deployment Environment

So here we need to gather requirments for what all it takes to deploy out a cluster. Frist you need to think about what he environement needs to look like so you can even deploy the cluster. This is where I think about the basic OS features that are required to even be able to use the tools that are available to deploy out your cluster. So first we need to be able to deploy hosts that are actually going to run our kuberenetes cluster so we are going to need tools like aws-cli, vcd-cli, or any other cloud providers cli tool. Not all cloud providers are created equal though so need to first ensure that our cloud provider acutally has a cli tool, isll get to why in a second, and with that tool we are actually able to deploy hosts of a specific image. The reason we need a cli tool to do this is that with CI/CD things we need to have a robot be able to just run some command and wait for an expected output, and if you think about it GUI's are just for humans to interface with something. Robots just care about 1's and 0's so a gui is wayyyyy more than they would ever need. 

Ahh one thing that I havent really pointed out here is that with CI/CD pipelines you are essentinally just building a container image of the ideal deployment workstation a human would use to deploy whatever. So with that in mind we have an idea of how we are going to deploy host we need to think about what the requirments of our deployment tools are to be able to do something with these hosts. At a very low level we should think, how does the tool connect to the hosts? In most cases its going to SSH to them so we will need a tool like openssh-clients to give our container the ability to use ssh. SSH is really the only way that I know of how these tools configure these host for deployment so that is really it. 

Now all we need after that for a base is the deployment tool. All the tools that you would need for this are usally available on github or through some url that you can directly download it from. Oh wait so how do we download that as a robot? wget is an awesome tool that if given a direct link to the download you can download things from the command line, so lets add that to the list of tools we need to install in the container. Ok now we have the deployment tool downloaded we need some kinda configuration for the tool to apply. So how do we get that as a robot? Well we dont, not directly.

### Setup Deployment configuration

So this part is something that robots are not really good at doing, yet at least. So lets do some leg work here our selves and see just how much work we need to pre stage to be able to do this. 

First we need to figure out what our base configuration will be for that we need to consult some documentation for me im going to look to this doc [from Rancher here](https://rancher.com/docs/rancher/v2.x/en/installation/ha/kubernetes-rke/). In this document the first thing they have is this yaml:

```
nodes:
  - address: 165.227.114.63
    user: ubuntu
    role: [controlplane,worker,etcd]
    ssh_key_path: ~/.ssh/id_rsa
  - address: 165.227.116.167
    user: ubuntu
    role: [controlplane,worker,etcd]
    ssh_key_path: ~/.ssh/id_rsa
  - address: 165.227.127.226
    user: ubuntu
    role: [controlplane,worker,etcd]
    ssh_key_path: ~/.ssh/id_rsa
```

So this is just a list of nodes, their roles, and the user/ssh key combo we need to be able to access them. Ok great but unless we want to put more work in to create a VPC with static ip addresses or pre-stage our nodes we are not going to know the IP address that they come up with. So do we just stop here and say robots cant do this part we need to do the leg work here our selves? Maybe that would be the case if we had not been around linux for a long enough time, but if you've ever spent the time to really learn about the awesomeness of configuration files and built-in tools in the linux tool box you might have stumbled upon some really useful ones for the issue right here. 

Ill cut to the case a bit here and just go ahead and list the linus tools that you will be using to build this file on the fly:

* find - does exactly what you would think finds stuff via the linux command line `find / -name file.name` that right there will search the `/` filesystem for `file.name` 
* sed - this is where the magic comes from. [sed](https://linux.die.net/man/1/sed) is something that not everyone knows about but is basically a regex tool for the linux command line
* grep - the best linux tool you will ever use [grep](https://linux.die.net/man/1/grep)
* tr - how we will remove what we dont need, also super handy [tr](https://linux.die.net/man/1/tr)

So the next question is how do we combine these tools to make it work for us? Well first you must look at the cli tool for your cloud provider. 99.99999999% of the time the tool that allows you to deploy these hosts will also allow you to query your VPC for information about these hosts... I think you might see where im going with this. The following is an example from the [vcd-cli documentation](https://github.com/vmware/vcd-cli)

```
 $ vcd login myserviceprovider.com org1 usr1 --password ******** -w -i
    usr1 logged in, org: 'org1', vdc: 'vdc1'

    $ vcd catalog create catalog1
    task: 893bff31-4bf6-48a6-84b8-55cee97e8349, Created Catalog catalog1(cc0a2b88-9e5a-4391-936f-df6e7902504b), result: success

    $ vcd catalog upload catalog1 photon-custom-hw11-2.0-304b817.ova
    upload 113,169,920 of 113,169,920 bytes, 100%
    property    value
    ----------  ----------------------------------
    file        photon-custom-hw11-2.0-304b817.ova
    size        113207424

    $ vcd vapp create vapp1 --catalog catalog1 --template photon-custom-hw11-2.0-304b817.ova \
      --network net1 --accept-all-eulas
    task: 0f98685a-d11c-41d0-8de4-d3d4efad183a, Created Virtual Application vapp1(8fd8e774-d8b3-42ab-800c-a4992cca1fc2), result: success

    $ vcd vapp list
    isDeployed    isEnabled      memoryAllocationMB  name      numberOfCpus    numberOfVMs  ownerName    status        storageKB  vdcName
    ------------  -----------  --------------------  ------  --------------  -------------  -----------  ----------  -----------  ---------
    true          true                         2048  vapp1                1              1  usr1         POWERED_ON     16777216  vdc1

    $ vcd vapp info vapp1
    property                     value
    ---------------------------  -------------------------------------
    name                         vapp1
    owner                        ['usr1']
    status                       Powered on
    vapp-net-1                   net1
    vapp-net-1-mode              bridged
    vm-1: 1 virtual CPU(s)       1
    vm-1: 2048 MB of memory      2,048
    vm-1: Hard disk 1            17,179,869,184 byte
    vm-1: Network adapter 0      DHCP: 10.150.221.213
    vm-1: computer-name          PhotonOS-001
    vm-1: password               ********
```

Well would you look at that, we deployed a host from an `ova` image of our choosing. Then we listed information about it like the IP of `Network Adapter 1`. So I think you can see where im going with this... Yep thats right we are going to deploy these hosts then take the information from the output of those commands to describe it and pipe them into the config file for our cluster. So lets look at the linux commands that will allow us to do this:

```
$ vcd vapp info vapp1 | grep "Network adapter 0"
vm-1: Network adapter 0      DHCP: 10.150.221.213
```

Nice that gives us the line but we just need a small piece of that line so lets change that up some:

```
$ vcd vapp info vapp1 | grep "Network adapter 0" | tr -d '[:space:]' | awk -F':' '{print $3}'
10.150.221.213
```

And there we go we are now just grabbing the IP of our host. Things to note about this is that it _only_ works for the output of the vcd-cli commands, for aws-cli the output information will look different so we would need to tailor that command slightly to account for how that would look. 

So now you might be asking, "Ok great wtf does that give us?" well lets look back at the linux command line and remember this concept called "Environment Variables". They give us a way to set values and use a "human friendly" name to address them. So lets look back at that command from above and instead of just printing the value lets make it an Environment Variable:

```
$ NODE1=$(vcd vapp info vapp1 | grep "Network adapter 0" | tr -d '[:space:]' | awk -F':' '{print $3}')
$ echo $NODE1
10.150.221.213
```

Victory is ours!!! We now have a way to set an environment variable within our container to our dynamically allocated IP address of our host. So what is next with this? Well next we are goind to need to change up our base rancher configuration like so:

```
nodes:
  - address: NODE1
    user: ubuntu
    role: [controlplane,worker,etcd]
    ssh_key_path: ~/.ssh/id_rsa
  - address: NODE2
    user: ubuntu
    role: [controlplane,worker,etcd]
    ssh_key_path: ~/.ssh/id_rsa
  - address: NODE3
    user: ubuntu
    role: [controlplane,worker,etcd]
    ssh_key_path: ~/.ssh/id_rsa
```

Ok so now that you've seen that can you guess where im going with this? Well if you cant or are still reading this we are going to edit this configuration file using another classic linux tool `sed`. 

```
$ sed -i "s/NODE1/${NODE1}/g" rancher-config.yml |  sed -i "s/NODE2/${NODE2}/g" rancher-config.yml |  sed -i "s/NODE3/${NODE3}/g" rancher-config.yml
```

There are couple things id like to note about the above command, the `-i` flag tells sed to just edit the file instead of its default behavior to complete recreate the file. Also really important here is the `""` around the regex used for the `sed` command that is how you get `sed` to put the actual value of the environment variable and not just the straight word. After we have run this command though our `rancher-config.yml` file would look something like this:

```
nodes:
  - address: 10.150.221.213
    user: ubuntu
    role: [controlplane,worker,etcd]
    ssh_key_path: ~/.ssh/id_rsa
  - address: 10.150.221.214
    user: ubuntu
    role: [controlplane,worker,etcd]
    ssh_key_path: ~/.ssh/id_rsa
  - address: 10.150.221.215
    user: ubuntu
    role: [controlplane,worker,etcd]
    ssh_key_path: ~/.ssh/id_rsa
```

## Create our CI/CD pipeline

So in the above sections we have covered lots of small things that gave us an understanding of what environment specific tools we will need for our ideal deployment workstation would look like, how we would like to deploy our hosts for our cluster, and how we would be able to dynamically create our `rancher-config.yml` based on our newly deployed hosts. We now need to tie all of that together into a nice CI/CD pipeline so our robot overlords can take over and the "hard work" for us, to deploy the cluster.

So frist we need to pick a CI/CD framework to use, in my case im going to what is built into gitlab. Depending on which one you pick/are required to use the syntax/style might be slightly different but the basic idea of this should still be the same. First things first we are going to need a git repo to hold our `rancher-config.yml` and within gitlab we will also need to set some CI/CD Variables, mainly we are going to need to set our `SSH_PRIVATE_KEY` variable so that we can actually ssh into our hosts. With AWS you pick the SSH key file you want to use when deploying the hosts however with vCD I didnt see this option when deploying the hosts so I just had to copy that file over later on not a big deal just a few extra lines in our `.gitlab-ci.yml`. Speaking of `.gitlab-ci.yml` lets see what that would look like for our deployment, since it along with the `rancher-confi.yml` are going to be the only files in our git repo.

```
##
## Set container image to use
##
image: ubuntu:16.04 

##
## Create stages for gitlab runner to go through
##
stages:
  - build_cluster 
  - validate_cluster

build_deployment:
  stage: build_cluster
  script:
    ## 
    ## Install dependices 
    ##
    - apt-get -y update && && apt-get -y upgrade && apt-get -y install openssh-client wget python-pip sshpass
    - pip install --user vcd-cli
    - wget https://github.com/rancher/rke/releases/download/v0.1.9/rke_linux-amd64

    ## 
    ## Set file permissions 
    ##
    - chmod +x rke_linux-amd64

    ##
    ## Create ~/.ssh directory and blank id_rsa and id_rsa.pub files
    ##
    - mkdir -p ~/.ssh
    - touch ~/.ssh/id_rsa
    - touch ~/.ssh/id_rsa.pub
    - chmod 600 ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa.pub

    ##
    ## Populate ssh key files with CI/CD variables
    ##
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' >> ~/.ssh/id_rsa
    - echo "$SSH_PUBLIC_KEY" >> ~/.ssh/id_rsa.pub

    ##
    ## Deploy three node cluster in vCD instance 
    ##
    - vcd login $VCD_CLOUD_PROVIDER_URL $VCD_ORG $VCD_USER --password $VCD_PASSWORD -w -i
    - vcd vapp create vapp1 --catalog $VCD_CATALOG_NAME --template $VCD_TEMPLATE_NAME --network net1 --accept-all-eulas
    - vcd vapp create vapp2 --catalog $VCD_CATALOG_NAME --template $VCD_TEMPLATE_NAME --network net1 --accept-all-eulas
    - vcd vapp create vapp3 --catalog $VCD_CATALOG_NAME --template $VCD_TEMPLATE_NAME --network net1 --accept-all-eulas

    ##
    ## Set dynamic NODE environment variables and PASSNODE variables
    ##
    - NODE1=$(vcd vapp info vapp1 | grep "Network adapter 0" | tr -d '[:space:]' | awk -F':' '{print $3}')
    - NODE2=$(vcd vapp info vapp2 | grep "Network adapter 0" | tr -d '[:space:]' | awk -F':' '{print $3}')
    - NODE3=$(vcd vapp info vapp3 | grep "Network adapter 0" | tr -d '[:space:]' | awk -F':' '{print $3}')
    - PASSNODE1=$(vcd vapp info vapp1 | grep "password" | tr -d '[:space:]' | awk -F'd' '{print $2}')
    - PASSNODE2=$(vcd vapp info vapp2 | grep "password" | tr -d '[:space:]' | awk -F'd' '{print $2}')
    - PASSNODE3=$(vcd vapp info vapp3 | grep "password" | tr -d '[:space:]' | awk -F'd' '{print $2}')

    ##
    ## Fill in our rancher-config.yml with NODE environment variables
    ##
    - sed -i "s/NODE1/${NODE1}/g" rancher-config.yml |  sed -i "s/NODE2/${NODE2}/g" rancher-config.yml |  sed -i "s/NODE3/${NODE3}/g" rancher-config.yml

    ##
    ## Using sshpass and PASSNODE values to copy id_rsa.pub to newly deployed hosts in vCD. I know not the must secure but gotta get them there some how
    ## NOTE: If you were using AWS you wouldnt need to do this the key file would be specified on deployment
    ##
    - echo $PASSNODE1 >> passnode1
    - echo $PASSNODE2 >> passnode2
    - echo $PASSNODE3 >> passnode3
    - sshpass -f passnode1 ssh-copy-id -o StrictHostKeyChecking=no ubuntu@$NODE1
    - sshpass -f passnode2 ssh-copy-id -o StrictHostKeyChecking=no ubuntu@$NODE2
    - sshpass -f passnode3 ssh-copy-id -o StrictHostKeyChecking=no ubuntu@$NODE3

    ##
    ## Standup rancher cluster
    ##
    - ./rke_linux-amd64 up --config rancher-config.yml
  artifacts:
    paths:
    - kube_config_rancher-config.yml

validate_deployment:
  stage: validate_cluster
  script:
    ##
    ## Install dependices
    ##
    - apt-get -y update && apt-get -y upgrade && apt-get install -y apt-transport-https
    - curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
    - touch /etc/apt/sources.list.d/kubernetes.list 
    - echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
    - apt-get update
    - apt-get install -y kubectl

    ##
    ## Set KUBECONFIG for kubectl
    ##
    - export KUBECONFIG=$(pwd)/kube_config_rancher-config.yml

    ##
    ## Validate Cluster, you can do more this is enough for me
    ##
    - kubectl get nodes
  artifacts:
    paths:
    - kube_config_rancher-config.yml
```

There we go! We now have our `.gitlab-ci.yml` file that will do our deployment for us. Ya there are some "goofy" things that are not really production worthy where we have to pass around some ssh keys, but remember this container will be deleted after we are done so thats good right? Ok we are not quite done if you notice in the `.gitlab-ci.yml` there are a few variables that we are using that need to be filled in on our repo. So if you are using gitlab go to your repo and navigate to Settings -> CI/CD. Under there we have a Variables section and you guessed it we are going to fill in our variables there. The ones we need to fill in are the following:

```
SSH_PRIVATE_KEY
SSH_PUBLIC_KEY
VCD_CLOUD_PROVIDER_URL
VCD_ORG 
VCD_USER
VCD_PASSWORD
VCD_CATALOG_NAME
VCD_TEMPLATE_NAME
```
***NOTE*** I did already have the $VCD_CATALOG_NAME and $VCD_TEMPLATE_NAME already create in my vCD org so I could just pass them here, you can create/upload those in this `.gitlab-ci.yml` however odds are you already have those in your vCD org already. 

Next we just need to push all this to our git repo and sit back and watch it work. Now in some cases this is just fine others you might want to specify in your `.gitlab-ci.yml` some conditions on when to do certian things here. Like only validate on merges to the master branch and only deploy when there is a push to say a deploy branch or based on some tags for more info on that check out the documentation [here](https://docs.gitlab.com/ce/ci/yaml/README.html#only-and-except-simplified).

# Conclusions

If you are a small company, lazy infrastructure dude, or just an automation guy CI/CD is for you. Learn it, live it, love it. It removes most of the human errors that happen in deployments and does not deviate from the deploy steps that are outlined. It has the power of a human but none of the fatigue when it comes to deploying/editing/destorying anything a human could from a linux command line. Embrace our robot overlords!!!!
