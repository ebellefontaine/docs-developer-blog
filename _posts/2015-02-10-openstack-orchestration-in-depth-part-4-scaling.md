---
layout: post
title: 'OpenStack Orchestration In Depth, Part IV: Scaling'
date: '2015-02-10 07:15'
comments: true
author: Miguel Grinberg
published: true
categories:
  - Private Cloud
  - Orchestration
bio: "Miguel Grinberg is a software engineer with a background in web technologies and REST APIs. He is the author of the book \"Flask Web Development\" from O'Reilly Media, and has a blog at http://blog.miguelgrinberg.com, where he writes about a variety of topics including web development, robotics, photography and the occasional movie review. Miguel works as a Software Developer with the Rackspace Private Cloud team. He lives in Portland, Oregon with his wife, four kids, two dogs and a cat. Follow @miguelgrinberg on Twitter."
---

This is the fourth and last article in my series on OpenStack orchestration with Heat. In the previous articles, I gave you a gentle [introduction to Heat](/blog/openstack-orchestration-in-depth-part-1-introduction-to-heat), and then I showed you some techniques to orchestrate the deployment of [single](/blog/openstack-orchestration-in-depth-part-2-single-instance-deployments/) and [multiple](/blog/openstack-orchestration-in-depth-part-3-multi-instance-deployments/) instance applications on the cloud, all done with generic and reusable components.

Today I'm going to discuss how Heat can help with one of the most important topics in cloud computing: scalability. Like in my previous articles, I'm going to give you actual examples that you can play with on your OpenStack cloud, so make sure you have an environment where you can run tests, whether it's a [Rackspace Private Cloud](http://www.rackspace.com/cloud/private/), [DevStack](http://devstack.org/) or any other OpenStack distribution that includes Heat.

<!-- more -->

## Vertical and Horizontal Scaling

Let's say you have deployed an application to the cloud, and after some time, the application acquired a reasonable user base, that is also constantly growing. You realize it is starting to feel sluggish during peak hours. What can you do to improve response times for your users?

The obvious solution is to use a more powerful instance. You can resize the compute instance, where your application runs, to a larger flavor, so that it gets more CPUs, more RAM or more disk space (whatever is necessary to bring response times back to a reasonable level). This is called *vertical scaling*. This type of scaling works well up to a point. However, you will eventually reach the maximum supported number of CPUs, RAM or disk that an instance can have, and once you do, you have no way to scale more. If the load generated by your users continues to grow beyond that, then you will need to find a different way to scale.

The other approach to scalability is to have your application running on multiple servers. If you figure out a way to run your application on two or more instances, and then add a load balancer that distributes client requests evenly among them, then your application will be able to handle much larger volumes of clients. This is called *horizontal scaling*. With this type of scaling, it is going to take longer to reach a limit, since you can continue to add servers behind the load balancer until you run out of compute resources in your cloud. But even then, you can always add more hardware to your cloud and push the limit further.

The nice thing about vertical scaling is that all applications can benefit from it. It is essentially a computer upgrade, so things work the same, but faster. Horizontal scaling, on the other side, requires applications to be carefully designed. For example, consider applications that write data to disk files. When one instance updates some file on disk, all the other instances become outdated. For things to work, data files that change may need to be replicated to all the instances, or else the load balancer could be configured to send a given client always to the same instance, so that the client specific files are always accessible. While Horizontal scaling gives you more freedom to grow, it is not a practical solution for all types of applications.

The main purpose of this article is to show you some of the tricks Heat offers to scale applications horizontally, the method of scaling that presents the most challenges. The application that I'm going to deploy and scale is a very simple Flask web server that shows the hostname of the server that handled the request when you connect to it. For those familiar with Flask, below is the entire source code of this application:

    import socket
    from flask import Flask

    hostname = socket.gethostname()
    app = Flask(__name__)

    @app.route('/')
    def index():
        return 'Hello from {0}!'.format(hostname)

    if __name__ == '__main__':
        app.run(debug=True)

