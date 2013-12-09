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

The other nice thing is that Push Jobs can be used inside recipes to orchestrate actions between machines. There is a really basic [Lightweight Resource Provider](http://docs.opscode.com/lwrp.html) that allows for you to fire push jobs from other nodes. You can find the [LWRP on github](https://github.com/mfdii/pushy). Why would you want to do this? Say for instance your webapp hosts have autoscaled. Your chef-client run interval on your HAProxy node is 15 minutes, but you don’t want to wait (at most) 15 minutes for the new webapp to be in the pool. Your webapp recipe can fire off a push job to have the HAProxy node run `chef-client`. 

```
pushy "chef-client" do
  action :run 
  nodes [ "1-lb-tomcat"]
end
```

This cross-node orchestration doesn’t have to be a full-fledged chef run as seen above. You could use it to simply restart a whitelisted service if needed. Let's look at a tomcat recipe for Ubuntu that allows us to find our HAProxy server and start a chef-client run.

```
include_recipe "java"

# define our packages to install
tomcat_pkgs =  ["tomcat6","tomcat6-admin"]

tomcat_pkgs.each do |pkg|
  package pkg do
    action :install
  end
end

#setup the service to run
service "tomcat" do
  service_name "tomcat6"
  supports :restart => true, :reload => true, :status => true
  action [:enable, :start]
end

#intsall templates for the configs
template "/etc/default/tomcat6" do
  source "default_tomcat6.erb"
  owner "root"
  group "root"
  mode "0644"
  notifies :restart, resources(:service => "tomcat")
end

template "/etc/tomcat6/server.xml" do
  source "server.xml.erb"
  owner "root"
  group "root"
  mode "0644"
  notifies :restart, resources(:service => "tomcat")
end

#search for our LB based on role and current environment, just return the name
pool_members = partial_search("node", "role:tomcat_fe_lb AND chef_environment:#{node.chef_environment}", :keys => {
               'name' => ['name']
               }) || []

pool_members.map! do |member|
  member['hostname']
end

#run chef-client on the LB, don't wait
pushy "chef-client-delay" do
  action :run
  wait false
  nodes pool_members.uniq
end
```

The real magic of this recipe is in the last several lines. First we query to get our loadbalancer, then we use this query result to execute the job `chef-client-delay` on the nodes that are part of our query result. The nodes attribute can be one node, or an array of nodes that you want the job to execute on.

If you run a `knife job list` you should see your job running on the load balancer.

```
ricardoII:intro michael$ knife job list
command:     chef-client-delay
created_at:  Mon, 09 Dec 2013 06:09:05 GMT
id:          6e0b432e3699018834ee0e199309e8a7
run_timeout: 3600
status:      running
updated_at:  Mon, 09 Dec 2013 06:09:05 GMT
```

You can take the job id and query the job status by running `knife job status <id>` as so:

```
ricardoII:intro michael$ knife job status 6e0b432e3699018834ee0e199309e8a7
command:     chef-client-delay
created_at:  Mon, 09 Dec 2013 06:09:05 GMT
id:          6e0b432e3699018834ee0e199309e8a7
nodes:
  succeeded: 1-lb-intro
run_timeout: 3600
status:      complete
updated_at:  Mon, 09 Dec 2013 06:10:07 GMT
```

Or course, the the Chef API can also be used to directly query the results of a job, get a listing of jobs, or start jobs. The API is [fully documented](http://docs.opscode.com/api_pushy.html), and accessible under your normal Chef API endpoint.

Hopefully this gives you a good idea on how to use Push Jobs, why it’s different than knife ssh, and gets you excited for the upcoming open source release. At Chef our goal is to provide our community with the primitive resources that they can use to make their jobs more delightful. Push Jobs are the first release of primitives to better orchestrate things like Continuous Delivery, Continuous Integration, and more complex use cases. 
