# Artemis ActiveMQ v7.x Console basic authentication & RBAC configuration

## Problem Statement

As a user I would like to configure AMQ 7.x to allow users with distinct roles to be able to login on the broker console; and only carry out operations explicitly allowed by their role(s). As an example I will setup three (3) users with the following roles:

- adminuser: This user will have the `admin` role which grants the user permission to run all CRUD operations supported by the broker; CRUD on users, addresses, queues are few examples.

- artemisuser: This user will have the `consumer, producer` roles which grants the user permissions to discover queues, addresses, consume and produce messages.

- supportuser: This user will have the `support` role which grants the user permission to only replay dead-letter-queues (DLQ) and expiry-queues (ExpiryQueue).

This solution was tested in an OpenShift/Kubernetes environmnent; however the same concepts generally apply in a standalone deployment model.

> IMPORTANT: `.xml` and `.properties` files in the `broker-configs` directory are examples. It is best to get these config files from the live broker and then make modifications.

Sample AMQ config files are avalaible [this amq-artemis repository.](https://github.com/apache/activemq-artemis/tree/main/artemis-cli/src/main/resources/org/apache/activemq/artemis/cli/commands/etc)


## Prerequisites

- Access to an up & running OpenShift cluster
- Up & Running AMQ Broker Pod
- Mounting ConfigMaps, Secrets to AMQ Broker supported

## Implementation

### 1. Prepare the [`artemis-users.properties`](broker-configs/artemis-users.properties) file

Althought plain text passwords are supported, one should mask passwords using the `BROKER_INSTANCE_DIR/bin/artemis mask PlainTextPassword` command.

Once password is masked, surround it with `ENC()` when placing it in the [`artemis-users.properties`](broker-configs/artemis-users.properties) file.

**Example:**

```properties
# Note: Password is not encrypted for simpliciy; 
# username = password
# For example:
# adminuser = ENC(45F26537AA4CD6CD473B19F9EEE17DF02:760ABB1B5B8A03D47399250879)

adminuser = ADMIN_SECRET_PASS
artemisuser = ARTEMIS_SECRET_PASS
supportuser = SUPPORT_SECRET_PASS
```

### 2. Prepare the [`artemis-roles.properties`](broker-configs/artemis-roles.properties) file

**Example:**

```properties
# Example:
# role(s) = user

admin = adminuser
consumer, producer = artemisuser
support = supportuser
```

### 3. Configure [`broker.xml`](broker-configs/broker.xml) `<security-settings>`

Add the following `<security-settings>` configs. Follow [this link](https://activemq.apache.org/components/artemis/documentation/1.0.0/security.html) to learn more about role based security for ActiveMQ adresses.

**Before:**

```xml
<security-settings>
    <security-setting match="#">
    <permission type="createNonDurableQueue" roles="${role}"/>
    <permission type="deleteNonDurableQueue" roles="${role}"/>
    <permission type="createDurableQueue" roles="${role}"/>
    <permission type="deleteDurableQueue" roles="${role}"/>
    <permission type="createAddress" roles="${role}"/>
    <permission type="deleteAddress" roles="${role}"/>
    <permission type="consume" roles="${role}"/>
    <permission type="browse" roles="${role}"/>
    <permission type="send" roles="${role}"/>
    <!-- we need this otherwise ./artemis data imp wouldn't work -->
    <permission type="manage" roles="${role}"/>
    </security-setting>
</security-settings>
```

**After:**

```xml
<security-settings>
    <security-setting match="#">
        <permission type="createNonDurableQueue" roles="admin"/>
        <permission type="deleteNonDurableQueue" roles="admin"/>
        <permission type="createDurableQueue" roles="admin"/>
        <permission type="deleteDurableQueue" roles="admin"/>
        <permission type="createAddress" roles="admin"/>
        <permission type="deleteAddress" roles="admin"/>
        <!-- Grant consumer, producer users permission to consume from all queues -->
        <permission type="consume" roles="admin, consumer"/>
        <permission type="browse" roles="admin, consumer, producer, support"/>
        <!-- Grant consumer, producer users permission to push to all queues -->
        <permission type="send" roles="admin, producer"/>
        <permission type="manage" roles="admin"/>
    </security-setting>
    <!-- Adding DLQ permissions -->
    <security-setting match="DLQ#">
        <permission type="send" roles="admin, consumer, support"/>
        <permission type="consume" roles="admin, producer, support"/>
        <permission type="manage" roles="admin, support"/>
        <permission type="browse" roles="admin, consumer, producer, support"/>
    </security-setting>
    <!-- Adding ExpiryQueue permissions -->
    <security-setting match="ExpiryQueue#">
        <permission type="send" roles="admin, consumer, support"/>
        <permission type="consume" roles="admin, producer, support"/>
        <permission type="manage" roles="admin, support"/>
        <permission type="browse" roles="admin, consumer, producer, support"/>
    </security-setting>
</security-settings>

```

### 4. Configure [`management.xml`](broker-configs/management.xml) to setup console RBAC permissions

Add the following `<role-access>` configs. Follow [this link](https://access.redhat.com/webassets/avalon/d/red_hat_jboss_enterprise_application_platform/7.4/javadocs/org/apache/activemq/artemis/api/core/management/class-use/Operation.html) to learn more about ActiveMQ managment ('console') operations.

**Before:**

```xml
<role-access>
    <match domain="org.apache.activemq.artemis">
        <access method="list*" roles="admin"/>
        <access method="get*" roles="admin"/>
        <access method="is*" roles="admin"/>
        <access method="set*" roles="admin"/>
        <access method="browse*" roles="admin"/>
        <access method="count*" roles="admin"/>
        <access method="*" roles="admin"/>
    </match>
</role-access>
```

**After:**

```xml
<role-access>
    <!-- Allow all roles to discover (list, get, browse) all artemis MBeans when they login to the console -->
    <match domain="org.apache.activemq.artemis">
        <access method="list*" roles="admin, consumer, producer, support"/>
        <access method="get*" roles="admin, consumer, producer, support"/>
        <access method="is*" roles="admin"/>
        <access method="set*" roles="admin"/>
        <access method="browse*" roles="admin, consumer, producer, support"/>
        <access method="count*" roles="admin, consumer, producer, support"/>
        <access method="*" roles="admin"/>
    </match>
    <!-- queue=DLQ: support can do everything but delete*, create*, destroy* -->
    <match domain="org.apache.activemq.artemis" key="queue=DLQ"> 
        <!-- Only allow admin role to execute destructive operations; more can be added -->  
        <access method="delete*" roles="admin"/>
        <access method="create*" roles="admin"/>
        <access method="destroy*" roles="admin"/>   
        <!-- All other operations -->
        <access method="*" roles="admin, support"/>
    </match>
    <!-- queue=ExpiryQueue: support can do everything but delete*, create*, destroy* -->
    <match domain="org.apache.activemq.artemis" key="queue=ExpiryQueue">   
        <!-- Only allow admin role to execute destructive operations; more can be added -->  
        <access method="delete*" roles="admin"/>
        <access method="create*" roles="admin"/>
        <access method="destroy*" roles="admin"/>   
        <!-- All other operations -->
        <access method="*" roles="admin,support"/>
    </match>
</role-access>
```

### 5. Set `AMQ_*` template environment variables

The following need to be set:

```yaml
env:
# List of roles to be enabled
- name: AMQ_ROLE
  value: "admin,consumer,producer,support"
# Admin username
- name: AMQ_USER
  value: adminuser
# Admin Password
- name: AMQ_PASSWORD
  value: ADMIN_PASS
# Enable RBAC login
- name: AMQ_ENABLE_MANAGEMENT_RBAC
  value: true
# Require Console Login
- name: AMQ_CONSOLE_REQUIRE_LOGIN
  value: true
# Configs Directory
- name: AMQ_CONFIGS_DIR
  value: /opt/amq/conf
```

### 6. Attach ConfigMap/Secrets containing `broker-configs` files to AMQ deployment manifest


```yaml
volumes:
  - name: amq-broker-configs
    configMap:
      name:  amq-broker-configs
      items:
        - key: broker.xml
          path: broker.xml
        - key: management.xml
          path: management.xml
        - key: artemis-roles.properties
          path: artemis-roles.properties
        - key: artemis-users.properties
          path: artemis-users.properties
```

### 7. Mount `amq-broker-configs` volume item to `${AMQ_CONFIGS_DIR}`

```yaml
volumeMounts:
- mountPath: ${AMQ_CONFIGS_DIR}
  name: amq-broker-configs
```

### Summary

This guide demonstrated how to configure basic auth and RBAC for the AMQ 7.x console interface.

## Sources

- [Sample AMQ config files](https://github.com/apache/activemq-artemis/tree/main/artemis-cli/src/main/resources/org/apache/activemq/artemis/cli/commands/etc)
- [ActiveMQ role based security permissions for addresses](https://activemq.apache.org/components/artemis/documentation/1.0.0/security.html)
- [Uses of ActiveMQ management operations](https://access.redhat.com/webassets/avalon/d/red_hat_jboss_enterprise_application_platform/7.4/javadocs/org/apache/activemq/artemis/api/core/management/class-use/Operation.html)

