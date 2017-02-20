---
layout: tutorial
title: Creating a Custom Cookbook Generator that Doesn't Interfere with Workflow
tutorial-category: chef
---
In this post, I plan to discuss how to create a custom cookbook generator using <a href="https://github.com/echohack/pan" target="_blank">Pan by Echohack</a>. While there are a few tutorials out there on how to make it work for YOU, I found that getting it to work as intended all while still allowing `delivery init` commands to work properly required a few extra steps. The intent is to bring it all together into one place.

Before I get started, one walkthrough I found particularly helpful came from Chef's own <a href="https://gist.github.com/andy-dufour/3eb74ccdcd29e4c4afd1" target="_blank">Andy Dufour</a>. Follow his steps to get started with a basic cookbook generator.

## What should I do to make it work with workflow?
To use this and still be able to use delivery init, you'll want to add in either your custom build cookbook or the default skeleton build cookbook in the generated cookbook. Why? Because without it, `delivery init` will attempt to generate a skeleton build cookbook -- which with the generator does not exist. Since we're pointing to the generator specifically in our knife.rb, it attempts to utilize it instead of the inbuilt one for generating the build cookbook skeleton.

### Step 1:
Under `flavor_cookbooks/[yourcustomname]_base` you will need to add the following /directories/files if you're simply using a cookbook skeleton:

``` bash
  ├── flavor_cookbooks
  │   └── saucepan_base
  │       ├── files
  │       │   ├── .delivery
  │       │   │   ├── build_cookbook
  │       │   │   │   ├── .kitchen.yml
  │       │   │   │   ├── Berksfile
  │       │   │   │   ├── LICENSE
  │       │   │   │   ├── README.md
  │       │   │   │   ├── chefignore
  │       │   │   │   ├── data_bags
  │       │   │   │   │   └── keys
  │       │   │   │   │       └── delivery_builder_keys.json
  │       │   │   │   ├── metadata.rb
  │       │   │   │   ├── recipes
  │       │   │   │   │   ├── default.rb
  │       │   │   │   │   ├── deploy.rb
  │       │   │   │   │   ├── functional.rb
  │       │   │   │   │   ├── lint.rb
  │       │   │   │   │   ├── provision.rb
  │       │   │   │   │   ├── publish.rb
  │       │   │   │   │   ├── quality.rb
  │       │   │   │   │   ├── security.rb
  │       │   │   │   │   ├── smoke.rb
  │       │   │   │   │   ├── syntax.rb
  │       │   │   │   │   └── unit.rb
  │       │   │   │   ├── secrets
  │       │   │   │   │   └── fakey-mcfakerton
  │       │   │   │   └── test
  │       │   │   │       └── fixtures
  │       │   │   │           └── cookbooks
  │       │   │   │               └── test
  │       │   │   │                   ├── metadata.rb
  │       │   │   │                   └── recipes
  │       │   │   │                       └── default.rb
  │       │   │   └── config.json
```


For me, I am generating a skeleton build cookbook because I'm utilizing a custom config.json to call a custom build cookbook that is stored in version control. As long as this cookbook is called build_cookbook, `delivery init` will recognize that a build cookbook already exists and will not fail due to your generator.

### Step 2:
Next, you will need to update your cookbook.rb under `flavor_cookbooks/saucepan_base/recipes/cookbook.rb` to include all the files we've just added. Mine looks something like this (you will of course need to modify to fit your needs):

