---
layout: post
title:  "OpenNMS meets Grafana v1.1.0"
date:   2015-10-31 17:19:34
categories: [opennms, grafana]
---

A new release of the [OpenNMS Grafana Datasource], is available in OpenNMS' stable repositories.

There are a few noteworthy changes in this release. For starters, we're now compatible with Grafana v2.5.0 (there were some backwards incompatible changes that were made to their [Plugin API]).

We've also reworked the layout for query editor - the previous implementation suffered from some visual issues on narrow displays. Inspired by the InfluxDB query editor, the new layout looks like:

![OpenNMS Query Editor]({{ site.baseurl }}/assets/grafana-opennms-query-editor-v1.1.0.png)


At the bottom of the editor, you'll also find helpful notes related to using the datasource.

The update also includes support for a new _Filter_ query type, for which support was added in OpenNMS 17.0.0. Filters are used to enhance, or alter the results returned by attribute, expression or previous filters queries. Some of the existing filters can be used to remove outliers, interpolate missing values, calculate trends and make forecasts. Additional filters can also be developed by implementating a simple interface.

Here we show an example of the _Trend_ filter applied to the number of JVM threads:

![Number of threads with trend line]({{ site.baseurl }}/assets/grafana-opennms-threads-with-trend-filter.png)

Another noteworthy feature is static template support. If you have multiple systems, or interfaces for which you want to see the charts, you can reference these using variables.

For example, in the dashboard's _Templating_ section I've added a variable called **nodes** that has 3 values: ny-cassandra-1, ny-cassandra-2 and ny-cassandra-3:

![Defining a template variable]({{ site.baseurl }}/assets/grafana-opennms-defining-template-variable.png)

In my query definition, I can now reference the **$nodes** variable in any attribute or expression query and have the query repeated for all of the selected nodes:

![Using a template variable]({{ site.baseurl }}/assets/grafana-opennms-using-template-variable.png)

Note that the variable name is also used in the query label, since the label must be unique.

Moving forward, we're interesting in adding support for metric queries, where values for template variables could be the result of queries like: all nodes in categories 'Web Server' and 'Production'.

[OpenNMS Grafana Datasource]: http://www.opennms.org/wiki/Grafana
[Plugin API]: https://github.com/grafana/grafana/blob/master/docs/sources/datasources/plugin_api.md
