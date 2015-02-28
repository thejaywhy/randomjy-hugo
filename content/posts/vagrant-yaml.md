+++
categories = []
date = "2015-02-28T14:16:19-05:00"
description = "How to modify your Vagrantfile to pull configuration data from a YAML file"
keywords = ["automation", "programming"]
title = "Store Vagrant config data in a YAML file"

+++
Have you heard of [Vagrant] (https://www.vagrantup.com/)? It's a pretty awesome way to manage VMs, containers, or even cloud instances. You can use the `Vagrantfile` to set up and configure the instance, check it into git, and share! Now you can rely on everyone to have the same starting point. Makes setting up development environments a breeze. And today I'd like to share with you something I've found to make it even more powerful.

So when you run `vagrant up` the vagrant tool is going to take and evaluate your `Vagrantfile`. Here's an example `Vagrantfile` taken from the Vagrant docs:

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "hashicorp/precise32"
end
```

You can use that to spin up an Ubuntu 12.04 (Precise) 32-bit VM. It's just that simple. Of course there are all sorts of additional options you might want to configure such as number of CPUs, memory size, or network interfaces. However, I feel that adding all of these options to the `Vagrantfile` makes for a hard read, so I found a better way!

Turns out you can just write `ruby` right there in the `Vagrantfile`, and you get access to _full ruby_. So that means you can instruct Vagrant to open other files when it evaluates the `Vagrantfile`. This means I can add my config to another file, where it'll be easier to manage. Here's my new `Vagrantfile` that reads its config from a YAML file:

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

DIR = File.dirname(__FILE__)

require "yaml"

$env_config = YAML.load_file(File.join(DIR, "env.yml"))

Vagrant.configure("2") do |config|
  config.vm.box = $env_config["box"]

  $env_config["instances"].each do |instance|
    instance_name = instance["name"]
    config.vm.define instance_name do |instance_config|

      # instance is the yaml configuration hash for the node
      # instance_config is the Vagrant configuration for the instance

      instance_config.vm.hostname = instance_name
      instance_config.vm.network :private_network, ip: instance["box_ip"]

      # set up the provider for the instance
      instance_config.vm.provider "virtualbox" do |v|
        v.customize ["modifyvm", :id, "--memory", instance["memory"]]
        v.customize ["modifyvm", :id, "--cpus", instance["cpu"]]
        v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
      end
    end
  end
end
```

Obviously what we have here is suddenly more complicated! Now we have added the ability to spin up multiple VMs, all specified from an `env.yml` file. We name each instance, assign it a private network IP, and configure the memory, CPU, and NAT settings. All of the relevant settings are taking from that YAML file, so lets take a look at it:

```YAML
instances:
  - name: database
    box: ubuntu/trusty64
    box_ip: 10.1.10.10
    memory: 512
    cpu: 2
  - name: web
    box: ubuntu/trusty64
    box_ip: 10.1.10.20
    memory: 256
    cpu: 1
```

Now you can see that our environment will have two VMs, the database and the web server. We've specified that they'll use the same base box, but have some different settings. If we find we need to add another box in the future, all we have to do is add another named instances in the `env.yml` file! Easy!

Lets test it out!

```bash
$ vagrant status
Current machine states:

database                  not created (virtualbox)
web                       not created (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
$ vagrant up
# lots of output....
$ vagrant status
Current machine states:

database                  running (virtualbox)
web                       running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
```

Vagrant will go ahead and download the ubuntu/trusty64 box for us, and then spin up the VM using virtualbox. We can then use `vagrant ssh database` or `vagrant ssh web` to SSH into our boxes and configure them. BAM. It's that simple.

Now we have a pretty common `Vagrantfile` we can use for all of our projects, and we just have to manage a YAML file. I've found that moving the node configuration out of the `Vagrantfile` and into a YAML file has decreased confusion on our team. Especially from the Python developers that have no desire to write _anything_ in ruby!
