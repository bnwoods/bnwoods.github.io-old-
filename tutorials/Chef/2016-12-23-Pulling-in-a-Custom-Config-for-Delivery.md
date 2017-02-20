---
layout: tutorial
title: Pulling in a Custom Config for Workflow
tutorial-category: chef
---
This one is alarmingly simple, so it will be really short. Do you have a custom config for delivery to pull in custom build cookbooks from version control? Do you skip certain phases? If so, here is an easy solution to pull in your customized file every time you run `delivery init`.

##### Put your custom config file somewhere you can find it
I have put mine in my .delivery directory that houses my cli.toml. You can literally put a copy of your custom config anywhere you want.

##### Modify your cli.toml
You will need to modify your cli.toml to point to your new config.json file. Mine looks something like this:

```
api_protocol = "https"
enterprise = "enterpriseName"
git_port = "8989"
organization = "organization-name"
pipeline = "master"
server = "automateservername.domain.com"
user = "username"
config_json = "/Users/username/chef-repo/.delivery/config.json"
```

Now, next time you run `delivery init`, instead of using the default config.json you will now be pulling in your custom config.json file!
