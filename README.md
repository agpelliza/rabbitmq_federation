Simple Federated RabbitMQ
=========================

This is the setup for a three node federated RabbitMQ installation using Vagrant.


Prerequisites
-------------

The following software needs to be installed:

* [Virtualbox](http://virtualbox.org)
* [Vagrant](http://vagrantup.com)

Download the precise32 box with

    vagrant init hashicorp/precise32

Installation
------------

The Vagrantfile starts 4 virtual machines, 3 RabbitMQ nodes and one worker. You can change the number of nodes
by adapting the `Vagrantfile`.

Run

    vagrant up

to install and start the 4 VMs.

You can ssh to the individual VMs like this:

    vagrant ssh rabbit1
    vagrant ssh rabbit2
    vagrant ssh rabbit3
    vagrant ssh worker

The RabbitMQ Webconsoles are available (via port forwarding) on http://localhost:10010 (and 10011, 10012)


Setup on RabbitMQ
-----------------

In order to setup the federation between the different RabbitMQ servers, you will need to a bit of command
line work on each of the servers. The federation plugin is already installed by the `rabbitmq.sh` shell script
that gets executed when the machines are installed.

For example, on rabbit1, federate to 2 and 3:

    sudo rabbitmqctl set_parameter federation-upstream rabbit2 '{"uri":"amqp://192.168.40.11"}'
    sudo rabbitmqctl set_parameter federation-upstream rabbit3 '{"uri":"amqp://192.168.40.12"}'
    sudo rabbitmqctl set_parameter federation-upstream-set test '[{"upstream":"rabbit2"},{"upstream":"rabbit3"}]'
    sudo rabbitmqctl set_parameter federation local-nodename '"rabbit1"'
    sudo rabbitmqctl set_policy federate-me "^amq\." '{"federation-upstream-set":"test"}'

This does the following:

* Define two upstream nodes (named rabbit2 and rabbit3) and assign them the correct addresses. Then build an upstream-set
  called 'test' that contains those two nodes
* Set the nodename of the local node
* Create a policy (called 'federate-me') that selects all queues whose name start with amq and federate them to the
  upstream-set 'test'

You will need to run a similar set of commands on each node to connect them as well.


Run your federated setup
------------------------

Connect to your worker (multiple times), change the dirctory to `/srv` run the following commands separately
in each shell:


    ruby receiver.rb 192.168.40.10 '#'
    ruby receiver.rb 192.168.40.11 'article.foo'
    ruby receiver.rb 192.168.40.12 '*.bar'

The first parameter is the IP address of the RabbitMQ node that this receiver should connect to, the second parameter
is the topic that this receiver subscribes to.

In a fourth shell you can then send out messages to one of the brokers:

    ruby sender.rb 192.168.40.10 article.foo

You will see that the receivers on the different nodes will get the federated message depending on if they
subscribe to that topic or not.
