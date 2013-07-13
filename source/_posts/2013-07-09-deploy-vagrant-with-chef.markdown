---
layout: post
title: "Deploy vagrant box with chef-solo"
date: 2013-07-09 15:02
comments: true
categories:
---

## Prerequisite

Below content is referenced from http://leopard.in.ua/2013/01/04/chef-solo-getting-started-part-1/, who has provided an excellent tutorial for chef-solo.

My example code is [here](https://github.com/Josephu/chef-solo-example/tree/1.0).

## Setup the kitchen with knife-solo

Let's create a directory for our kitchen,
```
$ mkdir chef-solo
$ cd chef-solo
```
... and setup knife-solo and berkshelf with bundler.

```
$ vim Gemfile
	source :rubygems

	gem 'knife-solo'
	gem 'berkshelf'

$ bundle install
```
And setup the kitchen with knife.

```
$ knife solo init .
```

## Setup with berkshelf

The official [website](http://berkshelf.com/ "Berkshelf") and [presentation](http://www.slideshare.net/opscode/the-berkshelf-way-20882903 "The Berk's way")  has some good intro. Let's initialize berkshelf, setup Berksfile by [reference](https://github.com/metastudio/chef-rails/blob/master/Berksfile), and install the cookbooks. Note that berkshelf works much like bundler.

```
$ berks init chef-solo

$ cat Berksfile
	site :opscode

	# Community cookbooks
	cookbook 'ohai'
	cookbook 'apt'
	cookbook 'build-essential'
	cookbook 'sudo'
	cookbook 'git'

	# Ruby
	cookbook 'rvm', git: 'https://github.com/fnichol/chef-rvm.git'

	# Server
	cookbook 'nginx'

	# Database
	cookbook 'mysql'

$ berks install --path cookbooks # install cookbooks in ~/.berkshelf/cookbook to ./cookbooks directory
```

## Setup Vagrant

You can get a box from [Here](http://www.vagrantbox.es/ "Vagrant boxes"). Let's use *precise32*, and configure Vagrantfile.

```
$ vagrant box add berk-precise32 http://files.vagrantup.com/precise32.box

$ cat Vagrantfile

	## Configure box

	  # Every Vagrant virtual environment requires a box to build off of.
		config.vm.box = "berk-precise32"

		# The url from where the 'config.vm.box' box will be fetched if it
		# doesn't already exist on the user's system.
		config.vm.box_url = "http://files.vagrantup.com/precise32.box"

		# Setup IP for private network
		config.vm.network :private_network, ip: "33.33.33.10"

	## Update Chef

	  # Precise32 uses chef v10.4 as default, but many cookbook need chef v11, eg. apt
	  # chef.json reference => https://coderwall.com/p/r4lv7w
	  config.vm.provision :shell, :inline => "gem install chef --version 11.4.2 --no-rdoc --no-ri"

	## Disable berkshelf

		# Disable berkshelf overwriting cookbook path,
		# using traditional cookbook/site-cookbook methodology
		config.berkshelf.enabled = false

```

[Plugin](https://github.com/riotgames/vagrant-berkshelf) need to be installed to use berkshelf in vagrant.

```
$ vagrant plugin install vagrant-berkshelf
```

Finally, we need to setup chef in Vagrantfile. Note that every time we run *vagrant provision*, this code will run again.

Lets say we want to have rvm with ruby 1.9.3, nginx compiled in source, git and mysql.

```

  config.vm.provision :chef_solo do |chef|
    chef.cookbooks_path = ["site-cookbooks", "cookbooks"]

    chef.json = {
      "run_list" => [
        "recipe[apt]", # Required for rvm to work properly.
        "recipe[git]",
        "recipe[rvm::vagrant]",
        "recipe[rvm::system]",
        "recipe[nginx::source]",
        "recipe[mysql::server]"
      ],
      "rvm" =>{
        :default_ruby => "ruby-1.9.3-p429"
      },
      "nginx" => {
          "default_site_enabled" => false,
          "version" => "1.4.0",
          "source" => {
             "prefix" => "/usr/local/nginx",
             "modules" =>["http_gzip_static_module", "http_ssl_module"]
          },
      },
      "mysql" => {
        "server_root_password" => "1234qwer",
        "server_repl_password" => "1234qwer",
        "server_debian_password" => "1234qwer"
      }
    }
  end
```

Let's start vagrant, and provision vagrant later if needed.

```
$ vagrant up # this starts VM and run provision

$ vagrant provision # this rerun provision defined in Vagrantfile
```

So finally, test your browser with 33.33.33.10, you should be able to see nginx 1.4.0 with 404 page.

