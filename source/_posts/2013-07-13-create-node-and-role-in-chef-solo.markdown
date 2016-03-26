---
layout: post
title: "Create Node and Role in Chef Solo"
date: 2013-07-13 09:36
comments: true
categories: 
---

## Prerequisite

Below content is referenced from http://leopard.in.ua/2013/01/07/chef-solo-getting-started-part-3/, who has provided an excellent tutorial for chef-solo.

## Create and Run Node File

A node in chef is a server to deploy. A node file defines the work to deploy this server.

Since in previous blogs, chef.json does the work to deploy our vagrant instance, so let's first move our chef.json settings into our node file. Note that I have converted a hash format to json format, since this works nicer with chef server.

``` ruby nodes/vagrant.json
{
  "run_list": [
    "recipe[apt]",
    "recipe[git]",
    "recipe[rvm::vagrant]",
    "recipe[rvm::system]",
    "recipe[nginx::source]",
    "recipe[mysql::server]",
    "recipe[tatoo]"
    ],
  "rvm":{
    "default_ruby": "ruby-1.9.3-p429"
  },
  "nginx": {
    "default_site_enabled": true,
    "version": "1.4.0",
    "source": {
       "prefix": "/usr/local/nginx",
       "modules":["http_gzip_static_module", "http_ssl_module"]
     }
  },
  "mysql": {
    "server_root_password": "1234qwer",
    "server_repl_password": "1234qwer",
    "server_debian_password": "1234qwer"
  },
  "app": {
    "name": "tatoo",
    "web_dir": "/home/vagrant/www/apps/tatoo"
  },
  "user": {
    "name": "vagrant"
  }
}

```

Next, we need to add a parsing script to parse vagrant.json in our Vagrant file. Note that I have replaced the content in chef.json with the parsed result. Note that you need to include 'json' library to use JSON.

``` ruby Vagrantfile
require 'json'

...
  config.vm.provision :chef_solo do |chef|
    chef.cookbooks_path = ["site-cookbooks", "cookbooks"]

    chef.json = json_config = JSON.parse(File.open(File.dirname(__FILE__)+'/nodes/vagrant.json',"rb").read)    json_config['run_list'].each do |recipe|
      chef.add_recipe(recipe)
    end if json_config['run_list']

  end
```

## Create and Run Role File

### Create Role

A role is the purpose of this server, eg. web server, memcacached server. We can name a role, and apply it in a node. Therefore we can apply a role to multiple node, so applying any change to role, will apply to these nodes. Very dry indeed.

Let's create our role file 'web.json' for web server, and update it with previous recipe list.

``` ruby roles/web.json
{
  "name": "web",
  "chef_type": "role",
  "json_class": "Chef::Role",
  "description": "The base role for systems that serve HTTP traffic",
  "default_attributes":{
    "rvm":{
      "default_ruby": "ruby-1.9.3-p429"
    },
    "nginx": {
      "default_site_enabled": true,
      "version": "1.4.0",
      "source": {
         "prefix": "/usr/local/nginx",
         "modules":["http_gzip_static_module", "http_ssl_module"]
       }
    },
    "mysql": {
      "server_root_password": "1234qwer",
      "server_repl_password": "1234qwer",
      "server_debian_password": "1234qwer"
    },
    "app": {
      "name": "tatoo",
      "web_dir": "/home/vagrant/www/apps/tatoo"
    },
    "user": {
      "name": "vagrant"
    }
  },
  "run_list": [
    "recipe[apt]",
    "recipe[git]",
    "recipe[rvm::vagrant]",
    "recipe[rvm::system]",
    "recipe[nginx::source]",
    "recipe[mysql::server]",
    "recipe[tatoo]"
  ]
}
```

### Load Role in Node

You can load recipe as well as role in node file. Let's create a new node file 'vagrant_role.json' to do this.

``` ruby nodes/vagrant_node.json
{
  "run_list": [
    "role[web]"
    ]
}
```

Note that you can load role and recipe at the same time, making it highly customizable.

``` ruby nodes/vagrant_node.json
{
  "run_list": [
    "role[web]",
    "recipe[apt]"
    ]
}
```

### Update Vagrantfile to support load recipe and role

In vagrant, add_role is used to load a role, eg. `add_role['web']`. To support both recipe and role in run list, let's change some code. Remember to rename 'vagrant.json' to 'vagrant_role.json'.

``` ruby Vagrantfile
...
    chef.json = json_config = JSON.parse(File.open(File.dirname(__FILE__)+'/nodes/vagrant_role.json',"rb").read)
    json_config['run_list'].each do |recipe|
      params = recipe.sub(']','').split('[')
      case params[0]
      when "recipe"
        chef.add_recipe(recipe)
      when "role"
        chef.add_role(params[1])
      end
    end if json_config['run_list']
```

