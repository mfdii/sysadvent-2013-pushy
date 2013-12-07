Day 9 - Getting Pushy With Chef

One of the long standing issues with Chef has always been that changes we wanted to make to nodes weren’t necessarily instant. Eventually your nodes would come in sync with the recipes on your Chef server, but depending on how complex your environment was this might take a few runs to happen. 

There were ways around this, mainly by using `knife ssh` to force the nodes to instantly update, in the order you want. While this method worked, it had its own problems (needing ssh keys and sudo rights for example). A few months ago Chef released an add-on to the enterprise version that allows users to initiate actions on a node without requiring SSH access; we call this feature Push Jobs. Right now, Push Jobs (formerly known as Pushy) are a feature of Enterprise Chef, but we are working towards open sourcing Push Jobs in early 2014 (think Q1). 

Getting started with Push Jobs is fairly easy. There are 2 additional components that need to be installed, the Push Jobs server and the Push Jobs clients. The Push Jobs server sits along side your Erchef server, either on the same machine or a separate host. The Push Jobs clients can be installed using the push-jobs cookbook. There is a [copious amount of documentation](http://docs.opscode.com/install_push_jobs.html) covering the installation, so I won’t cover that in detail. The two things you need to know is how to allow commands to be executed via Push Jobs, and how to start jobs. 

First, commands to be executed is controlled by a “whitelist” attribute. The push-jobs cookbook sets the `node['push_jobs']['whitelist']` attribute and writes a configuration file `/etc/chef/push-jobs-client.rb`. The `node['push_jobs']['whitelist']` attribute is used in this config file to determine what commands can be ran on a node. 

For example, if you want to add the ability to restart tomcat on nodes with the tomcat role, add this to the role:

```json
 "default_attributes": {
    "push_jobs": {
      "whitelist": {
        "chef-client": "chef-client",
        "apt-get-update": "apt-get update",
        "tomcat6_restart": "service tomcat6 restart"
      }
```

Second, you’ll need the ability to start jobs on nodes. This is accomplished by installing the knife-pushy plugin:

```
gem install knife-pushy-0.3.gem
```

This will give you the following new knife commands:

```
** JOB COMMANDS **
knife job list
knife job start <command> [<node> <node> ...]
knife job status <job id>

** NODE COMMANDS **
knife node status [<node> <node> ...]
```

`knife node status` will give you a list of nodes with [state detail](http://docs.opscode.com/plugin_knife_pushy.html#node-status). 

```
ricardoII:intro michael$ knife node status
1-lb-intro	available
1-tomcat-intro	available
1-tomcat2-intro	available
ricardoII:intro michael$
```

In this case, all of my nodes are available. Let’s say I want to run `chef-client` on one of my tomcat nodes. It’s as simple as:

```
knife job start chef-client 1-tomcat-intro 
```

Maybe I don’t know all the jobs I can run on a node or group of nodes. I can search for those jobs by running:

```
ricardoII:intro michael$ knife search "name:*tomcat*" -a push_jobs.whitelist
2 items found

1-tomcat2-intro:
  push_jobs.whitelist:
    apt-get-update:  apt-get update
    chef-client:     chef-client
    tomcat6_restart: service tomcat6 restart

1-tomcat-intro:
  push_jobs.whitelist:
    apt-get-update:  apt-get update
    chef-client:     chef-client
    tomcat6_restart: service tomcat6 restart
```

I can see that I have the ability to run a tomcat restart on my tomcat nodes, as I set in my attributes earlier. But since I have multiple tomcat servers, listing them on the command line could be a pain. I can use search with the `knife job start` command to find all the nodes I want based on a search string:

```
ricardoII:intro michael$ knife job start tomcat6_restart --search "name:*tomcat*"
Started.  Job ID: 6e0b432e369904e76de6e95bac99c9e6
Running (1/2 in progress) ...
Complete.
command:     tomcat6_restart
created_at:  Fri, 06 Dec 2013 21:05:32 GMT
id:          6e0b432e369904e76de6e95bac99c9e6
nodes:
  succeeded:
    1-tomcat-intro
    1-tomcat2-intro
run_timeout: 3600
status:      complete
updated_at:  Fri, 06 Dec 2013 21:05:40 GMT
```

The nice thing about using push jobs is that I can use the same credentials I use to access Chef to fire off commands on whitelisted nodes with Push Jobs enabled. I don’t need to have SSH keys for the remote nodes as I do with `knife ssh`. 

The other nice thing is that Push Jobs can be used inside recipes to orchestrate actions between machines. There is a really basic LWRP that allows for you to fire push jobs from other nodes. You can find the [LWRP on github](https://github.com/mfdii/pushy). Why would you want to do this? Say for instance your webapp hosts have autoscaled. Your chef-client run interval on your HAProxy node is 15 minutes, but you don’t want to wait (at most) 15 minutes for the new webapp to be in the pool. Your webapp recipe can fire off a push job to have the HAProxy node run `chef-client`. 

```
pushy "chef-client" do
  action :run 
  nodes [ "1-lb-tomcat"]
end
```

This cross-node orchestration doesn’t have to be part of a full-fledged chef run. You could use it to simply restart a whitelisted service if needed. 

Hopefully this gives you a good idea on how to use Push Jobs, why it’s different than knife ssh, and gets you excited for the upcoming open source release. At Chef our goal is to provide our community with the primitive resources that they can use to make their jobs more delightful. Push Jobs are the first release of primitives to better orchestrate things like Continuous Delivery, Continuous Integration, and more complex use cases. 
