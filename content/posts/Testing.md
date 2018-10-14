+++
categories = ["ci/cd","gitlab","etcd","kubernetes","rancher"]
date = "2018-10-14T09:32:37-04:00"
tags = ["gitlab","rancher","kubernetes"]
title = "testingdeployment"

+++

# Automated Testing

Now its time for something fun, this post is going to be taking the ideas from the previous post and giving you all an example of how they can be applied. We are going to be building out test nodes in our vCD environment, built from an valid image for running kubernetes, then taking our backups we have in our etcd gitlab repo and restoring our cluster to these new nodes. Oh and all this will be done by our robot overlords, who can always be called upon by a webhook 24/7 so we humans can restart our brains. 

## What is it we need to do?

So like in the previous post we started out by breaking down what all we needed to do in-order to achive our goal. First thing we are going to need for this posts goal is some hosts to deploy out our stuff to, ok simple enough ill just take the stuff from the previous post and paste it in. Ok but what stuff do I really need from that? Basically just the `.gitlab-ci.yml` that is the meat and potatoes of that post for a developer, I know that is what I look for first in any post I read. So lets think about what that file gives us, well all it actually does is deploy out some servers for us in vCD and _deploy_ out an rke kubernetes cluster. You see that _italics_, ("looks back for them":joy:), so if that whole post does the cluster deployment for us what is different about that vs a restore of a cluster? Ahhhh yes you thought of the same question I did! Well next we should consult the Rancher documentation on [etcd restores](https://rancher.com/docs/rke/v0.1.x/en/etcd-snapshots/#etcd-disaster-recovery), looking at that documentation it seems that all you really need to do in order to restore an rke kubernetes cluster is:

* First enable etcd snapshots, did that in the first blog post.
* Deploy out your server to restore to, did that and more in the second blog post.
* Move the etcd snapshot to the restore server, aka scp a file to folder on a linux host cake.
* Edit the `rancher-config.yml`, for the cluster we are restoring, to have the ip of the new restore server. Did that in our second blog post
* Run this one command to restore and build out the new node `rke etcd snapshot-restore --name snapshot.db --config rancher-config.yml`.
* Run the normal command to bring the cluster up `rke up --config rancher-config.yml`.
* Profit!

## Do the thang 

Ok so steps above are simple enough and we already have 3/7 things completed, 3/6 if you dont count the profit :joy:. So lets start with breaking down the `gitlab-ci.yml` we created in the last post to work with what we are wanna do here:

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

    ##
    ## Set dynamic NODE environment variables and PASSNODE variables
    ##
    - NODE1=$(vcd vapp info vapp1 | grep "Network adapter 0" | tr -d '[:space:]' | awk -F':' '{print $3}')
    - PASSNODE1=$(vcd vapp info vapp1 | grep "password" | tr -d '[:space:]' | awk -F'd' '{print $2}')

    ##
    ## Fill in our rancher-config.yml with NODE environment variables
    ##
    - sed -i "s/NODE1/${NODE1}/g" rancher-config.yml 

    ##
    ## Using sshpass and PASSNODE values to copy id_rsa.pub to newly deployed hosts in vCD. I know not the must secure but gotta get them there some how
    ## NOTE: If you were using AWS you wouldnt need to do this the key file would be specified on deployment
    ##
    - echo $PASSNODE1 >> passnode1
    - sshpass -f passnode1 ssh-copy-id -o StrictHostKeyChecking=no ubuntu@$NODE1

    ## 
    ## Copy snapshot to new host
    ##
    - scp snapshot.db ubuntu@$NODE1:/opt/rke/etcdbackup

    ##
    ## Restore rancher cluster
    ##
    - ./rke_linux-amd64 etcd snapshot-restore --name snapshot.db --config rancher-config.yml
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

Simple enough right? Well kinda sorta, since in our first blog post we only had a single node cluster we had to remove things for the second and third node. This is because the etcd backups we already have wont work for a cluster size or role change, that would come with the three node cluster. This also leads into another part of that kinda sorta, we are also going to need to put this `gitlab-ci.yml`, well at least how its currently written. Into a repo that already has this etcd snapshot, `snapshot.db`, within it. This could easily be changed to work with something like the `s3cmd` tool to keep it outside the repo for larger clusters. Ok so with the git repo requriements stated we can move on to the line where we actually added something new: 

```
    ## 
    ## Copy snapshot to new host
    ##
    - scp snapshot.db ubuntu@$NODE1:/opt/rke/etcdbackup
```

Here we are just simply copying the `snapshot.db` to the `/opt/rke/etcdbackup` folder, which you should verify that is the folder that holds your etcd backups. Assuming you are doing them... I have seen this vary based which OS you are using for your kubernetes host. 

Next change we made is actually where we specify the actual restore of the cluster it self:

```
    ##
    ## Restore rancher cluster
    ##
    - ./rke_linux-amd64 etcd snapshot-restore --name snapshot.db --config rancher-config.yml
    - ./rke_linux-amd64 up --config rancher-config.yml
```

Here you can see that its just one extra command, awwww yeahhhh. So that one command just tells `rke` to first restore etcd from a backup. Then it just runs the same command to bring the cluster up `rke up --config` and bam that is it. So with those simple changes, which are really just different steps for how to bring up the cluster, we have essentially automated the how in the recovery process of a kubernetes cluster. The where in this example is in vCD, since it was what I found, you all probably already know how or can easily find how to deploy these nodes via the some cli or api in order to get the servers for this deployed. After that its all about when which is something you all will have to figure out for your self, though the webhooks form the gitlab repo are pretty powerful. As long as you can overcome that step in your environment the robot overlords are yours to command. 

## Conclusions

Take what is already done and figure out how to reapply it else where, dont think you have to re-invent the wheel every time. The tools these's days are being built so that a human could easily do them but a robot can do them just as easily, why shouldnt it? Well some cases it might not be practical for the robot to do _everything_ but if you can break down the tasks into small repeatable steps you can create a robot to do all of those. You then just need to figure out the scheduling of it all, which there happens to be some pretty cool tools out there now to assist with that. Now go and think about some of the use cases for this type of automation in your environment and build something cool so some human brains can reboot for longer. 
