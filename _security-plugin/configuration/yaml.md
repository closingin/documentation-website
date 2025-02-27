---
layout: default
title: YAML Files
parent: Configuration
nav_order: 3
redirect_from: /docs/security/configuration/yaml/
---

# YAML files

Before running `securityadmin.sh` to load the settings into the `.opendistro_security` index, configure the YAML files in `plugins/opensearch-security/securityconfig`. You might want to back up these files so that you can reuse them on other clusters.

The best use of these YAML files is to configure [reserved and hidden resources]({{site.url}}{{site.baseurl}}/security-plugin/access-control/api#reserved-and-hidden-resources), such as the `admin` and `kibanaserver` users. You might find it easier to create other users, roles, mappings, action groups, and tenants using OpenSearch Dashboards or the REST API.


## internal_users.yml

This file contains any initial users that you want to add to the security plugin's internal user database.

The file format requires a hashed password. To generate one, run `plugins/opensearch-security/tools/hash.sh -p <new-password>`. If you decide to keep any of the demo users, *change their passwords* and re-run [securityadmin.sh]({{site.url}}{{site.baseurl}}/security-plugin/configuration/security-admin/) to apply the new passwords.

```yml
---
# This is the internal user database
# The hash value is a bcrypt hash and can be generated with plugin/tools/hash.sh

_meta:
  type: "internalusers"
  config_version: 2

# Define your internal users here
new-user:
  hash: "$2y$12$88IFVl6IfIwCFh5aQYfOmuXVL9j2hz/GusQb35o.4sdTDAEMTOD.K"
  reserved: false
  hidden: false
  opensearch_security_roles:
  - "specify-some-security-role-here"
  backend_roles:
  - "specify-some-backend-role-here"
  attributes:
    attribute1: "value1"
  static: false

## Demo users

admin:
  hash: "$2a$12$VcCDgh2NDk07JGN0rjGbM.Ad41qVR/YFJcgHp0UGns5JDymv..TOG"
  reserved: true
  backend_roles:
  - "admin"
  description: "Demo admin user"

kibanaserver:
  hash: "$2a$12$4AcgAt3xwOWadA5s5blL6ev39OXDNhmOesEoo33eZtrq2N0YrU3H."
  reserved: true
  description: "Demo user for the OpenSearch Dashboards server"

kibanaro:
  hash: "$2a$12$JJSXNfTowz7Uu5ttXfeYpeYE0arACvcwlPBStB1F.MI7f0U9Z4DGC"
  reserved: false
  backend_roles:
  - "kibanauser"
  - "readall"
  attributes:
    attribute1: "value1"
    attribute2: "value2"
    attribute3: "value3"
  description: "Demo read-only user for OpenSearch dashboards"

logstash:
  hash: "$2a$12$u1ShR4l4uBS3Uv59Pa2y5.1uQuZBrZtmNfqB3iM/.jL0XoV9sghS2"
  reserved: false
  backend_roles:
  - "logstash"
  description: "Demo logstash user"

readall:
  hash: "$2a$12$ae4ycwzwvLtZxwZ82RmiEunBbIPiAmGZduBAjKN0TXdwQFtCwARz2"
  reserved: false
  backend_roles:
  - "readall"
  description: "Demo readall user"

snapshotrestore:
  hash: "$2y$12$DpwmetHKwgYnorbgdvORCenv4NAK8cPUg8AI6pxLCuWf/ALc0.v7W"
  reserved: false
  backend_roles:
  - "snapshotrestore"
  description: "Demo snapshotrestore user"
```

## opensearch.yml

In addition to many OpenSearch settings, this file contains paths to TLS certificates and their attributes, such as distinguished names and trusted certificate authorities.

```yml
plugins.security.ssl.transport.pemcert_filepath: esnode.pem
plugins.security.ssl.transport.pemkey_filepath: esnode-key.pem
plugins.security.ssl.transport.pemtrustedcas_filepath: root-ca.pem
plugins.security.ssl.transport.enforce_hostname_verification: false
plugins.security.ssl.http.enabled: true
plugins.security.ssl.http.pemcert_filepath: esnode.pem
plugins.security.ssl.http.pemkey_filepath: esnode-key.pem
plugins.security.ssl.http.pemtrustedcas_filepath: root-ca.pem
plugins.security.allow_unsafe_democertificates: true
plugins.security.allow_default_init_securityindex: true
plugins.security.authcz.admin_dn:
  - CN=kirk,OU=client,O=client,L=test, C=de

plugins.security.audit.type: internal_opensearch
plugins.security.enable_snapshot_restore_privilege: true
plugins.security.check_snapshot_restore_write_privileges: true
plugins.security.restapi.roles_enabled: ["all_access", "security_rest_api_access"]
plugins.security.system_indices.enabled: true
plugins.security.system_indices.indices: [".opendistro-alerting-config", ".opendistro-alerting-alert*", ".opendistro-anomaly-results*", ".opendistro-anomaly-detector*", ".opendistro-anomaly-checkpoints", ".opendistro-anomaly-detection-state", ".opendistro-reports-*", ".opendistro-notifications-*", ".opendistro-notebooks", ".opendistro-asynchronous-search-response*"]
node.max_local_storage_nodes: 3
```

If you want to run your users' passwords against some validation, specify a regular expression (regex) in this file. You can also include an error message that loads when passwords don't pass validation. The following example demonstrates how to include a regex so OpenSearch requires new passwords to be a minimum of eight characters with at least one uppercase, one lowercase, one digit, and one special character.

Note that OpenSearch validates only users and passwords created through OpenSearch Dashboards or the REST API.

```yml
plugins.security.restapi.password_validation_regex: '(?=.*[A-Z])(?=.*[^a-zA-Z\d])(?=.*[0-9])(?=.*[a-z]).{8,}'
plugins.security.restapi.password_validation_error_message: "Password must be minimum 8 characters long and must contain at least one uppercase letter, one lowercase letter, one digit, and one special character."
```

## whitelist.yml

You can use `whitelist.yml` to allow list any endpoints and HTTP requests. If enabled, all users except the SuperAdmin are allowed access to only the specified endpoints and HTTP requests, and all other HTTP requests associated with the endpoint are denied. For example, if GET `_cluster/settings` is allow listed, users cannot submit PUT requests to `_cluster/settings` to update cluster settings.

Note that while you can configure access to endpoints this way, for most cases, it is still best to configure permissions using the security plugin's users and roles, which have more granular settings.

```yml
---
_meta:
  type: "whitelist"
  config_version: 2

# Description:
# enabled - feature flag.
# if enabled is false, all endpoints are accessible.
# if enabled is true, all users except the SuperAdmin can only submit the allowed requests to the specified endpoints.
# SuperAdmin can access all APIs.
# SuperAdmin is defined by the SuperAdmin certificate, which is configured with the opensearch.yml setting plugins.security.authcz.admin_dn:
# Refer to the example setting in opensearch.yml to learn more about configuring SuperAdmin.
#
# requests - map of allow listed endpoints and HTTP requests

#this name must be config
config:
  enabled: true
  requests:
    /_cluster/settings:
      - GET
    /_cat/nodes:
      - GET
```

To enable PUT requests to cluster settings, add PUT to the list of allowed operations under `/_cluster/settings`.

```yml
requests:
  /_cluster/settings:
    - GET
    - PUT
```

You can also allow list custom indices. `whitelist.yml` doesn't support wildcards, so you must manually specify all of the indices you want to allow list.

```yml
requests: # Only allow GET requests to /sample-index1/_doc/1 and /sample-index2/_doc/1
  /sample-index1/_doc/1:
    - GET
  /sample-index2/_doc/1:
    - GET
```


## roles.yml

This file contains any initial roles that you want to add to the security plugin. Aside from some metadata, the default file is empty, because the security plugin has a number of static roles that it adds automatically.

```yml
---
complex-role:
  reserved: false
  hidden: false
  cluster_permissions:
  - "read"
  - "cluster:monitor/nodes/stats"
  - "cluster:monitor/task/get"
  index_permissions:
  - index_patterns:
    - "opensearch_dashboards_sample_data_*"
    dls: "{\"match\": {\"FlightDelay\": true}}"
    fls:
    - "~FlightNum"
    masked_fields:
    - "Carrier"
    allowed_actions:
    - "read"
  tenant_permissions:
  - tenant_patterns:
    - "analyst_*"
    allowed_actions:
    - "kibana_all_write"
  static: false
_meta:
  type: "roles"
  config_version: 2
```


## roles_mapping.yml

```yml
---
manage_snapshots:
  reserved: true
  hidden: false
  backend_roles:
  - "snapshotrestore"
  hosts: []
  users: []
  and_backend_roles: []
logstash:
  reserved: false
  hidden: false
  backend_roles:
  - "logstash"
  hosts: []
  users: []
  and_backend_roles: []
own_index:
  reserved: false
  hidden: false
  backend_roles: []
  hosts: []
  users:
  - "*"
  and_backend_roles: []
  description: "Allow full access to an index named like the username"
kibana_user:
  reserved: false
  hidden: false
  backend_roles:
  - "kibanauser"
  hosts: []
  users: []
  and_backend_roles: []
  description: "Maps kibanauser to kibana_user"
complex-role:
  reserved: false
  hidden: false
  backend_roles:
  - "ldap-analyst"
  hosts: []
  users:
  - "new-user"
  and_backend_roles: []
_meta:
  type: "rolesmapping"
  config_version: 2
all_access:
  reserved: true
  hidden: false
  backend_roles:
  - "admin"
  hosts: []
  users: []
  and_backend_roles: []
  description: "Maps admin to all_access"
readall:
  reserved: true
  hidden: false
  backend_roles:
  - "readall"
  hosts: []
  users: []
  and_backend_roles: []
kibana_server:
  reserved: true
  hidden: false
  backend_roles: []
  hosts: []
  users:
  - "kibanaserver"
  and_backend_roles: []
```


## action_groups.yml

This file contains any initial action groups that you want to add to the security plugin.

Aside from some metadata, the default file is empty, because the security plugin has a number of static action groups that it adds automatically. These static action groups cover a wide variety of use cases and are a great way to get started with the plugin.

```yml
---
my-action-group:
  reserved: false
  hidden: false
  allowed_actions:
  - "indices:data/write/index*"
  - "indices:data/write/update*"
  - "indices:admin/mapping/put"
  - "indices:data/write/bulk*"
  - "read"
  - "write"
  static: false
_meta:
  type: "actiongroups"
  config_version: 2
```

## tenants.yml

```yml
---
_meta:
  type: "tenants"
  config_version: 2
admin_tenant:
  reserved: false
  description: "Demo tenant for admin user"
```


## nodes_dn.yml

```yml
---
_meta:
  type: "nodesdn"
  config_version: 2

# Define nodesdn mapping name and corresponding values
# cluster1:
#   nodes_dn:
#       - CN=*.example.com
```
