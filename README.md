# puppet-module-keycloak

[![Puppet Forge](http://img.shields.io/puppetforge/v/treydock/keycloak.svg)](https://forge.puppetlabs.com/treydock/keycloak)
[![Build Status](https://travis-ci.org/treydock/puppet-module-keycloak.png)](https://travis-ci.org/treydock/puppet-module-keycloak)

#### Table of Contents

1. [Overview](#overview)
    * [Supported Versions of Keycloak](#supported-versions-of-keycloak)
2. [Usage - Configuration options](#usage)
    * [Keycloak](#keycloak)
    * [Deploy SPI](#deploy-spi)
    * [keycloak_realm](#keycloak_realm)
    * [keycloak_ldap_user_provider](#keycloak_ldap_user_provider)
    * [keycloak_ldap_mapper](#keycloak_ldap_mapper)
    * [keycloak_sssd_user_provider](#keycloak_sssd_user_provider)
    * [keycloak_client](#keycloak_client)
    * [keycloak::client_scope::oidc](#keycloakclient_scopeoidc)
    * [keycloak::client_scope::saml](#keycloakclient_scopesaml)
    * [keycloak_client_scope](#keycloak_client_scope)
    * [keycloak_protocol_mapper](#keycloak_protocol_mapper)
    * [keycloak_client_protocol_mapper](#keycloak_client_protocol_mapper)
    * [keycloak_identity_provider](#keycloak_identity_provider)
    * [Keycloak Flows](#keycloak-flows)
    * [keycloak_api](#keycloak_api)
3. [Reference - Parameter and detailed reference to all options](#reference)
4. [Limitations - OS compatibility, etc.](#limitations)

## Overview

The keycloak module allows easy installation and management of Keycloak.

### Supported Versions of Keycloak

Currently this module supports Keycloak version 8.x to 9.x.

| Keycloak Version | Keycloak Puppet module versions |
| ---------------- | ------------------------------- |
| 3.x              | 2.x                             |
| 4.x - 6.x        | 3.x                             |
| 6.x - 8.x        | 4.x - 5.x                       |
| 8.x - 9.x        | 6.x                             |

## Usage

### keycloak

Install Keycloak using default `h2` database storage.

```puppet
class { 'keycloak': }
```

Install a specific version of Keycloak.

```puppet
class { 'keycloak':
  version           => '6.0.1',
  datasource_driver => 'mysql',
}
```

Upgrading Keycloak version works by changing `version` parameter as long as the `datasource_driver` is not the default of `h2`. An upgrade involves installing the new version without touching the old version, updating the symlink which defaults to `/opt/keycloak`, applying all changes to new version and then restarting the `keycloak` service.

If the previous `version` was `6.0.1` using the following will upgrade to `7.0.0`:

```puppet
class { 'keycloak':
  version           => '7.0.0',
  datasource_driver => 'mysql',
}
```

Install keycloak and use a local MySQL server for database storage

```puppet
include mysql::server
class { 'keycloak':
  datasource_driver   => 'mysql',
  datasource_host     => 'localhost',
  datasource_port     => 3306,
  datasource_dbname   => 'keycloak',
  datasource_username => 'keycloak',
  datasource_password => 'foobar',
}
```

The following example can be used to configure keycloak with a local PostgreSQL server.

```puppet
include postgresql::server
class { 'keycloak':
    datasource_driver     => 'postgresql',
    datasource_host       => 'localhost',
    datasource_port       => 5432,
    datasource_dbname     => 'keycloak',
    datasource_username   => 'keycloak',
    datasource_password   => 'foobar',
}
```

Configure a SSL certificate truststore and add a LDAP server's certificate to the truststore.

```puppet
class { 'keycloak':
  truststore                              => true,
  truststore_password                     => 'supersecret',
  truststore_hostname_verification_policy => 'STRICT',
}
keycloak::truststore::host { 'ldap1.example.com':
  certificate => '/etc/openldap/certs/0a00000.0',
}
```

Setup Keycloak to proxy through Apache HTTPS.

```puppet
class { 'keycloak':
  proxy_https => true
}
apache::vhost { 'idp.example.com':
  servername => 'idp.example.com',
  port        => '443',
  ssl         => true,
  manage_docroot  => false,
  docroot         => '/var/www/html',
  proxy_preserve_host => true,
  proxy_pass          => [
    {'path' => '/', 'url' => 'http://localhost:8080/'}
  ],
  request_headers     => [
    'set X-Forwarded-Proto "https"',
    'set X-Forwarded-Port "443"'
  ],
  ssl_cert            => '/etc/pki/tls/certs/idp.example.com/crt',
  ssl_key             => '/etc/pki/tls/private/idp.example.com.key',
}
```

Setup a host for theme development so that theme changes don't require a service restart, not recommended for production.

```puppet
class { 'keycloak':
  theme_static_max_age  => -1,
  theme_cache_themes    => false,
  theme_cache_templates => false,
}
```

Run Keycloak using standalone clustered mode (multicast):

```puppet
class { 'keycloak':
  operating_mode => 'clustered',
}
```

Run Keycloak using standalone clustered mode (JDBC_PING):

> [JDBC_PING](http://jgroups.org/manual/#_jdbc_ping) uses port **7600** to ensure cluster members are discoverable by each other. This module **does NOT manage firewall changes**.

```puppet
class { 'keycloak':
  operating_mode             => 'clustered',
  datasource_driver          => 'postgresql',
  enable_jdbc_ping           => true,
  jboss_bind_private_address => $facts['networking']['ip'],
  jboss_bind_public_address  => $facts['networking']['ip'],
}

# your puppet code to open port 7600
# ...
# ...
```

### Deploy SPI

A simple example of deploying a custom SPI from a URL:

```puppet
keycloak::spi_deployment { 'duo-spi':
  ensure        => 'present',
  deployed_name => 'keycloak-duo-spi-jar-with-dependencies.jar',
  source        => 'https://example.com/files/keycloak-duo-spi-jar-with-dependencies.jar',
}
```

The `source` can be a URL or a file path like `/tmp/foo.jar` or prefixed with `file://` or `puppet://`

The following example will deploy a custom SPI then check the Keycloak API for the resource to exist.
This is useful to ensure SPI is loaded into Keycloak before attempting to add custom resources.

```puppet
keycloak::spi_deployment { 'duo-spi':
  deployed_name => 'keycloak-duo-spi-jar-with-dependencies.jar',
  source        => 'https://example.com/files/keycloak-duo-spi-jar-with-dependencies.jar',
  test_url      => 'authentication/authenticator-providers',
  test_key      => 'id',
  test_value    => 'duo-mfa-authenticator',
  test_realm    => 'test',
  before        => Keycloak_flow_execution['duo-mfa-authenticator under form-browser-with-duo on test'],
}
```

### keycloak_realm

Define a Keycloak realm that uses username and not email for login and to use a local branded theme.

```puppet
keycloak_realm { 'test':
  ensure                   => 'present',
  remember_me              => true,
  login_with_email_allowed => false,
  login_theme              => 'my_theme',
}
```

**NOTE:** If the flow properties such as `browser_flow` are changed from their defaults then this value will not be set when a realm is first created. The value will also not be updated if the flow does not exist. For new realms you will have to run Puppet twice in order to create the flows then update the realm setting.

### keycloak\_ldap\_user_provider

Define a LDAP user provider so that authentication can be performed against LDAP.  The example below uses two LDAP servers, disables importing of users and assumes the SSL certificates are trusted and do not require being in the truststore.

 ```puppet
keycloak_ldap_user_provider { 'LDAP on test':
  ensure             => 'present',
  users_dn           => 'ou=People,dc=example,dc=com',
  connection_url     => 'ldaps://ldap1.example.com:636 ldaps://ldap2.example.com:636',
  import_enabled     => false,
  use_truststore_spi => 'never',
}
```

**NOTE** The `Id` for the above resource would be `LDAP-test` where the format is `${resource_name}-${realm}`.

### keycloak\_ldap_mapper

Use the LDAP attribute 'gecos' as the full name attribute.

```puppet
keycloak_ldap_mapper { 'full name for LDAP-test on test:
  ensure         => 'present',
  resource_name  => 'full name',
  type           => 'full-name-ldap-mapper',
  ldap_attribute => 'gecos',
}
```

### keycloak\_sssd\_user\_provider

Define SSSD user provider.  **NOTE** This type requires that SSSD be properly configured and Keycloak service restarted after SSSD ifp service is setup.  Also requires `keycloak` class be called with `with_sssd_support` set to `true`.

```puppet
keycloak_sssd_user_provider { 'SSSD on test':
  ensure => 'present',
}
```

### keycloak_client

Register a client.

```puppet
keycloak_client { 'www.example.com':
  ensure          => 'present',
  realm           => 'test',
  redirect_uris   => [
    "https://www.example.com/oidc",
    "https://www.example.com",
  ],
  client_template => 'oidc-clients',
  secret          => 'supersecret',
}
```

### keycloak::client_scope::oidc

Defined type that can be used to define both `keycloak_client_scope` and `keycloak_protocol_mapper` resources for OpenID Connect. 

```puppet
keycloak::client_scope::oidc { 'oidc-clients':
  realm => 'test',
}
```

### keycloak::client_scope::saml

Defined type that can be used to define both `keycloak_client_scope` and `keycloak_protocol_mapper` resources for SAML. 

```puppet
keycloak::client_scope::saml { 'saml-clients':
  realm => 'test',
}
```

### keycloak\_client_scope

Define a Client Scope of `email` for realm `test` in Keycloak:

```puppet
keycloak_client_scope { 'email on test':
  protocol => 'openid-connect',
}
```

### keycloak\_protocol_mapper

Associate a Protocol Mapper to a given Client Scope.  The name in the following example will add the `email` protocol mapper to client scope `oidc-email` in the realm `test`.

```puppet
keycloak_protocol_mapper { "email for oidc-email on test":
  claim_name     => 'email',
  user_attribute => 'email',
}
```

### keycloak\_client\_protocol\_mapper

Add `email` protocol mapper to `test.example.com` client in realm `test`

```puppet
keycloak_client_protocol_mapper { "email for test.example.com on test":
  claim_name     => 'email',
  user_attribute => 'email',
}
```

### keycloak\_identity\_provider

Add `cilogon` identity provider to `test` realm

```puppet
keycloak_identity_provider { 'cilogon on test':
  ensure                        => 'present',
  display_name                  => 'CILogon',
  provider_id                   => 'oidc',
  first_broker_login_flow_alias => 'browser',
  client_id                     => 'cilogon:/client_id/foobar',
  client_secret                 => 'supersecret',
  user_info_url                 => 'https://cilogon.org/oauth2/userinfo',
  token_url                     => 'https://cilogon.org/oauth2/token',
  authorization_url             => 'https://cilogon.org/authorize',
}
```

### Keycloak Flows

The following is an example of deploying a custom Flow.
The name for the top level flow is `$alias on $realm`
The name for an execution is `$provider under $flow on $realm`.
The name for the flow under a top level flow is `$alias under $flow_alias on $realm`.

```puppet
keycloak_flow { 'browser-with-duo on test':
  ensure => 'present',
}
keycloak_flow_execution { 'auth-cookie under browser-with-duo on test':
  ensure       => 'present',
  configurable => false,
  display_name => 'Cookie',
  index        => 0,
  requirement  => 'ALTERNATIVE',
}
keycloak_flow_execution { 'identity-provider-redirector under browser-with-duo on test':
  ensure       => 'present',
  configurable => true,
  display_name => 'Identity Provider Redirector',
  index        => 1,
  requirement  => 'ALTERNATIVE',
}
keycloak_flow { 'form-browser-with-duo under browser-with-duo on test':
  ensure      => 'present',
  index       => 2,
  requirement => 'ALTERNATIVE',
  top_level   => false,
}
keycloak_flow_execution { 'auth-username-password-form under form-browser-with-duo on test':
  ensure       => 'present',
  configurable => false,
  display_name => 'Username Password Form',
  index        => 0,
  requirement  => 'REQUIRED',
}
keycloak_flow_execution { 'duo-mfa-authenticator under form-browser-with-duo on test':
  ensure       => 'present',
  configurable => true,
  display_name => 'Duo MFA',
  alias        => 'Duo',
  config       => {
    "duomfa.akey"    => "foo-akey",
    "duomfa.apihost" => "api-foo.duosecurity.com",
    "duomfa.skey"    => "secret",
    "duomfa.ikey"    => "foo-ikey",
    "duomfa.groups"  => "duo"
  },
  requirement  => 'REQUIRED',
  index        => 1,
}
```

### keycloak\_api

The keycloak_api type can be used to define how this module's types access the Keycloak API if this module is only used for the types/providers and the module's `kcadm-wrapper.sh` is not installed.

 ```puppet
keycloak_api { 'keycloak'
  install_dir => '/opt/keycloak',
  server     => 'http://localhost:8080/auth',
  realm      => 'master',
  user       => 'admin',
  password   => 'changeme',
}
```

The path for `install_dir` will be joined with `bin/kcadm.sh` to produce the full path to `kcadm.sh`.

## Reference

[http://treydock.github.io/puppet-module-keycloak/](http://treydock.github.io/puppet-module-keycloak/)

## Limitations

This module has been tested on:

* CentOS 7 x86_64
* RedHat 7 x86_64
* Debian 9 x86_64
* Ubuntu 18.04 x86_64
