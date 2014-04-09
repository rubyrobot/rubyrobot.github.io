---
layout: post
title: "Secure Rails deployment and Passwords: Best practices"
date: 2014-04-08 00:23:23 +0200
comments: true
categories:
---

*This article explains how to setup and deploy a Ruby on Rails application on your Linux servers with reasonable security measures and without making deployment any less convenient.*

I assume you are using Apache and Phusion Passenger. If you use another combination, you may have to adapt a little.

In short, you want to:

* Keep your database passwords and application's secrets out of version control and make them readable to the application only.
* Have a different database user for each application with access to it's respective database only.
* Access your remote servers via SSH without typing passwords or compromising your private key.

## Keeping Your Database Passwords Out of Version Control

I see people promoting the idea to keep the database password in an environment variable. It's easy and tempting, but have you ever seen how Passenger prints out your environmental variables if a startup error occurs?

The Linux environment was never intended to hold sensitive information and you can't rely on it not being exposed by server processes.

### Can You Keep a Secret?

By any means - keeping a secret is a problem that Linux file permissions are designed for, so why not put the secrets in a file only readable by the application. You can even put a serialized Ruby hash with SMTP credentials in that file.

On your Linux server, create a user named after your application and a file to hold the secrets:

``` bash
$ sudo useradd alice
$ sudo mkdir /home/alice
$ sudo touch /home/alice/.rails_secrets.yml
$ sudo chown -R alice:alice /home/alice
$ sudo chmod 400 /home/alice/.rails_secrets.yml
```

Inside that file, provide your database password and any other secrets you may have:
``` yaml /home/alice/.rails_secrets.yml
---
DB_PASS: secret1234
SMTP:
  address: my.mailserver.com
  port: 587
  authentication: cram_md5
  password: secret4321
  [...]
```

Now add this you your application.rb:

``` ruby
secrets_file = "/home/alice/.rails_secrets.yml"
SECRET = File.exists?(secrets_file) ? YAML.load_file(secrets_file) : {}
```

Finally tell Passenger to run as your application's user. This needs *PassengerUserSwitching* enabled (default) and your Apache to spawn from a root process (default i.e. on Ubuntu). In your Apache virtual host container:

``` apache
PassengerUser alice
PassengerGroup alice
```

That's it. Now you can access your secrets anywhere in your application, i.e. in your database.yml:

``` erb
production:
  adapter: mysql2
  encoding: utf8
  reconnect: true
  database: alice_production
  username: alice
  password: <%= SECRET["DB_PASS"] %>
  socket: /var/run/mysqld/mysqld.sock
```

## Restrict Database Access on Application Level

This is a short tip but it can't be stressed enough.

If you don't keep seperate database accounts for each application and restrict access to their respective database, just one weak PHP script on your server vulnerable to SQL injection can compromise all other applications.

## Best Practices For SSH Authentication

I like to use one single SSH key to access all user accounts on all servers from all my computers. Since the key challenge mechanism never exposes your private key, it's not less secure than having separate keys per account. But it's really up to you how much granularity in access control you want.

Just make sure to keep a backup of your private key and that it is encrypted with a strong password:

``` bash
$ cat .ssh/id_rsa
# Should output:
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
[...]
```

Encryption doesn't mean you have to enter the password each time you use the key. Use ssh agent to keep it in memory or add it to your system's key chain:

``` bash
# The -K switch works only on Mac OS X and will add the password to your system's key chain
$ ssh-add -K
```

Place your public key in alice's home directory on the production server (/home/alice/.ssh/authorized_keys) and deploy in her name:

``` ruby config/deploy.rb
set :user, "alice"
```

Also add the following line to authenticate with Git from your development machine, so you won't have to copy your private key on the production server:

``` ruby config/deploy.rb
ssh_options[:forward_agent] = true
```

Finally, why not disable password authentication via SSH on your server altogether? On your Linux server:
``` bash /etc/ssh/sshd_config
# Make sure the following lines exist:
ChallengeResponseAuthentication no
PasswordAuthentication no
```