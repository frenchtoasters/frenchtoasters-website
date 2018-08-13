+++
categories = ["ci/cd","gitlab","etcd","kubernetes","rancher"]
date = "2018-08-13T09:32:37-04:00"
tags = ["gitlab","rancher","kubernetes"]
title = "cicdallthethings"

+++

# The Setup

This article is about how to configure our Rancher test ENV.

## WTF Am i doing?

This is what I want to have setup here:

- Rancher RKE Deployment in AWS with ETCD snapshots enabled writing to `/opt/rke/snapshots`
- Gitlab single node instance running on the cluster
- Turns out we have to scp it off the host first then do the rest on the bastin host
- Git init `/opt/rke/snapshots` as a repo `etcd-snapshots`
- Create cron job that runs:

```
git add .
if FRIDAY:
    git commit -m "Test Day"
else:
    git commit -m "Daily update"
git push origin master
```

- Create CI/CD pipeline in gitlab for `etcd-snapshots` repo that when `message="Test Day"` or some other logic that is better than this. 
    - Workflow:
        - Deploy RKE cluster
            - With snapshots enabled would be same config
        - From Bastin host run following:
            - SSH to all and install git
            - Run git clone of `etcd-snapshots` to get latest backup
            - run `rke etcd snapshot-restore --name ~/etcd-snapshots/latest --config cluster.yml`
            - If you get Success message good
                - Tare down everything
            - Else fail
                - Maybe leave it up for investigation? 
                - Maybe revert to previous state?
                - Tare down and notifiy?

## How am i doing this?

Well so what does this do for a person that is say, just responsable for kubernetes accessability? There job... well ok not all of it but most of the annoying shit. Really though this eliminates the need to rely on any thing other than git and kuberenetes, two things that are already pretty married at the hip. How many times have you had to tell someone, "Oh the backups are actually curropt...", "The backups werent scheduled correctly and ran over...", or "We missed that server when we rebuilt the backup server..." but hey they always open a support ticket with the vendor right? :face_palm: Git was built to manage changes in files, the snapshot of that etcd cluster you have is what again? Oh ya a file. Dont believe me, open it in vim :smile:. So if we are using this great tool as the persistent storage that for your database snapshots how are we going to trigger the actual saves? Well that is going to taken care of by a cronjob, something that has been around pretty much as long as the linux kernel. How do you get the snapshot? Its as easy as adding a few lines to your `cluster.yml`, that will create a recurring snapshot policy and bam we have our setup!

## Configuration

- Get `cluster.yml` for RKE deployment
- Deploy RKE cluster
- Deploy gitlab in container (optional) 
    - Deployment in container is up and running using the config, however i cannot push to it atm ssh key issue
    - Using gitlab.com for demo
- Configure `/opt/rke/ectd-snapshots/` as git repo in gitlab
    - note that you have to use sudo so the ssh key you are using is for root

## Requirements