The reason I chose to use such a simple application is that, with this application, it is easy to test that the load balancer is working. Connecting to the application through the load balancer and refreshing the page should show a changing hostname, because each time the page will be handled by a different server.

Continuing with the component approach I introduced in the previous article, I created a template that deploys this application, which I called "tiny", using gunicorn and nginx. I'm not going to show you the complete template here, since it uses all the same techniques I showed you in Parts 2 and 3 of this series. If you are interested in studying this template in detail, you can find it in file [lib/tiny.yaml](https://raw.githubusercontent.com/miguelgrinberg/heat-tutorial/master/lib/tiny.yaml) in the Github repository.

The first example I'm going to show you is the deployment of this little server as a single instance application, so that you can become familiar with it and ensure that you can get it to work on your cloud in its simplest form. After you clone the [github project](https://github.com/miguelgrinberg/heat-tutorial) to your local disk, you can launch this application as follows:

    (venv) $ heat stack-create tiny-single -f heat_4a.yaml -e lib/env.yaml

As the examples in previous articles, this template has parameters that allow you to configure what image, flavor, key pair and public network to use, so add those with `-P` if you need to. Once the application is deployed, you can check the outputs to find the public IP address that was assigned to it. If you connect to this address with your web browser, you will receive a page with a content similar to this:

    Hello from ti-y-instance-cpklvzeqwbwy-tiny-instance-mitkww7zbr52!

If you get a message like the one above, then the application is working. The long and cryptic symbol is the hostname that was assigned to the compute instance that runs the application. Later when you deploy a horizontally scaled version of this application, you will see that as you refresh the page different hostnames appear, confirming that requests are being handled by multiple servers.

## Horizontal Scaling with Resource Groups

The most basic approach to horizontal scaling with Heat is based on a resource type called `OS::Heat::ResourceGroup`. This resource wraps a standard resource definition and creates multiple identical copies of it. For example, to horizontally scale the little Flask server I shown above, you could use the following definitions:

    parameters:
      # ...

      cluster_size:
        type: number
        label: Cluster size
        description: Number of instances in cluster.
        default: 2

    resources:
      tiny_cluster:
        type: OS::Heat::ResourceGroup
        properties:
          count: { get_param: cluster_size }
          resource_def:
            type: Lib::MSG::Tiny
            properties:
              image: { get_param: image }
              flavor: { get_param: flavor }
              key: { get_param: key }
              private_network: { get_param: private_network }

As you can see, the `count` property defines how many copies of the application to launch. In this case, I'm setting this value from a parameter `cluster_size`, which the user can set. The `resource_def` property is where the resource that is getting scaled is configured. Note that I'm invoking the same nested template I used in the first example. Simply by wrapping the resource with `OS::Heat::ResourceGroup`, I made it into a horizontally scaled resource!

## Load Balancing a Horizontally Scaled Application

Of course having a resource group is only a part of the solution. Once you have your application running on multiple instances, you need to have a load balancer that presents itself as the entry point to clients. The load balancer will accept all requests and internally dispatch them to the actual servers.

