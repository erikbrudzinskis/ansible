# Ansible

## Host listing
To list all hosts contained in your current inventory use:
`ansible --list-hosts all`

To specify which inventory you want to use for an ansible command use:
`ansible -i {myInventoryFileName} --list-hosts all`

You can also use wildcards when listing hosts.
This command is equivalent to specifying `all`:
`ansible --list-hosts "*"`

To get hosts from specific group write group name at the end:
`ansible --list-hosts loadbalancer`

To get individual host write host name at the end:
`ansible --list-hosts db01`

You can also use wildcards within a name:
`ansible --list-hosts "app0*"`

You can have multiple search parameters divided by comma:
`ansible --list-hosts "db0*",control`

To get a particular host from a group you can specify the number of the host in square brackets:
`ansible --list-hosts webserver[0]`

To get all hosts not belonging to a particular group use exclamation mark:
`ansible --list-hosts \!control`

## Tasks and commands
You can specify modules, arguments and target patterns to use.

To specify which module to use write `-m`:
`ansible -m ping all`

To specify which argument to use write `-a`:
`ansible -m command -a "hostname" all`
`command` is the default option, so if you don't specify a module, then command module will be used.
With `command` you can run any command you want, and it will return whether the command was successful or not
as well as any STDOUT output the command gave:
`ansible -a "/bin/false" all` - this uses built-in Linux command that simply returns false and in our case will return FAILED on all hosts with no output

## Playbooks and plays
Playbook is a yaml file, which is made out of plays. A play is a set of target hosts and the tasks to execute against these target hosts.

You can see example of using `command` with `hostname` argument in ./playbooks/hostname.yml file.
To execute this playbook use:
`ansible-playbook playbooks/hostname.yml`
To see which kind of modules can be used in playbooks visit Ansible Docs.

### apt
You can use `apt` module to install packages on Ubuntu. It accepts such parameters as:
- name: name of the package
- state: present, absent, latest, etc. is used to determine what to do with the package,
i.e. only install it if it is not installed, delete it, always update to latest, etc.
- update_cache: equivalent of running `apt-get update` before installation

### become
For some commands you might need root access. To do that you need to state `become: true` in your yaml file.
You might need to specify `--ask-become-pass` depending on your setup.

### with_items
Sometimes you might need to do one and the same thing several times, but with different arguments.
For example, if you want to install several packages in a row. In that case we can use `with_items`
array and list all the items we would like to install. Each item will be executed and `name={{item}}`
will be replaced with the name of your item. 

### service
If you need to do something with your services, then use `service` module. It accepts several parameters:
- name: name of the service
- state: started, restarted, stopped, reload, which allow to start the service if it is not started,
restart the service, stop it or reload it.
- enabled: true or false representing whether any action should be taken with the state. For example,
if the state is 'started', then if 'enabled' is true, then the service will be started.

### apache2_modules
If you need to modify, start, stop or restart apache modules, then you can use `apache2_modules` module.

### handlers
`handlers` can be added on the same level as `tasks` in your yaml files. `handlers` work and look just like
the `tasks`, but they will not be executed unless one of the `tasks` notifies a handler:
`notify: {nameOfYourHandler}`

`handlers` are only executed as part of a `task`, if there was a change during that task. For example,
if you are installing a package with `state=present` and the task has `notify` to reference a `handler`,
then `handler` will only be called, if package was installed. If package was already present, then
`handler` is not called.

`handler` is only executed once at the end of a play, even if it is notified several times.

### Other modules
- `pip` - to manage python dependencies
- `copy` = copy files
- `file` - manage files
- `template` - same as copy but uses your variables
- `lineinfile` - replace or modify a particular line in a file
- `mysql_db` - manage MySQL databases
- `mysql_user` - manage MySQL users
- `wait_for` - wait for a particular port to start, stop listening or drain all connection
- `uri` - can be used to test your endpoints together with `register`, `fail` and `when`

## Roles
To encapsulate our Ansible code we can use roles. To initialize a role we can use `ansible-galaxy`.
For example, to initialize `control` role:
`ansible-galaxy init control`
This will create the required folder structure and associated files in order to start using roles.

This allows us to have a clear folder structure, in which we define all the tasks, handlers, etc.
To use a role in a playbook simply specify the role in `roles` array.

It is also possible to split your tasks into several roles to make the roles reusable. For example,
you could split apache2 server installation (reusable component) and your web-site installation (
project specific component), which allows apache2 to be reused.

## site.yml
The most common way to combine all the playbooks necessary for your deployment is `site.yml` file.
You can include all the playbooks you want using `ansible.builtin.import_playbook` keyword.

## Variables
Ansible has different types of variables used in different scenarios. Below are the list of commonly
used variable types.

### Facts
To get the facts about a host you can use `setup` method, i.e.:
`ansible -m setup server01`
These facts can then be used in your playbooks, for example to get the IP address of a host:
`{{ ansible_default_ipv4.address }}`

### Defaults
If you want, you can provide some default values inside of defaults/main.yml. This allows you to use these
values in your playbooks just like you can use facts, for example:
`{{ myCustomValue }}`

### vars
Defaults can be overridden with your own variables. Your own variables can be put inside vars/main.yml file
inside a role, directly into your tasks or in your play using `vars` key as well as inside your
`ansible.builtin.import_playbook` statements and inside roles definition in your playbooks.

Note that all variables you define are global in scope no matter where you define them.

