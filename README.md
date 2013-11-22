Aerosol
=======

Aerosol is a gem made to ease the pain of deploying. For this gem to be useful, quite a few assumptions are made about your stack:

- You are using `ActiveRecord` if you're deploying a Rails repo
- You use AWS

Getting Started
---------------

Add it to your Gemfile:

```ruby
gem 'aerosol'
```

And build an aerosol.rb

If you're not using Rails, add it to your Rakefile:

```ruby
require 'aerosol'
```

Usage
-----

### Rake Tasks

#### The separate deploy rake tasks

`aerosol:deploy_name:run_migration` - Runs the ActiveRecord migration through the SSH connection given
`aerosol:deploy_name:create_auto_scaling_group` - Creates a new auto scaling group for the current git hash
`aerosol:deploy_name:wait_for_new_instances` - Waits for instances of the new autoscaling groups to start up
`aerosol:deploy_name:stop_old_app` - Runs command to shut down the application on the old instances instead of just terminating
`aerosol:deploy_name:destroy_old_auto_scaling_groups` - Terminates instances with the current tag and different git hash
`aerosol:deploy_name:run_post_deploy` - Runs a post deploy command

#### The combinations

`aerosol:deploy_name:all` - Full serial deployment: Run mgiration, create auto scaling group, wait for instances, stop old application, destroy old auto scaling groups and run the post deploy command
`aerosol:deploy_name:all_asynch` - Same as `all` but runs the migration and create auto scaling groups in parallel

#### Non-deploy rake tasks

`aerosol:ssh:deploy_name` - Prints out ssh command to all instances of the latest deploy

### CLI

`aerosol ssh deploy_name` - Same as aerosol:ssh:deploy_name
`aerosol ssh -r deploy_name` - Runs the ssh command to the first instance available (still prints out the others)

The DSL
-------

The DSL is broken down into multiple objects, all of which conform to a specific format. Each object starts with the name of the section,
followed by a name for the object you're creating, and a block for configuration.

```ruby
auto_scaling :test_auto_scaling do
  # code here
end
```

Each object has an enumeration of valid attributes. The following code sets the `max_size` attribute in a `auto_scaling` group called `test_auto_scaling`:

```ruby
auto_scaling :test_auto_scaling do
  max_size 1
end
```

Finally, each object has zero or more valid references to other DSL objects. The following code sets `auto_scaling` that references a `launch_configuration`:

```ruby
launch_configuration :my_launch_config do
  max_size 1
end

auto_scaling :my_auto_scaling do
  launch_configuration :my_launch_config
end
```

Below is an alternative syntax that accomplishes the same thing:

```ruby
auto_scaling :my_auto_scaling do
  launch_configuration do
    max_size 1
  end
end
```

Network Design
--------------

A simplified version of the AWS network design that Aerosol manages is: auto scaling groups control running instances, and determine how those instances start up.

Aerosol defines a way to deploy to an auto scaling group, with the correct launch configuration and properly brings up the instances.

Controlling SSH connections
---------------------------

SSH connections are used when connecting for migrations, checking if the instances are live and running local connections.

Options:

- user - The user to connect with
- host - Where to connect to (will be filled in for instance and local connections)
- jump - A hash of a user and host to connect through. For secure/VPN connections.

### Examples

Minimal SSH connection for local connections:

```ruby
ssh :local_ssh do
end
```

Since a machine could start up with your user and not need a jump server, this could be all you need.

SSH connection with a jump server:

```ruby
ssh :jump_ssh do
  jump :user => 'ec2_user', :host => 'jump.example.com'
end
```

A lot of network structures use a middle jump server before connecting to the internal network.  Supplying either a user, host or both, you can establish a jump server to connect through.

SSH connection with a user and host:

```ruby
ssh :migration_ssh do
  user 'db_user'
  host 'dbserver.example.com'
end
```

When using ActiveRecord, you'll need to supply a connection to pass through. Supply your database server credentials like this (you can also use a jump server here).

Controlling instance definition
-------------------------------

Launch configurations define the EC2 instance startup.  All the features entered via the EC2 wizard are available.

Options:

- ami - The Amazon Machine Image that designates what operating system and possible tiers to run on.
- instance_type - The tier to run the instance on (m1.large, etc.)
- security_groups - A list of security groups the instances are paired with.
- user_data - The bash script that is run as the instance comes live
- iam_role - the Identity and Access Management role the instances should startup with
- kernel_id - The kernel ID configurable for the EC2 instance (best found through the EC2 wizard)
- key_name - The name of the EC2 key pair for initial connections to the instance
- spot_price - The max hourly price to be paid for any spot prices

### Examples

Launch configurations must specify an AMI and instance type. Everything else can be ignored and the instance will start up:

```ruby
launch_configuration :minimal_lc do
  ami 'ami-1715317e' # Ubuntu 13.04 US-East-1 ebs amd64
  instance_type 'm1.small'
end
```

This instance will startup and do nothing, with almost no way to connect to it though.

Adding a script into the user_data section sets how the instance will start up. Either use plain text, read from a file, or use ERB:

Text:

```ruby
launch_configuration :startup_lc do
  ami 'ami-1715317e' # Ubuntu 13.04 US-East-1 ebs amd64
  instance_type 'm1.small'
  user_data "#!/bin/bash\n# Get file here"
end
```

File:

```ruby
launch_configuration :startup_lc do
  ami 'ami-1715317e' # Ubuntu 13.04 US-East-1 ebs amd64
  instance_type 'm1.small'
  user_data File.read('startup.sh')
end
```

ERB:

```ruby
launch_configuration :startup_lc do
  ami 'ami-1715317e' # Ubuntu 13.04 US-East-1 ebs amd64
  instance_type 'm1.small'

  thing = "google.com" # This is accessible within the ERB file!

  user_data ERB.new(File.read('startup.sh.erb')).result(binding)
end
```

Using an ERB file is likely the most powerful especially since anything defined within the launch configurations context is available, as well as any other gems required before this.

Controlling auto scaling groups
-------------------------------

Auto scaling groups define how your network expands as needed and spans multiple availability zones.

Options:

- availability_zones - A list of availability zones
- min_size/max_size - The min and max number of instances for the auto scaling group
- default_cooldown - The number of seconds after a scaling activity completes before any further trigger-related scaling activities can start
- desired_capacity - The number of instances that should be running in the group
- health_check_grace_period - The number of seconds AWS waits before it befores a health check
- health_check_type - The type of health check to perform, `'ELB'` and `'EC2'` are valid options
- load_balancer_names - A list of the names of desired load balancers
- placement_group - Physical location of your cluster placement group created in EC2
- tag - A hash of tags for the instances in the group
- launch_configuration - A reference to the launch configuration used by this auto scaling group

Auto scaling groups only require an availability zone, a min and max size, and a launch configuration.

```ruby
auto_scaling :minimal_asg do
  launch_configuration :minimal_lc
  availability_zones ['us-east-1a']
  min_size 1
  max_size 1
end
```

This will start up a single instance using our most basic launch configuration in US East 1A

Adding tags can help identify the instances (two are already made for you: Deploy and GitSha)

```ruby
auto_scaling :tagged_asg do
  launch_configuration :minimal_lc
  availability_zones ['us-east-1a']
  min_size 1
  max_size 1

  tag 'Name' => 'tagged_asg'
  tag 'NextTag' => 'NewValue'
  # or
  tag 'Name' => 'tagged_asg', 'NextTag' => 'NewValue'
end
```

Configuring your deploy
-----------------------

A deploy ties the auto scaling groups and possible SSH connections together for application logic

Options:

- auto_scaling - Auto scaling group for this deployment
- ssh - SSH connection from deployment computer to new instance
- migration_ssh - SSH connection from deployment computer to database connection instance (optional: only needed if deployment computer cannot connect to database directly)
- do_not_migrate! - Do not try and do a migration (for non-ActiveRecord deploys)
- local_ssh - SSH connection from local computer to running instances
- stop_command - This command is run before an auto scaling group is run to stop your application gracefully
- live_check - 'Local path to query on the instance to verify the application is running'
- instance_live_grace_period - The number of seconds to wait for an instance to be live (default: 30 minutes - 1800)
- db_config_path - relative path of your database config file (default: 'config/database.yml')
- app_port - which port the application is running on
- stop_app_retries - The number of times to retry stopping the application (default: 2)
- continue_if_stop_app_fails - When true, will ignore a failure when stopping the application (default: 'false')
- post_deploy_command - Command run after the deploy has finished

A lot of the options for deployment are required, but we've defined some sane defaults.

A basic deployment could look like:

```ruby
deploy :quick_app do
  ssh :jump_ssh
  do_not_migrate!
  auto_scaling :minimal_asg
  stop_command 'sudo stop app'
  live_check '/version'
  app_port 80
end
```

This won't perform a migration, will stop the application with 'sudo stop app' after checking the instance is live at 'localhost:80/version' after SSHing to the instance through jump_ssh.
It will use the default of waiting 30 minutes for a deploy before failing, and fail if the previous application does not stop correctly.

If the server you're deploying from is not behind the same restrictions a development machine is, add a local SSH connection.  The local connection will be used when generating commands for connecting to the instances.

Demo
====

```ruby
launch_configuration :launch_config do
  ami 'ami-1715317e' # Ubuntu 13.04 US-East-1 ebs amd64
  instance_type 'm1.small'
  iam_role 'role-app'
  key_name 'app'

  user_data ERB.new(File.read('startup.sh.erb')).result(binding)
end

auto_scaling :auto_scaling_group do
  availability_zones ['us-east-1a']
  max_size 1
  min_size 1
  launch_configuration :launch_config
end

ssh :ssh do
  user 'ubuntu'
end

ssh :migration do
  user 'ubuntu'
  host 'dbserver.example.com'
end

ssh :local do
  jump :user => 'ubuntu', :host => 'jump.example.com'
end

deploy :deploy do
  ssh :ssh
  migration_ssh :migration
  local_ssh :local
  auto_scaling :auto_scaling_group
  stop_command 'sudo stop app'
  live_check '/version'
  app_port 443
  post_deploy_command 'bundle exec rake postdeploycommand'
end
```

Copyright (c) 2013 Swipely, Inc. See LICENSE.txt for further details.
