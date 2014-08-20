---
title: Deploy NodeJS project with Capistrano
layout: post
published: true
comments: true
---

## Install Capistrano

You need to have **ruby** and **gem** installed in order to install **capistrano**.

```bash
$ gem install capistrano
$ gem install capistrano-ext
```

## Server

We need to configure the destination server in order to deploy our application.

### Create a deploy user

The first step is to create this user :

```bash
$ adduser deploy
```
Then we need to change the user's password to an untypable string, guaranteeing that the user has no password which can be used to log in.

```bash
$ passwd -l deploy
```

#### SSH key

In order to deploy with the *deploy* user we need to have an ssh key in its authorized keys.

On your local machine :

```bash
$ ssh-keygen -t rsa -C "test@example.com"
```
You'll be prompted for a passphrase. Enter one and remember it.

In order to not be prompted every time the passphrase you just entered we can use the ssh agent.
We can see which keys are loaded in the SSH agent by running :

```bash
$ ssh-add -l
```

If you don't see any keys listed, you can simply run ssh-add:

```bash
$ ssh-add
Identity added: /home/me/.ssh/id_rsa (/home/me/.ssh/id_rsa)
```
Now that the key is loaded into the agent you can print it by doing so :

```bash
$ ssh-add -L
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCiULq......qnBZr test@example.com
```

Now we need to put this key into the *deploy* user `/home/deploy/.ssh/authorized_keys` file on the server.

```bash
me@localhost $ ssh root@remote
root@remote $ su - deploy
deploy@remote $ cd ~
deploy@remote $ mkdir .ssh
deploy@remote $ echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCiULq........qnBZr test@example.com" >> .ssh/authorized_keys
deploy@remote $ chmod 700 .ssh
deploy@remote $ chmod 600 .ssh/authorized_keys
```

If everything went good, we should now be able to do something like this:

```bash
me@localhost $ ssh deploy@one-of-my-servers.com 'hostname; uptime'
one-of-my-servers.com
19:23:32 up 62 days, 44 min, 1 user, load average: 0.00, 0.01, 0.05
```

More info on this section on [capistrano website](http://capistranorb.com/documentation/getting-started/authentication-and-authorisation/)

### Nodejs and npm

If it's not already the case, we have to install **nodejs** and **npm** on our server.
**Git** has to be install too in order for Capistrano to clone the repo.


```
sudo apt-get update
sudo apt-get install nodejs npm
sudo apt-get install git
```

If your server is running on ubuntu you might need to create a symlink for node like so `sudo ln -s /usr/bin/nodejs /usr/bin/node`.
More info on [Stack Overflow](https://stackoverflow.com/questions/18130164/nodejs-vs-node-on-ubuntu-12-04)

### Forever

We are going to use **forever** for nodejs which make sure a nodejs process will always stays up.

```
npm install forever -g
```


## Init capistrano for the project

Go to your project and init capistrano :

```
cd /path/to/your/project
cap install
```

This will create the following files :

```
├── Capfile
├── config
│   ├── deploy
│   │   ├── production.rb
│   │   └── staging.rb
│   └── deploy.rb
└── lib
    └── capistrano
            └── tasks
```

>You need to have you project under git with a remote repository in order for Capistrano to clone it on the destination server.

Now that we have the different files generated we are going to edit our configuration.

In `config/deploy.rb` edit the following line to your need :

```ruby
set :application, 'test_app'
set :repo_url, 'git@github.com:username/test_app.git'
set :deploy_to, '/var/www/test_app'
```
The above lines indicate :

- what's the name of your application
- the location of your git repository 
- where to deploy in the destination server.

Make sure that :

- the destination folders exist on your server
- your *deploy* user has the right to write in the destination folder
 
If it not the case let's create it and give permissions to *deploy*. Run the following command on your server :

```bash
mkdir -p /var/www/test_app
chown -R deploy:deploy /var/www/test_app
```  

Then replace into `config/deploy.rb` the `namespace :deploy do` section with the following one :

```ruby
namespace :deploy do

  desc "Stop Forever"
  task :started do
    on roles(:app) do
      execute "forever stopall", raise_on_non_zero_exit: false
    end
  end
 
  desc "Install node modules non-globally"
  task :npm_install do
    on roles(:app) do
      execute "cd #{current_path} && npm install"
    end
  end
 
  desc 'Restart application'
  task :restart do
    on roles(:app), in: :sequence, wait: 5 do
	 execute  :default_env['NODE_ENV'], " forever start #{current_path}/app.js", raise_on_non_zero_exit: true
    end
  end
 
  after :publishing, :restart
  before :restart, :npm_install
 
end

```

The section above will describe 3 tasks for the deploy namespace:

- stop every forever instance on the server (it's a bit brutal and not advised if you have multiple nodejs instance on the server using forever)
- install the different modules required for the project
- start the application

You can see that we defined a very basic rule at the end of the section. It basically says:
>Once you are done publishing then restart the project, but before you can restart the project, you have to install the dependencies.



#### Production configuration
We are almost done, we need to configure the `config/deploy/production.rb` where we are going to write the specificity for the production environment.

Replace the content of your file with the following :

```ruby
role :app, %w{deploy@0.0.0.0}

server '0.0.0.0', user: 'deploy', roles: %w{app}

set :default_env, {
  'NODE_ENV' => 'production'
}

set :ssh_options, {
    forward_agent: true,
}

```

In this file we :
 
- define the role for *app* and we define a server with the role we created above. We also specify the user that we are going to use for the deployment (here: *deploy*).
- set the default production environment.
- forward our ssh key during deployment in order to be able to fetch the git repo.

## Deploy

Now we should be ready to deploy.

Run the following command :

```bash
$ cap production deploy
INFO[b54f6d4a] Running /usr/bin/env mkdir -p /tmp/test_app/ on 0.0.0.0
DEBUG[b54f6d4a] Command: ( NODE_ENV=production /usr/bin/env mkdir -p /tmp/test_app/ )
INFO[b54f6d4a] Finished in 0.972 seconds with exit status 0 (successful).
DEBUG Uploading /tmp/test_app/git-ssh.sh 0.0%
INFO Uploading /tmp/test_app/git-ssh.sh 100.0%
.....
DEBUG[8f30fec2] Command: echo "Branch master (at 11d2638) deployed as release 20140818152948 by wendannor" >> /var/www/test_app/revisions.log
INFO[8f30fec2] Finished in 0.037 seconds with exit status 0 (successful).
```

Hope this post helped you deploy your nodejs application with Capistrano.

##Sources :
[http://capistranorb.com/](http://capistranorb.com/)