- `git lfs` enabled for repo
    - We are using `git lfs` as in genral our DB is larger than the 50MB that GitLab starts complaining about when it comes to file size. Since we are using this we have a couple of limitations that come with it:
        - Max file size supported 2GB
        - This should not be a conceren those as by default the max size the etcd cluster is set to is 2GB, it wasnt till v3.3 that the limit for a max supported limit for the DB size was increased to 10GB (here)[https://coreos.com/blog/announcing-etcd-3.3]
- 12:48 start fresh dump: 12:55 finish fresh dump
    - Local connection has no effect from AWS -> Gitlab.com
    - Will have faster? speeds with hosted gitlab instance
    - Speed went from 5MB to 43KB, idk would have been faster 
- each node will be working on their own branch in the repo
    - verification happens by diff the snapshots with the same name?
    - or after we get a push from all 3 branches they are merged in order of FIFM(First branch in, is the first one merged)


## Start of Blog

Insert `ci/cd all the things meme`

What developer or engineer doesnt love `git` for version control? I cant really think of any, that got me thinking one day. Whaten all can I use `git` to store for me? Well it just so happen that at he time of that thought I was working on deploying `kubernetes` into production. I had just configured `etcd` to start taking snapshots and happen to see if I could open one up in `vim` and when it did, it hit me. Where better to store these snapshots, the life blood of a `kubernetes` cluster in the the same location where we are storing all the configuration, coomand line tools, and service deployments at. Then you know while we are at since we get a webhook to play with from that `git` repo we could create for the snapshots what can we do with it? Well why not just do the very thing we want to garuentee is we can do later using our fancy pants `CI/CD` pipelines and build out a test cluster from the snapshot. Dont worry I cant do all that in one post, so grab a beer and get to your reading spot. 

So first things first we are going to need a cluster to start backing up. So in this repo I have included the `cluster.yml` file that was used to build out my test cluster. It has enough settings to get you started on defining your own cluster. Now that is sorted we some nodes to deploy to, spin up one of the supported OS's for `rke` deployments. I choose to go with the basic `ubuntu-16.04-lts` image on AWS, this is because we are going to need to be able to install `git` on these boxes for simplicity you can get real fancy with how you run the commit say a cron scheduled container? However I first attempted this I had some issues with getting `git lfs` to run correctly with the mounted volume on the container. After all the nodes are deploy and up you are going to need to make sure in your current `PATH` you have the `rke` binary included in this repo. Then you only need to run the following to get the cluster up:

```
rke up --config /path/to/cluser.yml
```

Wait a few mins and everything should be gucci and you wil have an `rke` deployment of `kubernetes`. Next we are going to install a couple things:

* First we need to install `open-iscsi` on each cluster node.

```
sudo apt install open-iscsi
```

***NOTE*** If you used the AWS image this is already installed on all nodes

* Next we are going to need to install `Longhorn` from the catalog in the `longhorn` namespace in our cluster. This is done by first clicking `Default` or which ever project you wish from the cluster drop down. Then going to `Catalogs` at the top and searching for `longhorn` at the top right. Then you can click the deployment here and select the `namespace` you wish to install it into and the port you wish to expose for its `UI`. After that give it a few mins and then you will have a simple `Persistent Storage` provider installed on the cluster. This is just one of many different `StorageClass` drivers available for `kubernetes` also one of the simplest to install which is why I used it for this article. Once all the containers are up for this new service we can then start making `Persistent Volume Claims` with our `kubernetes` services. Can you guess what the first one we are going to install is?

You may have guessed it, `GitLab`, so using the `gitlab.yaml` that is included in this repo you should be able to deploy a `GitLab` instance with `80`,`443`, and `22`. You may need to make some tweaks on how you have these `nodeports` exposed to the outside world based on which loadbalancer you are working with. This `StatefulSet` deployment will also create `3 PVC` claims which will trigger `longhorn` to create, by default, `3` replicas of each claim. After those are created the main container will start up and go through its inital configuration. It is important to note that Gitlyou will need to pass the flag which enables `git lfs` within your `GitLab` server. After all that is finsihed you should have `GitLab` deployed with the key directories mounted to `PVC` claims and ready for you to create the repo we are going to be storing to.

I dont really feel like writing here how to get the repo configured as its kinda outside this demo, so we will skip that here and assume your repo is configured. Now you may have seen that in the `cluster.yml` we configured some `etcd` snapshot guidelines. These tell the cluster to take an `etcd-snapshot` of the cluster at a given interval and keep a certian amount of them at `/opt/rke/etcd-snapshots`. So guess where we are going? Thats right, fire up tmux and ssh to each of the different nodes in our cluster and we are going to do the following to `git init` the `/opt/rke/etcd-snapshots` folder as out target for the `git repo`:

* We are first going create an ssh-key for the `root` user on each node:

```
sudo ssh-keygen
#Hit enter a bunch
```

* Then we need to run `cat /root/.ssh/id_rsa.pub` copy the output and add all the keys as a deployment keys to the `git repo`

* Now that it is added we need to run the following in `/opt/rke/etcd-snapshots` on each node

```
git init
git remote add origin git@gitlab.frenchtoastman.com:root/etcd-snapshots
git lfs track /opt/rke/etcd-snapshots
git add .
git commit -m "Seed"
git push -u origin master
```

* Wait about 5-10 mins, this varies based on the size of the cluster

* After the above completes on each node you are going to run the following to give each node their own branch to push to:

```
git checkout -b node-name
git add .
git commit -m "node branch seed"
git push -u origin node-name
```

After that compeletes on each of or nodes we now have our full seed of each `etcd` snapshot backup. All that is left to do is build out our `CI/CD` pipeline for what we want to do! Here is where we can get creative and do some fancy things, this is also where im going to call it on this first post. The next post about this will have to contian all the cool stuff we can do with these backups in `git`, which could be quite a few things. 