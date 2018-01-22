---
layout: post
title:  "this old ceilometer"
date:   2017-12-03 08:00:00 -0500
tags: ceilometer gnocchi mitaka
---

Gnocchi is a great solution for large scale time series storage and in
combination with Ceilometer, it provides a solid solution for tracking
resources across your OpenStack environment that can be extended to solve your
use cases such as billing or monitoring.

The above statement is correct for Gnocchi4. It’s even more accurate when
speaking about Gnocchi4.1 which provides even better performance, more
flexible query syntax, and less resource usage. That said, the prior versions
of Gnocchi, like all things, are not as great. Gnocchi3.x is arguably only a
good solution unless you acquire an in-depth knowledge of the code and
Gnocchi2.x is arguably unusable for any medium or large cloud deployment.

So the obvious solution here is to just upgrade to Gnocchi4 and your monitoring
solutions are resolved!… except depending on the packages you’re using, the
available version of Gnocchi might be something unusable such as Gnocchi2.x.

As I currently have a Mitaka deployment running, let’s hack that to support
Gnocchi4.1

## What is known

* Ceilometer Newton is compatible with Gnocchi3 (maybe more)
* Ceilometer Ocata is compatible with Gnocchi3.1+ (to at least 4.1.1)
* Ceilometer Pike is compatible with Gnocchi4.0+ (and gated against master)
* Ceilometer master (Queens) is compatible with Gnocchi4.0+ (and gated against
master)
* Gnocchi2.x previously created Ceilometer resource types but that
functionality is now contained in Ceilometer.
* The errors gnocchiclient throws are different now from what they did in
Mitaka so we’ll need a dispatcher that can handle this.

As I know Ocata works with Gnocchi4.1.1 and I’m lazy, I’m just going to use
Ocata code.

## Installing Gnocchi

To install, I’m running through the Ocata install guide with the only
difference being, rather than installing the Ocata packages, I’m
installing Gnocchi and the client via pip:

{% highlight bash %}
pip install gnocchi[mysql,redis,keystone]
pip install “gnocchiclient<4.0.0”  # my fix doesn't support >=4.0
{% endhighlight %}

for reference, below is my gnocchi.conf

>    [api]
>   auth_mode = keystone
>
>   [indexer]
>   url = mysql+pymysql://gnocchi:password@controller/gnocchi
>
>   [keystone_authtoken]
>   # i copied this section from my ocata ceilometer.conf that packstack
>   # configured
>   auth_uri=http://controller:5000/v2.0
>   identity_uri=http://controller:35357
>   admin_user=gnocchi
>   admin_tenant_name=services
>   admin_password=password
>
>   [metricd]
>   workers = 18
>
>   [storage]
>   redis_url = redis://controller:6379
>   driver = redis
>   coordination_url = redis://controller:6379
>   aggregation_workers_number = 8
>   metric_processing_delay = 60
>   metric_reporting_delay = 10
>
>   [incoming]
>   driver = redis
>   redis_url = redis://controller:6379

with all that set, I initialise the Gnocchi service using:

{% highlight bash %}
gnocchi-upgrade
{% endhighlight %}

## Creating Ceilometer resources

Now that I have Gnocchi4.1 installed, I’ve decided to backport Ceilometer’s
Gnocchi integration from Ocata.

I first began by copying the gnocchi_client code from Ceilometer as it is not
in Mitaka. The big difference between Ocata and Mitaka is that in Mitaka,
Ceilometer uses a global configuration object and also does not use
keystoneauth1. To get around this, I edited the get_gnocchiclient method:

{% highlight python %}
def get_gnocchiclient(conf, endpoint_override=None):
    requests_session = requests.session()
    for scheme in list(requests_session.adapters.keys()):
        requests_session.mount(scheme, ka_session.TCPKeepAliveAdapter(
            pool_block=True))

session = keystone_client.get_session(requests_session=requests_session)
    return client.Client('1', session,
                         interface=conf.service_credentials.interface,
                         region_name=conf.service_credentials.region_name)
{% endhighlight %}

Once this is created, we can make Ceilometer create required resource types in
Gnocchi. This can be skipped if you already have an existing Gnocchi deployment
and just upgraded it to Gnocchi4.1.1 as Gnocchi previously created these
resources-types

{% highlight python %}
from oslo_config import cfg

from ceilometer import gnocchi_client
from ceilometer import service

service.prepare_service()
gnocchi_client.upgrade_resource_types(cfg.CONF)
{% endhighlight %}

The above script should trigger POST requests against the gnocchi-api, creating
OpenStack specific resource-types.

## Using Ocata Gnocchi dispatcher

First, I began by creating gnocchi_resources.yaml by copying the Ocata file
verbatim.

Next, I overwrote the existing Gnocchi dispatcher with the Ocata version.
Doing so provides support for capturing event data to track a resource’s
lifespan. Quite a few tweaks were required in this module ranging from changing
the dispatcher to handle a global configuration object, to fixing some import
statements to handle different dependencies. The end result can be found in my
fork.

With that, all the changes to required to leverage Gnocchi4.1 in Mitaka is
complete. What remains is to edit ceilometer.conf and add:

>    [DEFAULT]
>    meter_dispatchers = gnocchi
>    event_dispatchers = gnocchi
>    # meter|event_dispatchers = database can still remain if you wish to
>    # continue publishing to ceilometer storage as well as Gnocchi.
>
>    [cache]
>    # this is used to minimise requests made to Gnocchi
>    backend_argument = redis_expiration_time:600
>    backend_argument = db:0
>    backend_argument = distrubuted_lock:True
>    backend_argument = url:redis://controller:6379
>    backend = dogpile.cache.redis

Once this is set, restarting openstack-ceilometer-collector service should
start publishing to Gnocchi4, giving you a scalable metric storage solution in
Mitaka.
