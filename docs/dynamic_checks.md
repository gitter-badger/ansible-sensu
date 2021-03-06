# Dynamic Check Deployment
One of the awesome features of this role (if I do say so myself) is the deployment of Sensu [checks](https://sensuapp.org/docs/0.18/checks) on a dynamic basis. Deployment of which checks should be distributed to which nodes is determined from group membership within the Ansible inventory.

Have a group of webservers you need to monitor webservices on? Well, I'm sure you've bunched them together in your inventory under the `[webservers]` group, right? Or perhaps you only want to monitor disk space on your production systems; if they're a member of `[production]` within the inventory, you can do this easily.

## How it works
Much like the dynamic datastore, I have also defined a static data store within my Ansible codebase, by setting the variable `static_data_store`.
Within this static datastore resides the following directory structure:
```
$ tree data/static
data/static
`-- sensu
    |-- checks
    |   |-- rabbitmq_servers
    |   |   `-- check_rabbitmq.sh
    |   |-- redis_servers
    |   |   `-- check_redis.sh
    |   |-- webservers
    |   |   `-- check_nginx.sh
    |   `-- zones
    |       |-- check_cpu.rb
    |       |-- check_disk.rb
    |       `-- check_mem.rb
    |-- definitions
    |   |-- check_nginx.json.j2
    |   |-- check_rabbitmq.json.j2
    |   |-- check_redis.json.j2
    |   |-- pushover.json.j2
    |   |-- pushover_handler.json.j2
    |   |-- smartos_check_cpu.json.j2
    |   |-- smartos_check_disk.json.j2
    |   `-- smartos_check_mem.json.j2
    |-- handlers
    |   `-- pushover.rb
    `-- mutators
```
As you can see, in the `sensu/checks` directory, there are the `rabbitmq_servers`, `redis_servers`, `webservers` & `zones` subdirectories.
If you've had a peruse through some of the other documentation here, you'll know that these groups are defined within my Ansible inventory:
``` ini
[rabbitmq_servers]
test.cmacr.ae

[redis_servers]
redis.cmacr.ae

[sensu_masters]
sensu.cmacr.ae

[webservers]
blog.cmacr.ae
web.cmacr.ae
sab.cmacr.ae
tater.cmacr.ae
beardy.cmacr.ae
pkgsrc.cmacr.ae

[zones]
ansible.cmacr.ae
beardy.cmacr.ae
blog.cmacr.ae
bunny.cmacr.ae
git.cmacr.ae
irc.cmacr.ae
pkgsrc.cmacr.ae
redis.cmacr.ae
sab.cmacr.ae
sensu.cmacr.ae
tater.cmacr.ae
web.cmacr.ae
test.cmacr.ae
```
Under these subdirectories, you can see [checks](https://sensuapp.org/docs/0.18/checks) that relate to directory they're placed in.
The `webservers` subdirectory includes a `check_nginx.sh` script, whilst the `rabbitmq_servers` subdirectory has one that most likely checks for RabbitMQ problems (it does... trust me).  

So how do these checks actually get deployed to their associated nodes?
With this pair of plays, in the `tasks/plugins.yml` playbook:
``` yaml
  - name: Register available checks
    local_action: command ls {{ static_data_store }}/sensu/checks
    register: sensu_available_checks
    changed_when: False

  - name: Deploy check plugins
    copy: src={{ static_data_store }}/sensu/checks/{{ item }}/
          dest={{ sensu_config_path }}/plugins/ mode=755
          owner={{ sensu_user_name }} group={{ sensu_group_name }}
    when: sensu_available_checks.stdout.find('{{ item }}') != -1
    with_flattened:
      - group_names
    notify: restart sensu-client service
```
This will [register](http://docs.ansible.com/playbooks_conditionals.html#register-variables) a list of available checks, then deploy them to their intended groups based on node membership, as set within the Ansible inventory.

And, because nodes can of course be members of more than just one group, checks will be deployed in full to nodes that belong to several groups!

## Defining your static data store
Just like defining the dynamic data store, this needs to be done somewhere it can be made available to all the nodes you'll be applying this role to. I do this in the same place I define `dynamic_data_store`: `group_vars/all.yml`, like so:
``` yaml
static_data_store: path/to/ansible/codebase/data/static
```
Once you've defined where you'd like your static data store to be, you can drop in some checks by creating a similar data structure to mine:
```
data/static
`-- sensu
    `-- checks
        |-- rabbitmq_servers
        |   `-- check_rabbitmq.sh
        |-- redis_servers
        |   `-- check_redis.sh
        |-- webservers
        |   `-- check_nginx.sh
        `-- zones
            |-- check_cpu.rb
            |-- check_disk.rb
            `-- check_mem.rb
```

## Picking up changes on the fly
Let's say, for some reason, one of your nodes decides to switch roles, or take on the responsibility of another role.
You've got a webserver, who all of a sudden you decide "this guy should be running Redis too". No problem!
You probably had something like:
```
[webservers]
mime.domain.name
```
In your Ansible inventory, after this spontaneous decision to have your webserver act as a key-value store, you may change it to the following:
```
[webservers]
mime.domain.name

[redis_servers]
mime.domain.name
```
Not to worry, the next time your playbook applying this Sensu role runs through (notably the `tasks/client.yml` & `tasks/plugins.yml` playbooks), the new checks for Redis will be deployed to `mime.domain.name` and it'll be subscribed to the `redis_servers` stream within Sensu. Pretty slick, right?

The same goes for the removal of a node from a group. Did you just realize you really don't want `mime.domain.name` to act as a Redis server?
It's cool, we all make mistakes, just take him out of the `[redis_servers]` group in your inventory. When your play comes through again, applying this Sensu role, he'll be unsubscribed from the `redis_servers` stream, and Redis'll stop being monitored!
