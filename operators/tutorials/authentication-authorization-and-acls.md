---
description: Set up HTTP basic auth and ACLs for access to controller and broker
---

# Authentication, Authorization, and ACLs

Apache Pinot 0.8.0+ comes out of the box with support for HTTP Basic Auth. While disabled by default for easier setup, authentication and authorization can be added to any environment simply via configuration. ACLs can be set on both API and table levels. This upgrade can be performed with zero downtime in any environment that provides replication.

For external access, Pinot exposes two primary APIs via the following components:

* **pinot-controller** handles cluster management and configuration
* **pinot-broker** handles incoming SQL queries

Both components can be protected via auth and even be configured independently. This makes it is possible to separate accounts for administrative functions such as table creation from accounts that are read the contents of tables in production.

Additionally, all other Pinot components such as **pinot-server** and **pinot-minion** can be configured to authenticate themselves to pinot-controller via the same mechanism. This can be done independently of \(and in addition to\) using 2-way TLS/SSL to ensure intra-cluster authentication on the lower networking layer.

### Quickstart

If you'd rather dive directly into the action with an all-in-one running example, we provide an **AuthQuickstart** runnable with Apache Pinot. This sample app is preconfigured with the settings below but only intended as a dev-friendly, local, single-node  deployment.

### Tokens and User Credentials

The configuration of HTTP Basic Auth in Apache Pinot distinguishes between **Tokens,** which are typically provided to service accounts, and **User Credentials**, which can be used by a human to log onto the web UI or issue SQL queries. While we distinguish these two concepts in the configuration of HTTP Basic Auth, they are fully-convertible formats holding the same authentication information. This distinction allows us to support future token-based authentication methods not reliant on username and password pairs.