In OpenStack, there are a couple of options to work with load balancer. Neutron has an optional service called *load balancing as a server* (or LBAAS), which Heat fully supports. To use it, you can instantiate resources of type [OS::Neutron::LoadBalancer](http://docs.openstack.org/developer/heat/template_guide/openstack.html#OS::Neutron::LoadBalancer), [OS::Neutron::Pool](http://docs.openstack.org/developer/heat/template_guide/openstack.html#OS::Neutron::Pool) and [OS::Neutron::HealthMonitor](http://docs.openstack.org/developer/heat/template_guide/openstack.html#OS::Neutron::HealthMonitor).

However, I'm using a [Rackspace Private Cloud](http://www.rackspace.com/cloud/private/), which does not currently include the LBAAS service, so I had to come up with a different solution. What I did is create yet another nested template and configure it to run [HAProxy](http://www.haproxy.org/), the open-source load balancer. In the end, this is actually more powerful than the Neutron based solution, as it gives me full control of the load balancer.

You can find the HAProxy template in file [lib/haproxy.yaml](https://raw.githubusercontent.com/miguelgrinberg/heat-tutorial/master/lib/haproxy.yaml) in the Github repository. Once again, I'm using the same techniques I used in previous templates, so you will find this new template easy to understand. At the core, there is a server resource with a bash script in the `userdata` property which performs he installation and configuration of the HAProxy service.

The tricky aspect of the HAProxy server is that it needs to know which servers are available to accept client requests. I'm passing the list of IP addresses for the servers to the instance as *metadata*:

    resources:
      # ...

      haproxy_instance:
        type: OS::Nova::Server
        properties:
          # ...
          metadata:
            servers: { get_param: servers }

An instance can access its metadata by contacting Nova's metadata service, which, from inside the instance, is always available at address *http://169.254.169.254*. The list of key/value pairs of metadata is available as a JSON payload at URL *http://169.254.169.254/openstack/latest/meta_data.json*. The bash script in the HAProxy instance not only installs HAProxy, it also creates a little Python script called `/etc/haproxy/update.py`. The script parses the JSON metadata and looks for the key `servers`, which it then writes to a HAProxy configuration file as the list of IP addresses. Below you can see the command that gets all this done:

    curl -s http://169.254.169.254/openstack/latest/meta_data.json | python /etc/haproxy/update.py

Here I'm feeding the JSON, which I obtain with `curl`, as standard input to the Python script. You can see the gory details of the script in the [template source code](https://raw.githubusercontent.com/miguelgrinberg/heat-tutorial/master/lib/haproxy.yaml). If you look for this command in it, you will see that I haven't included it on its own. Instead, I'm registering it as a cron job that runs every minute, to ensure that any changes to the server pool cause an update to the HAProxy configuration.

So now, thanks to the fantastic support for nested templates, I can add a load balancer with the following resource declaration:

    resources:
      # ...

      load_balancer:
        type: Lib::MSG::HAProxy
        properties:
          image: { get_param: image }
          flavor: { get_param: flavor }
          key: { get_param: key }
          private_network: { get_attr: [network, name] }
          servers: { get_attr: [tiny_cluster, ip] }

Now you should be ready to take a look at the second example, called [heat_4b.yaml](https://raw.githubusercontent.com/miguelgrinberg/heat-tutorial/master/heat_4b.yaml). This example deploys a fully working horizontally scaled application. You will be suprised at how simple it is, as there are only four top-level resources in it:

- a private network, for all the instances launched. This resource is the same as in *heat_4a.yaml*.
- the cluster of "tiny" servers, shown above.
- the load balancer, also shown above. This is based on the [lib/haproxy.yaml]() nested template, which takes the list of servers as a property. To generate this list I use the `ip` property of the `OS::Heat::ResourceGroup` attribute.
- a floating IP address, which is attached to the load balancer.

Isn't this amazing? I'm not going to kid you and claim that this is a simple solution, but I really like that the complexity is all hidden away in the reusable nested templates, so that they don't need to clutter the top-level templates. Go ahead and give this example a try:

    (venv) $ heat stack-create tiny-horizontal -f heat_4b.yaml -e lib/env.yaml -P cluster_size=3

Here I'm launching the template with three servers. After the stack is created, you can connect to the assigned floating IP address shown in the outputs. Once the stack is deployed, you can refresh the page multiple times to see how the responses come randomly from three different hostnames.

## Updating a Heat Stack

So far, I have shown you how to create a horizontally scaled solution, but one very important aspect of scaling is to do so dynamically. Your application may get busier at certain times of the day, or on certain days, so you may want to add more resources at those times, without having to take the stack offline and make a new one.

To make changes to a deployed stack, the Heat command line client includes a `stack-update` command. For example, I can easily increase the number of servers in the example above from three to five with the following command:

    (venv) $ heat stack-update tiny -f heat_4b.yaml -e lib/env.yaml -P "cluster_size=5"

When Heat receives this command, it will launch two new instances, which will be added to the three already in the server pool. Reducing the server count works in the same way. In that case, Heat will just randomly pick some instances to kill. Once the cron job I installed in the load balancer detects that the server list has changed, it will update the HAProxy configuration file and restart the service.

Remember vertical scaling? The `stack-update` command of Heat can also be used for that, simply by setting a new flavor:

    (venv) $ heat stack-update tiny -f heat_4b.yaml -e lib/env.yaml -P "cluster_size=5;flavor=m1.large"

With the above command, Heat will detect that the flavor went from the default of `m1.small` to `m1.large`, so it will resize all the instances.

## Auto-Scaling

The last `stack-update` example allows an operator to manually scale the server pool up or down, simply by invoking the `stack-update` function provided by Heat. But wouldn't it be great if scaling the pool up and down happened automatically, based on demand or server load?

Heat has yet another resource type called `AutoScalingGroup`, which in many ways is similar to `ResourceGroup`. The difference is that the `AutoScalingGroup` resource can be configured to apply scaling changes on certain events, triggered by Ceilometer or by a custom monitoring agent. With this resource, you can create scaling policy rules that alter the size of the server pool based on CPU usage or any other variable that can be monitored.

Unfortunately my experience with auto-scaling groups in Heat with Icehouse and Juno has not been great. As I mentioned several times, I'm mostly working with a Rackspace Private Cloud, which like many other distributions does not ship Ceilometer yet, so, to make use of auto-scaling, it is necessary to implement a custom monitoring agent. While I was working on that, I found a bug I could not workaround that occurs when combining auto-scaling resources with nested templates. I submitted a fix, and it has been merged towards the Kilo release,. Therefore, the state of auto-scaling with Heat is going to improve in the near future, but I feel it isn't ready for production use yet.

So how can one auto-scale Heat stacks? I can think of two viable options at this time:

- Create cron jobs that send `stack-update` calls on a schedule (assuming it is possible to predict server loads).
- Install a custom monitoring script on an instance dedicated to that purpose. This instance can receive the list of servers as a property (like the `lib/haproxy.yaml` nested template does) and periodically SSH into the servers to take a look at the load or any other variable. This instance can then make a scaling up or down decision based on the readings obtained from the servers and send a `stack-update` call into Heat to make an adjustment.

If I needed to implement auto-scaling right now, I would probably go with the second option myself.

## Horizontal Scaling and Databases

I thought I would make a brief mention regarding databases and horizontal scaling, because applications that work with data are special, in that they are much harder to scale. Consider a MySQL server like the one you can start using my `lib/mysql.yaml` nested template. The instance that runs the MySQL server has its own storage, be it internal or external.

If you deploy several copies of the MySQL nested template, then you end up with several independent MySQL servers, each with its own separate storage. Pointing all the instances to the same storage does not really help much, because then the storage remains a bottleneck that puts a limit to how far out you can scale.

For the database scaling to work, you need all these independent storages to be synchronized, so that when a write operation happens on one of them, the operation is replicated on all the others. Database replication is a tough problem to solve, and not all database servers have a solution for it. For MySQL specifically, there are a few options. In my experience, the [Galera Cluster](http://galeracluster.com/products/) is by far the best option. So, to have a horizontally scaled MySQL Server, you would create another nested template that deploys Galera instead of plain MySQL. This nested template would export the same outputs as `lib/mysql.yaml`, so it can take its place on any templates that were made to work with the single-server database.

It is important to note that the issues with horizontal scaling are not exclusive of databases. In reality, any process that writes data to disk, even regular files, is potentially harder to scale due to the need to replicate writes to all the servers in the pool.

## Conclusion

I hope you have found this article, and the previous ones, useful in learning how to keep the complexity of your Heat templates under control. I have covered the topics I intended to discuss in the series, so I am going to end it here, at least until Kilo is out, at which point I will probably do an update on the state of auto-scaling.

For now, I wish you the best luck in your adventures with Heat. If you have any questions, or have suggestions to improve my examples, feel free to contact me by writing an issue on the [Github repository](https://github.com/miguelgrinberg/heat-tutorial).

Miguel
