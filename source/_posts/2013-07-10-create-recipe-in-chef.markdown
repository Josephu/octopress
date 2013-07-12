---
layout: post
title: "Create Recipe in Chef"
date: 2013-07-10 14:43
comments: true
categories: 
---

## Prerequisite

Below flow is referenced from http://leopard.in.ua/2013/01/05/chef-solo-getting-started-part-2/, who has provided an excellent tutorial for newcomers. 

## Create a cookbook

Prepare a cookbook structure in site-cookbooks. Note that site-cookbooks are for customized cookbooks.

```
$ cd site-cookbooks
$ mkdir tatoo
$ cd tatoo
$ mkdir recipes; 
$ mkdir files; mkdir files/default
$ mkdir templates; mkdir templates/default
$ cat default.rb
  package "git"

```

And add tatoo to recipe in runlist in Vagrantfile, and necessary parameters

```
    chef.json = {
      "run_list" => [
...
        "recipe[tatoo]" # a customize cookbook
      ],
```

Now let's start vagrant, and the new cookbook should be loaded.
```
$ vagrant up
```

## Create a nginx setting file to start a web server

First, let's develop our recipe file, which has the main process flow.

``` ruby recipes/default.rb
%w(public logs).each do |dir|
  directory "#{node.app.web_dir}/#{dir}" do
    owner node.user.name
    mode "0755"
    recursive true
  end
end

template "#{node.nginx.source.prefix}/conf/sites-available/#{node.app.name}.conf" do
  source "nginx.conf.erb"
  mode "0644"
end

bash "create sites-enable link" do
	code <<-EOH
		ln -s #{node.nginx.source.prefix}/conf/sites-available/#{node.app.name}.conf #{node.nginx.source.prefix}/conf/sites-enabled/#{node.app.name}.conf
	EOH
end

nginx_site "#{node.app.name}.conf"

cookbook_file "#{node.app.web_dir}/public/index.html" do
  source "index.html"
  mode 0755
  owner node.user.name
end
```

Note that:

* sites-available folder is used for multiple Vhost management, and the file name can be random. In our case we name it tatoo.conf
* cookbook_file searches file source from files/ in your cookbook directory
* nginx_site, how does this work?

Next, let's create a template file for nginx.conf. Template allows you to customize content in chef.

``` ruby templates/default/nginx.conf.erb
server {
    listen 80 default;

    access_log <%= node.app.web_dir %>/logs/nginx_access.log;
    error_log <%= node.app.web_dir %>/logs/nginx_error.log;

    keepalive_timeout 10;
    root <%= node.app.web_dir %>/public;
}
```

Next, let's create index.html in tatoo/files
``` html files/index.html
<h1>Hi!</h1>
```

Finally, we have got ourselves settled, let's add our new recipe to the runlist in Vagrantfile.

``` ruby Vagrantfile
    chef.json = {
      "run_list" => [
	...
        "recipe[tatoo]" # a customize cookbook
      ],
	...
      "app" => {
        "name" => "tatoo",
        "web_dir" => "/home/vagrant/www/apps/tatoo"
      },
      "user" => {
        "name" => "vagrant"
      }
    }

```

Now we can provision vagrant, and you should see your new site ready on 33.33.33.11 with your browser.

``` ruby
$ vagrant provision
```