Currently, Tokens are merely base64-encoded username & password tuples, similar to those you can find in HTTP Authorization header values \(e.g. see [RFC 7617](https://tools.ietf.org/html/rfc7617)\):

{% tabs %}
{% tab title="User Credentials" %}
```
controller.admin.access.control.principals=admin
controller.admin.access.control.principals.admin.password=verysecret 
```
{% endtab %}

{% tab title="Service Token" %}
```text
controller.segment.fetcher.auth.token=Basic YWRtaW46dmVyeXNlY3JldA
### "Basic " + base64encode("admin:verysecret")
```
{% endtab %}
{% endtabs %}

### Authentication with Web UI and API

Apache Pinot's Basic Auth follows the established standards for HTTP Basic Auth. Credentials are provided via an HTTP Authorization header. The pinot-controller web ui dynamically adapts to your auth configuration and will display a login prompt when basic auth is enabled. Restricted users are still shown all available ui functions, but their operations will fail with an error message if ACLs prohibit access.

If you're using pinot's CLI clients you can provide your credentials either via dedicated username and password arguments, or as pre-serialized token for the HTTP Authorization header. Note, that while most of Apache Pinot's CLI commands support auth, not all of them have been back-fitted yet. If you encounter any such case, you can access the REST API directly, e.g. via curl.

![](../../.gitbook/assets/screen-shot-2021-05-25-at-3.35.05-pm.png)

{% tabs %}
{% tab title="CLI Arguments" %}
```
$ bin/pinot-admin.sh PostQuery \
  -user user -password secret \
  -brokerPort 8000 -query 'SELECT * FROM baseballStats'
```
{% endtab %}

{% tab title="CLI Token" %}
```
$ bin/pinot-admin.sh PostQuery \
  -authToken "Basic dXNlcjpzZWNyZXQ=" \
  -brokerPort 8000 -query 'SELECT * FROM baseballStats'
```
{% endtab %}

{% tab title="HTTP Headers" %}
```text
$ curl http://localhost:8000/query/sql \
  -H 'Authorization: Basic dXNlcjpzZWNyZXQ=' \
  -d '{"sql":"SELECT * FROM baseballStats"}'
```
{% endtab %}
{% endtabs %}

### Manual Token Setup

Before we can require authentication, we first have to distribute service tokens among all pinot nodes. **Nodes use tokens to authenticate themselves as clients** to other nodes. The simplest approach is to create a single internal _admin_ token that is shared across all nodes. Alternatively, every group of components or even every single node can be provided their own token. All tokens live exclusively within a node's local configuration properties \(which in turn also enable injection via environmental variables\).

Here's an example of a simple shared _admin_ credential with password _verysecret_. Note, that the same token is duplicated for different sub-components. While this causes verbosity in a simple scenario, it enables fine-grained control over which tokens are used for different purposes if needed.

{% tabs %}
{% tab title="Controller" %}
```
controller.segment.fetcher.auth.token=Basic YWRtaW46dmVyeXNlY3JldA
```
{% endtab %}

{% tab title="Broker" %}
```
# no tokens required yet
```
{% endtab %}

{% tab title="Minion" %}
```
segment.fetcher.auth.token=Basic YWRtaW46dmVyeXNlY3JldA
 task.auth.token=Basic YWRtaW46dmVyeXNlY3JldA

```
{% endtab %}

{% tab title="Server" %}
```text
pinot.server.segment.fetcher.auth.token=Basic YWRtaW46dmVyeXNlY3JldA
 pinot.server.segment.uploader.auth.token=Basic YWRtaW46dmVyeXNlY3JldA 
pinot.server.instance.auth.token=Basic YWRtaW46dmVyeXNlY3JldA
```
{% endtab %}
{% endtabs %}

### Controller Authentication and Authorization

Pinot-controller has supported custom access control implementations for quite some time. We expanded the scope of this support in 0.8.0+ and added a default implementation for HTTP Basic Auth. Furthermore, the controller's web UI added support for user login workflows and graceful handling of authentication and authorization messages.

Controller Auth can be enabled via configuration in the controller properties. The configuration options allow the specification of usernames and passwords as well as optional ACL restrictions on a per-table and per-access-type \(_CREATE_, _READ_, _UPDATE_, _DELETE_\) basis.

The example below creates two users, _admin_ with password _verysecret_ and _user_ with password _secret_. _admin_ has full access, whereas _user_ is restricted to READ operations and, additionally, to tables named _myusertable_, _baseballStats_, and _stuff_ in all cases where the API calls are table-specific.

```text
controller.admin.access.control.factory.class=org.apache.pinot.controller.api.access.BasicAuthAccessControlFactory

controller.admin.access.control.principals=admin,user
controller.admin.access.control.principals.admin.password=verysecret
controller.admin.access.control.principals.user.password=secret
controller.admin.access.control.principals.user.tables=myusertable,baseballStats,stuff
controller.admin.access.control.principals.user.permissions=READ
```

This configuration will automatically allow other pinot components to access pinot-controller with the shared _admin_ service token set up earlier.

{% hint style="info" %}
If `*.principals.<user>.tables`is not configured, then all the tables are considered accessible for this &lt;user&gt;.
{% endhint %}

### Broker Authentication and Authorization

Pinot-Broker, similar to pinot-controller above, has supported access control for a while now and we added a default implementation for HTTP Basic Auth. Since pinot-broker does not provide a web UI by itself, authentication is only relevant for SQL queries hitting the broker's REST API.

Broker Auth can be enabled via configuration in the broker properties, similar to the controller. The configuration options allow specification of usernames and passwords as well as optional ACL restrictions on a per-table table basis \(access type is always READ\). Note, that it is possible to configure a different set of users, credentials, and permissions for broker access. However, **if you want a user to be able to access data via the query console on the controller web UI,** that user must \(a\) share the **same username and password** on both controller and broker, and \(b\) have **READ permissions and table-level access**.

The example below again creates two users, _admin_ with password _verysecret_ and _user_ with password _secret_. _admin_ has full access, whereas _user_ is restricted to tables named _baseballStats_ and _otherstuff_.

```text
# the factory class property is different for the broker
pinot.broker.access.control.class=org.apache.pinot.broker.broker.BasicAuthAccessControlFactory 

pinot.broker.access.control.principals=admin,user
 pinot.broker.access.control.principals.admin.password=verysecret 
pinot.broker.access.control.principals.user.password=secret 
pinot.broker.access.control.principals.user.tables=baseballStats ,otherstuff
```

### Minion and ingestion jobs

Similar to any API calls, offline jobs executed via command line or minion require credentials as well if ACLs are enabled on pinot-controller. These credentials can be provided either as part of the job spec itself or using CLI arguments and as values \(via **-values**\) or properties \(via **-propertyFile**\) if Groovy templates are defined in the jobSpec.

{% tabs %}
{% tab title="Job Spec YAML" %}
```
authToken: Basic YWRtaW46dmVyeXNlY3JldA
```
{% endtab %}

{% tab title="CLI Arguments" %}
```
$ bin/pinot-admin.sh LaunchDataIngestionJob \
  -user admin -password verysecret \
  -jobSpecFile myJobSpec.yaml
```
{% endtab %}

{% tab title="CLI Token" %}
```
$ bin/pinot-admin.sh LaunchDataIngestionJob \
  -authToken "Basic YWRtaW46dmVyeXNlY3JldA" \
  -jobSpecFile myJobSpec.yaml
```
{% endtab %}

{% tab title="CLI Values" %}
```text
# this requires a reference to "${authToken}" in myJobSpec.yaml!
$ bin/pinot-admin.sh LaunchDataIngestionJob \
  -jobSpecFile myJobSpec.yaml \
  -values "authToken=Basic YWRtaW46dmVyeXNlY3JldA"
```
{% endtab %}
{% endtabs %}