``` ruby
  context = ChefDK::Generator.context
  cookbook_dir = File.join(context.cookbook_root, context.cookbook_name)
  attribute_context = context.cookbook_name.gsub(/-/, '_')

  # Create cookbook directories
  cookbook_directories = [
    'attributes',
    'recipes',
    'templates/default',
    'test/integration/default',
    '.delivery/build_cookbook',
    '.delivery/build_cookbook/data_bags/keys',
    '.delivery/build_cookbook/recipes',
    '.delivery/build_cookbook/secrets'
  ]
  cookbook_directories.each do |dir|
    directory File.join(cookbook_dir, dir) do
      recursive true
    end
  end

  # Create basic files
  files_basic = %w(
    chefignore
    Gemfile
    LICENSE
    recipes/default.rb
    attributes/default.rb
    .delivery/build_cookbook/data_bags/keys/delivery_builder_keys.json
    .delivery/build_cookbook/recipes/default.rb
    .delivery/build_cookbook/recipes/deploy.rb
    .delivery/build_cookbook/recipes/functional.rb
    .delivery/build_cookbook/recipes/lint.rb
    .delivery/build_cookbook/recipes/provision.rb
    .delivery/build_cookbook/recipes/publish.rb
    .delivery/build_cookbook/recipes/quality.rb
    .delivery/build_cookbook/recipes/security.rb
    .delivery/build_cookbook/recipes/smoke.rb
    .delivery/build_cookbook/recipes/syntax.rb
    .delivery/build_cookbook/recipes/unit.rb
    .delivery/build_cookbook/secrets/fakey-mcfakerton
    .delivery/build_cookbook/.kitchen.yml
    .delivery/build_cookbook/Berksfile
    .delivery/build_cookbook/chefignore
    .delivery/build_cookbook/LICENSE
    .delivery/build_cookbook/metadata.rb
    .delivery/build_cookbook/README.md
  )
  files_basic.each do |file|
    cookbook_file File.join(cookbook_dir, file) do
      source file
      action :create_if_missing
    end
  end

  # Create basic files from template
  files_template = %w(
    metadata.rb
    README.md
    CHANGELOG.md
    .kitchen.yml
    Berksfile
  )
  files_template.each do |file|
    template File.join(cookbook_dir, file) do
      helpers(ChefDK::Generator::TemplateHelper)
      action :create_if_missing
    end
  end



  # Create more complex files from templates
  template "#{cookbook_dir}/attributes/default.rb" do
    source 'default_attributes.rb.erb'
    helpers(ChefDK::Generator::TemplateHelper)
    action :create_if_missing
    variables(
      attribute_context: attribute_context)
  end
  template "#{cookbook_dir}/recipes/default.rb" do
    source 'default_recipe.rb.erb'
    helpers(ChefDK::Generator::TemplateHelper)
    action :create_if_missing
  end
  template "#{cookbook_dir}/.delivery/config.json" do
    source 'custom_config.json.erb'
    helpers(ChefDK::Generator::TemplateHelper)
    action :create_if_missing
  end
  template "#{cookbook_dir}/test/integration/default/metadata.rb" do
    source 'test_metadata.rb.erb'
    helpers(ChefDK::Generator::TemplateHelper)
    action :create_if_missing
  end

  # git
  if context.have_git
    unless context.skip_git_init

      execute('initialize-git') do
        command('git init .')
        cwd cookbook_dir
      end
    end

    cookbook_file "#{cookbook_dir}/.gitignore" do
      source 'gitignore'
    end
  end
```

### Step 3:
If you haven't already, you will need to bump the version in your .gemspec file. Then, you will simply build and install (if you've already done this with the other tutorial, you'll need to do it again). Commands are as follows:

###### For Mac/Linux
`$ /opt/chefdk/embedded/bin/gem build [yourGeneratorName].gemspec`. This will generate a .gem file in your generator directory. You will then run:
`$ /opt/chefdk/embedded/bin/gem install [nameOfYourGem]-[insert.version.here].gem`.

###### For Windows
`c:\opscode\chefdk\embedded\bin\gem build [yourGeneratorName].gemspec`. This will generate a .gem file in your generator directory. You will then run:
`c:\opscode\chefdk\embedded\bin\gem install [nameOfYourGem]-[insert.version.here].gem`.


Helpful hint -- you will need to use the full path given above in your command for both Linux/Mac and Windows.


I hope you find this helpful! Until next time!
