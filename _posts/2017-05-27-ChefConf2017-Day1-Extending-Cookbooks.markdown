---
layout: post
title:  ChefConf 2017 Day 1 - Extending Cookbooks
description: Pre-Conference Workshop -- Extending Cookbooks with Custom Resources and Ohai Plugins
date:   2017-05-27 14:22:00
categories: chefThings
---

I am going to consider my pre-conference workshop as Day 1 of ChefConf. For this, I decided to take the session on extending cookbooks with Custom Resources and Ohai plugins. This workshop focused mostly on custom resources, which to my delight they are no longer difficult. [Insert plug about a low barrier to entry with using Chef here]. 

## Custom Resources

Historically speaking, there were a couple ways to extend cookbooks. Those ways included HWRP (Heavy Weight Resource Providers), LWRP (Light Weight Resource Providers), and Definitions. Each way has it's advantages and disadvantages. Now though, you can simply use Custom Resources. For me, these are more straight forward and align much more with the rest of a cookbook. They follow the same syntax and structure as a recipe but allow you to easily create a resource for repeatable tasks within your cookbook.

TL;DR Custom resources make cookbooks easier to maintain.

So, equipped with my coffee, I prepared myself to dive in. 

<img src="{{ site.baseurl }}/images/IMG_0402.jpg" style="width: 50%; height: 50%;">

We started the course by learning about the other ways to extend cookbooks and their advantages and disadvantages. Then we dug in. We were provided with centOS workstations that already had a skeleton cookbook for us to work with. This cookbook just happened to be the httpd cookbook. 

Custom resources are available in version of chef-client _after_ 12.5.0. So, if you are wildly out of date with your Chef-Client updates, HWRP and LWRP are your options. Update!

The default recipe in the cookbook started out like this: 

``` ruby

package 'httpd'

file '/var/www/html/index.html' do
  content '<h1>Welcome home!</h1>
end

directory '/srv/apache/admins/html' do
  recursive true
  mode '0755'
end

template '/etc/httpd/conf.d/admins.conf' do
  source 'conf.erb'
  mode '0644'
  variables(document_root: '/srv/apache/admins/html', port: 8080)
  notifies :restart, 'service[httpd]'
end

file '/srv/apache/admins/html/index.html' do
  content '<h1>Welcome admins!</h1>'
end

service 'httpd' do
  action [:enable, :start]
end
```

So basically, we're installing the 'httpd' package, setting up some conf files and directories, and restarting the httpd service after we make those conf changes. So far so good. But what about turning that into a custom resource? 

The first step we went through was creating the directories and files required for a custom resource. From the httpd directory, you can run `chef generate lwrp RESOURCE_NAME` so in this case, we ran `chef generate lwrp vhost`. This created a directory in the cookbook called resources with a vhost.rb file. This also creates a directory called providers, which is no longer needed for custom resources. That one can be removed. 

So, in `resources/vhost.rb` we put the following code: 

``` ruby
action :create do

  directory '/srv/apache/admins/html' do
    recursive true
    mode '0755'
  end

  template '/etc/httpd/conf.d/admins.conf' do
    source 'conf.erb'
    mode '0644'
    variables(document_root: '/srv/apache/admins/html',port: 8080)
    notifies :restart, 'service[httpd]'
  end

  file '/srv/apache/admins/html/index.html' do
    content '<h1>Welcome admins!</h1>'
  end
end
```

Looks pretty familiar right? Essentially, we moved the logic from our default.rb recipe and placed it in a custom resource. That info was then removed from our default.rb recipe and instead replaced with calls to our custom resource. Our default.rb recipe at this point  is below: 

``` ruby
package 'httpd'

file '/var/www/html/index.html' do
  content '<h1>Welcome home!</h1>'
end

httpd_vhost 'admins' do
  action :create
end

service 'httpd' do
  action [:enable, :start]
end
```
Next, we made our resource accept custom parameters. To do this, we simply added in site_name, which would allow us to use this across multiple sites. In vhost.rb, we added a line at the top to accept a string, and also replaced values in the custom resource to replace with that variable. See below: 

``` ruby
property :site_name, String

action :create do
  directory "/srv/apache/#{site_name}/html" do
    recursive true
    mode '0755'
  end

  template "/etc/httpd/conf.d/#{site_name}.conf" do
    source 'conf.erb'
    mode '0644'
    variables(document_root: "/srv/apache/#{site_name}/html", port: 8080)
    notifies :restart, "service[httpd]"
  end

  file "/srv/apache/#{site_name}/html/index.html" do
    content "<h1>Welcome #{site_name}!</h1>"
  end
end
```
Adding this in, we then updated our default recipe: 

``` ruby
package 'httpd'

file '/var/www/html/index.html' do
  content '<h1>Welcome home!</h1>'
end

httpd_vhost 'admins' do
  site_name 'admins'
  action :create
end

service 'httpd' do
  action [:enable, :start]
end
```

We of course added in more to our resource, namly an `:action delete` but the point of my post is to illustrate how easy this is! I'm guilty sometimes of looking at documetnation and gettig entirely overwhelmed. I often have to put myself in the appropriate headspace to get more difficult concepts. This concept though was delightfully easy. I would recommend everyone brush up on this as I can see how useful it can actually be. 

Another great thing about Chef is that many of their training materials are available online. If you would like to go through this in full, the slide deck can be found at <a href="https://github.com/chef-training/extending_cookbooks" target="_blank">https://github.com/chef-training/extending_cookbooks</a> and the example httpd cookbook to get started with can be found at <a href="https://github.com/chef-training/extending_cookbooks-repo" target="_blank">https://github.com/chef-training/extending_cookbooks-repo</a>. 

My next post will be more focused on the many sessions I attended and the taking of my first Certification test. 

Cheers!
