---
layout: post
title:  "Metric queries in the OpenNMS datasource"
date:   2016-04-15 10:01:01
categories: [opennms, grafana]
---

The [OpenNMS Datasource for Grafana], now on version 2.0.0, adds support for metric queries that allow you to populate template variables with node references and/or resource ids. Coupling this with our support for permutations of variables make it a very powerful tool.

To help get you started, we'll walk you through setting up a dashboard that populates variables using metric queries and show you how these variables can be nested together.

Note that metric queries only work against OpenNMS Horizon 18.x or higher, so you'll need to upgrade if you haven't already. (At the time of this writing 18.0.0 isn't actually released, but [snapshots] are available.)

Let's start off by writing a query for a single metric: the available disk space on the root partition of a single node.

![rootfs on ny-cassandra-1]({{ site.url }}/assets/2016-04-15-rootfs-on-ny-cassandra-1.png)

Now let's assume we wanted to show the available disk space on the root partition of all nodes that belong to the *Newts* category. To do this, we start off by configuring a template variable called **node** that contains the list of nodes as follows:

![define the node variable]({{ site.url }}/assets/2016-04-15-define-node-variable.png)

  > Supported queries are of the form _nodeFilter($filter)_ or _nodeResources(FS:FID)_, where *$filter* is any [filter expression].

The **node** variable show now be populated with all of the nodes that belong to the *Newts* category., and we can reference it in our query:

![rootfs on Newts nodes]({{ site.url }}/assets/2016-04-15-use-node-variable.png)

  > We include the **node** variable in the label field as well since the label needs to be unique across all of the series.

We can take this one step further and define a variable that contains the list of all the available disk resources:

![define the disk variable]({{ site.url }}/assets/2016-04-15-define-disk-variable.png)

  > We can reference other variables in the query field, but if they are multi-valued, only the first value will be used.

By combining both the **node** and **disk** variables, we can now graph all of the disks on all of nodes that belong to the *Newts* category:

![all disk on Newts nodes]({{ site.url }}/assets/2016-04-15-use-disk-variable.png)

   > All of the resources must be present on all of the nodes, otherwise the query will fail.

Pretty cool.

In the graphs above we ensure the labels are unique by having them include the variable references. However, this can prevent us from be able to use those series in expressions due to special characters in the label names.

In order to work around this, we introduce a special variable named **index** that is guaranteed to be unique. By using the **index** variable in the label name we can now reference the series in one or more expressions and/or filters:

![expressions with index variable]({{ site.url }}/assets/2016-04-15-use-index-variable.png)

If you need additional help any of these topics, you can reach out to the OpenNMS community on the mailing lists, on IRC, or in [chat].

[OpenNMS Datasource for Grafana]: https://grafana.net/plugins/opennms-datasource
[snapshots]: http://yum.opennms.org/snapshot/common/opennms/
[filter expression]: https://www.opennms.org/wiki/Filters
[chat]: https://chat.opennms.com
