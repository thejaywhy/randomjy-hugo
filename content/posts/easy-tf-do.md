+++
categories = []
date = "2015-05-28T20:12:30-04:00"
description = "Terraform makes Infrastructure easy"
keywords = ["hacker"]
title = "Terraform - that was easy!"

+++

Dyn gives engineers a couple of days every quarter to work on “off roadmap” projects. During the most recent one, I worked on various small projects, this is one of them! I wanted to experiment with [Terraform](http://terraform.io/), a tool for making your infrastructure declarable - treating it just like your code.

My example is simple, yet a great starting point; and I hope to expand on it soon. All I did was spin up a node and install NGINX. You can find the source for it [here](https://github.com/thejaywhy/easy_tf_do).

# Setup
You’ll need to set up a DigitalOcean account, if you don’t have one, you can use my [Referral Link](https://www.digitalocean.com/?refcode=38b47f5765f4) to get a free credit! (I get some too).

Once you have a DO account, you’ll need to set up a “Personal Access Token” (PAT) for the API. and an SSH key setup as well. See these excellent DigitalOcean tutorials for how to do that:

- [How To Use the DigitalOcean API v2](https://www.digitalocean.com/community/tutorials/how-to-use-the-digitalocean-api-v2#HowToGenerateaPersonalAccessToken)
- [How To Use SSH Keys with DigitalOcean Droplets](https://www.digitalocean.com/community/tutorials/how-to-use-ssh-keys-with-digitalocean-droplets)

And of course, you’ll want to install Terraform. You can use the [downloads page](http://www.terraform.io/downloads.html) to get the appropriate package for your platform. I’m assuming you’re running from not Windows (sorry).

My Terraform configuration assumes that the configuration variables are available via environment variables:

```
$ export TF_VAR_do_token={your_PAT}
$ export TF_VAR_ssh_fingerprint=$(ssh-keygen -lf ~/.ssh/id_rsa.pub | awk '{print $2}')
$ export TF_VAR_pub_key=$HOME/.ssh/do_rsa.pub
$ export TF_VAR_pvt_key=$HOME/.ssh/do_rsa
```

Alternatively, you can pass in the variables to the Terraform command line interface:

```
$ export DO_PAT={your_PAT}
$ export SSH_FINGERPRINT=$(ssh-keygen -lf ~/.ssh/id_rsa.pub | awk '{print $2}')
$ terraform plan \
  -var "do_token=${DO_PAT}" \
  -var "pub_key=$HOME/.ssh/do_rsa.pub" \
  -var "pvt_key=$HOME/.ssh/do_rsa" \
  -var "ssh_fingerprint=$SSH_FINGERPRINT"
```

# Usage
Now that you’ve’ got your DO access setup and your environment configured, you’re ready to go! Clone this repo if you haven’t done so already:

```
$ git clone https://github.com/thejaywhy/easy_tf_do.git
```

Now to the fun part, planning out your deployment:

```
$ terraform plan

+ digitalocean_droplet.web
    image:                "" => "ubuntu-14-04-x64"
    ipv4_address:         "" => "<computed>"
    ipv4_address_private: "" => "<computed>"
    ipv6_address:         "" => "<computed>"
    ipv6_address_private: "" => "<computed>"
    locked:               "" => "<computed>"
    name:                 "" => "web-1"
    region:               "" => "nyc3"
    size:                 "" => "512mb"
    ssh_keys.#:           "" => "1"
    ssh_keys.0:           "" => "${ssh_fingerprint}"
    status:               "" => "<computed>"
```

Which shows you exactly what this Terraform configuration is going to do. To apply it, all you have to do is run:

```
$ terraform apply
# ...bunch of output....
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

The state of your infrastructure has been saved to the path
below. This state is required to modify and destroy your
infrastructure, so keep it safe. To inspect the complete state
use the `terraform show` command.

State path: terraform.tfstate

Outputs:

  ipv4 = <droplet public ip>
```

You should now be able to visit the IP address and see the standard NGINX welcome screen. Hooooray!

{{< figure src="/img/nginx_hooray.png" alt="NGINX - Hoooray!" >}}


## Cleaning up
I’ll assume you don’t want this super simple droplet to be eating into your wallet, you’ll probably want to shut it down! You could use the Web UI or another API wrapper, or use Terraform, it’ll be easy.

```
$ terraform plan -destroy
- digitalocean_droplet.web
$ terraform destroy
Do you really want to destroy?
  Terraform will delete all your managed infrastructure.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes

digitalocean_droplet.web: Refreshing state... (ID: 5364117)
digitalocean_droplet.web: Destroying...
digitalocean_droplet.web: Destruction complete

Apply complete! Resources: 0 added, 0 changed, 1 destroyed.
And that’s it! Super simple to make your infrastructure with code!
```

# Infrastructure as Code
That was fun and all, but lets look at the actual code to see how we were able to spin up a DO droplet so easily.

In the file `digitalocean.tf` we specify the variables that we’ll use throughout our system:

```
variable "do_token" {}
variable "pub_key" {}
variable "pvt_key" {}
variable "ssh_fingerprint" {}

provider "digitalocean" {
  token = "${var.do_token}"
}
```

These variables, you may notice are the environment variables specified above in the Setup section.

We’ve also specified a provider, `"digitalocean"`. Terraform supports many of your favorite cloud services, such as AWS, Google Compute Engine, CloudFlare, Heroku, and more.

One provider that is interesting is the Template provider. It lets you specify a template as part of your configuration, and pass variables to it. Terraform will render the file on another resource so that it can be consumed later. This is a step towards managing your configuration data with Terraform.

The other file we have, `www.tf`, is a bit more complicated, and where the magic really happens.

```
# Create a new Web droplet in the nyc2 region
resource "digitalocean_droplet" "web" {
    image = "ubuntu-14-04-x64"
    name = "web-1"
    region = "nyc3"
    size = "512mb"
    ssh_keys = [
      "${var.ssh_fingerprint}"
    ]

    connection {
      user = "root"
      type = "ssh"
      key_file = "${var.pvt_key}"
      timeout = "2m"
    }

    provisioner "remote-exec" {
      inline = [
        "export PATH=$PATH:/usr/bin",
        # install nginx
        "sudo apt-get update",
        "sudo apt-get -y install nginx"
      ]
    }
}

output "ipv4" {
    value = "${digitalocean_droplet.web.ipv4_address}"
}
```

In this file we define a `"digital_ocean"` `resource` and name it `web`. Within the resource we specify what kind of droplet we want, and where it should go. There are two sub-blocks defined, `connection` and `provisioner`.

The `connection` block tells Terraform how to connect to the resource, in this case via ssh. It also relies on the `ssh_keys` field, which will place our public key on the node. We need the `connection` specified before we can run our `provisioner`.

The `provisioner` is a remote-exec type, but there are several others available, including a Chef implementation that will call out to your Chef server and converge against your run lists. The remote-exec resource is using a ssh connection to create a remote terminal, and then run commands to install NGINX - just like you would if you had SSH’d in to the remote node.

There’s another block included in the file, `output`, that I used just to print out the IPv4 address of the droplet after a Terraform run. This just makes it easier to debug for the time being. We could however later add another resource to create a DNS A record that points at the IPv4 address. Maybe next time!

# Conclusion
Terraform has made it _super simple_ to define your infrastructure as code, so why wouldn’t you? The Terraform CLI also provides a uniform way to manage your infrastructure across multiple cloud providers. No more worry about finding the right screen from a provider’s web UI, or writing your own API clients!