### with_dict
`with_dict` is practically the same thing as `with_items`, but it allows you to pass key-value pairs instead.
After that you can use them in your tasks and templates via `{{ item.key }}` and `{{ item.value.myCustomValueKey }}`

### group_vars
You can also create a folder in the parent directory called `group_vars`, which contains all the variables,
which can be referenced from any part of your Ansible project.

### vault
If you want to store passwords or anything that shouldn't be clear text in your Ansible project, you can
use `vault`. It allows you to encrypt a file with you data with a passphrase, so you can commit it to a repo.
You can then run Ansible specifying your passphrase, which will let you use your encrypted variables.

To create a `vault` file use:
`ansible-vault create vault`
To edit the created file use:
`ansible-vault edit vault`

The values in your vault file can then be used across your playbooks and var files, if you provide access
to them (for example, by placing the `vault` file inside group_vars folder).

Note that you either need to add `--ask-vault-pass` to your ansible-playbook command,
or you can create a txt file (don't forget to `chmod 0600 !$` to restrict access to the file)
with your password (i.e. ~/.vault_pass.txt) and then specify the file in your ansible.cfg:
`vault_password_file: ~/.vault_pass.txt`

## Galaxy
Online repository for roles is available at `galaxy.ansible.com`, which you can download and use inside your
projects. To install a role from `galaxy` locally use:
`ansible-galaxy install username.rolename`

## Performance
To check the performance of your playbook you can use `time ansible-playbook myPlaybook.yml`. There are number
of ways to increase this performance, most popular of which are listed below.

### gather_facts
Ansible gathers all the facts about the hosts as first step of every playbook. You can switch off fact gathering,
if it is not necessary in your case with `gather-facts: false`. It might give you a significant performance boost,
especially if the tasks themselves don't take much time.

### cache_valid_time
Some commands like `apt` has an option to do `update_cache: yes` (equivalent to `apt-get update`). When you specify it
(which you should in most cases), cache is being updated every single time the task is run. It is not ideal, because
if you ran the same task five minutes ago, then you can be pretty much sure that the cache is already updated. To avoid
that we can set `cache_valid_time=86400` (24 hours as an example).

Also it might be useful to extract cache updating to a separate step and run it on all hosts simultaneously before any
other tasks that include `apt` module. This way you update the cache only once in the beginning instead of updating it
every time the `apt` module runs.

### limit
Sometimes you might not want to run tasks for all the hosts inside a playbook, especially if you have a big playbook
with a lot of hosts and a lot of tasks. To limit the execution of the playbook to a particular host you can use
`ansible-playbook myPlaybook.yml --limit server01`. This will limit the execution to parts only applicable to server01.

### tags
It is also possible to split your tasks and plays by `tags` by adding keyword `tags` to your task declaration and specifying your
own tag names. You can then run only the tasks associated with a certain tags:
`ansible-playbook myPlaybook.yml --tags "myCustomTag"`

You can also check which tags are used in a playbook by running:
`ansible-playbook myPlaybook.yml --list-tags`

It is also possible to skip tasks associated with a certain tag by running:
`ansible-playbook myPlaybook.yml --skip-tags "myCustomTag"`

### Pipelining
It is also possible to setup SSH pipelining to speed up the process by specifying this in your ansible.cfg:
`[ssh_connection]`
`pipelining = True`

Usually, it is not required though, but you can read more in the Ansible docs.

## Idempotence
Some commands you execute in Ansible will be reported as 'changed' on play recap, however in reality nothing was really changed.
As an example we can take `command: service nginx status` task. It doesn't change anything, but always reports as changed.

To avoid that you can use `changed_when`. For example, specifying `change_when: false` means that this task will never be
reported as changed, only success or fail. Alternatively, you can use any variables you are registered in boolean expression:
`changed_when: "active.stdoutlines != sites.key()"`

The same logic can also be applied with `failed_when`.

## Troubleshooting
There are different approaches to troubleshooting in Ansible. Some of them are described here.

### ignore_errors
You can add `ignore_errors: true` to your task to allow Ansible to continue playbook execution, even if a certain task has
failed. This is useful as a temporary configuration in case you know that certain step is going to fail this time, but you
still want to run the playbook without SSHing into the host and doing things manually.

### Certain task execution
There are several ways on how you can avoid executing the whole playbook and only execute a certain task you are currently
working on:
`--step` - will ask you whether a task needs to be executed before each task in a playbook.
`--start-at-task` - allows you to start playbook execution at certain task instead of executing it from the beginning

### Retry
If some of the hosts are unreachable, then Ansible shows us a command at play recap:
`to retry, use: --limit @/home/ansible/site.retry`
This file contains all the hosts that were not reachable during the last execution, so if you run the playbook again
with `--limit`, then only (previously) unreachable hosts will be affected.

### Dry runs
You can add `--syntax-check` to your playbook execution and it will not execute the playbook, but instead it will only
check that the syntax inside this playbook is correct. Note: You still need to provide a valid inventory in order to
execute the syntax check.

To initiate an actual dry run you can use `--check`, which will allow Ansible to predict how a particular task is going to
behave. Note: `--check` is only supported by some Ansible modules.

### Debugging
You can use `debug` task to get some information into your terminal. For example, you can add this between your tasks:
`debug: var=activeSites.stdout_lines`
This will show your variable in the console and you can compare it to an expected value.

You can also use `var=vars` to get all currently available variables.