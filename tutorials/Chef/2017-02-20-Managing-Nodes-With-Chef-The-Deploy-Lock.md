---
layout: tutorial
title: Managing Nodes with Chef - The Chef-Client Run Lock with Consul
description: How to use Consul K/V store to control automated chef-client runs
tutorial-category: chef
---

So you want to manage nodes with Chef, but your organization is nervous about automatic changes happening across all nodes in the organization at random. This story is not a story about a single organization, but rather a story that many organizations face.

Chef is *GREAT* at managing nodes, and if you aren't using it in that way I encourage you to do so. But sometimes the thought of a system making changes to nodes in a _random_ way is just too much for the powers that be. What if a breaking change is introduced? What if that breaking change makes it all the way to production? Does it have the potential to take down an entire organization, which means dollars lost and uptime crucified? I am 100% of the belief that you should trust your code, trust your rollback plan. But for those that need the peace of mind, and even those that are new to Chef and would like to safeguard themselves against themselves I give you Consul Key Value storage.

Through utilization of the Key/Value store feature in Consul, you can lock out nodes from Chef convergence within a series of parameters. The parameters in question is the "path".

Say you have a dev, test, and production environment. Within that environment you have groups of servers. You know that you only want one server in that group to go down at a time _just in case everything gets weird_... The following is a way to make that happen, guaranteed, while running Chef on an automated interval cycle.

First things first, you will need to setup your Consul Servers. Hashicorp has provided great documentation on just that, which you can find <a href="https://www.consul.io/intro/getting-started/agent.html" target="_blank">here</a>

Next, you will need to determine how you want to split up your nodes. For the purposes of this article, we're going to split these up on the dev, test, and production level. You can, of course, get more granular with this and say "I only one a single node in a given cluster to converge at a time". This is highly customizable. In order to be able to "key" off of a value, you'll want to have attributes set at build time that define your criteria. In our example, we're going to set this value to <code>default[node_level] = "dev"</code>.

Keep in mind, you will need to run the consul _agent_ on all of your nodes. These will all be set up as "client", while those systems that belong to your Consul cluster will need to be set up as "server".

Next, you will need to write a cookbook that is applied to each node's run-list. In this cookbook you'll need to put the following into your default recipe to "acquire a lock" at the start of each Chef run.

``` ruby
chef_gem 'diplomat' do
  action :install
end

require 'diplomat'

node_fqdn = node['fqdn']
path_for_lock = "/#{node['node_level']}/lock"
path_for_value = "/#{node['node_level']}/value"

Chef.event_handler do
  on :converge_start do
    if ::File.exist?('/opt/consul')
      acquired = false
      while !acquired
        Chef::Log.info("Acquiring session from Consul")
        sessionid = Diplomat::Session.create({:Name => "consul_lock_start"})
        Chef::Log.info("Acquiring lock from Consul on #{path_for_lock} using sessionid #{sessionid}")
        Diploat::Lock.wait_to_acquire(path_for_lock, sessionid)
        Chef::Log.info("Lock Acquired")
        lock_owner = Diplomat::Kv.get(path_for_value, nil, :return)
        acquired = (lock_owner == node_fqdn or lock_owner.nil? or lock_owner.empty?)
        Chef::Log.info("Another owner found for lock while attempting to acquire using session id #{sessionid}. The owner is #{lock_owner}")

        if !acquired
          Chef::Log.warn("Lock owner #{lock_owner} is currently doing work and has a value in #{path_for_value}. Releasing our lock and retrying")
          Diplomat::Lock.release(path_for_lock, sessionid)
          Diplomat::Session.destroy(sessionid)
          sleep(60)
          next
        end

        Chef::Log.info("Lock acquired in Consul. #{path_for_value} is being set to #{node_fqdn}")
        Diplomat::Kv.put(path_for_value, node_fqdn, release: sessionid)
        Chef::Log.info("Session lock being released on #{path_for_lock}")
        Diplomat::Lock.release(path_for_lock, sessionid)
        Diplomat::Session.destroy(sessionid)
      end
    end
  end
end
```

Keep in mind, there is not a way to "abort" a Chef run (at least, not a way that I have been able to find). This leaves two options for a system like this: either wait to acquire a lock or fail the run because it cannot acquire a lock. In this example, I have opted to show the "wait to acquire" option. I believe that failing the Chef run based on expected behavior skews reporting data. You will also need to keep in mind that the above example doesn't necessarily scale well. I'll explain...
The way the above example is written, every node you have on the dev level will be fighting for a single lock. The same can be said for the test level and production levels. If you are running chef-client every hour for example, and each chef-client run takes on average 1 minute and you have 75 dev level nodes, someone is going to get skipped over. Extrapolate that over the randomness of the chef-client run queue, and you could very quickly have several nodes that will not check in indefinitely. Finding the sweet spot for runtime as well as tweaking the <code>path_for_value</code> and <code>path_for_lock</code> is critical to the success of this solution.

Now, in your default recipe, you'll add the following to release the lock value at the end of the run:

``` ruby
Chef.event_handler do
  on :converge_complete do
    if ::File.exist?('/opt/consul/')
      Chef::Log.info("chef-client run complete. Cleaning up the lock values.")
      sleep(10)
      sessionid = Diplomat::Session.create({:Name => "consul_lock_end"})
      lock_acquired = Diplomat::Lock.wait_to_acquire(path_for_lock, sessionid)
      if not lock_acquired
        Diplomat::Session.destroy(sessionid)
        raise Excpetion.new("Unable to acquire a lock")
      end

      lock_owner = Diplomat::Kv.get(path_for_value, nil, :return)
      if lock_owner != node_fqdn and not lock_owner.nil? and not lock_owner.empty?
        Diplomat::Lock.release(path_for_lock, sessionid)
        Diplomat::Session.destroy(sessionid)
        raise Exception.new("Node #{lock_owner} is currently working at #{path_for_value}")
      end
      Diplomat::Kv.put(path_for_value, "")
      sleep(10)
      Diplomat::Lock.release(path_for_lock, sessionid)
      Diplomat::Session.destroy(sessionid)
    end
  end
end
```

What does this all mean? At the beginning of each chef-client run, you will acquire a "lock" from Consul. Once that lock has been acquired other nodes will not be able to run chef-client until the node that owns lock has finished it's run and released it.

#### The Downsides (that I can think of off-hand)
1. You're only allowing a single Chef run to happen at a time which means you'll have to complete an exercise in finesse to get the run interval just right

2. Nodes that try to check in when a lock is already taken will wait indefinitely to be able to acquire a lock.

3. If a chef-client run fails, other nodes trying to obtain a lock along that same lock/value path will not be able to check in until the failing node releases the lock. This is by design, but can be an inconvenience if the nodes are unrelated and could otherwise check in simultaneously.


Is this the only solution? Absolutely not. This is just one of many options to gain some control over when chef-client runs. This is a very complicated solution, but it's one that does work for what it's supposed to work for. Consul is a pretty handy tool to have in your tool-belt and is capable of many things outside of Key/Value storage. If you haven't, check it out. Props to Hashicorp!
