---
layout: post
title:  "OCI Authentication in Helidon"
date:   2026-04-09 08:00:00 +0200
categories: [helidon, oci, security, authentication]
description: OCI authentication support in Helidon 4.4.0 for SE and MP applications.
---

# OCI Authentication in Helidon
_How Helidon 4.4.0 discovers and exposes OCI `BasicAuthenticationDetailsProvider` instances._

If you use the [OCI Java SDK](https://docs.oracle.com/en-us/iaas/Content/API/SDKDocs/javasdkgettingstarted.htm)
directly, sooner or later you need a
[`BasicAuthenticationDetailsProvider`](https://docs.oracle.com/en-us/iaas/tools/java/latest/com/oracle/bmc/auth/BasicAuthenticationDetailsProvider.html).
That is the object OCI SDK clients use for request signing,
tenant and user identification, session tokens, or workload identity.

Helidon 4.4.0 has dedicated OCI integration in `integrations/oci` that turns this into framework
plumbing instead of application boilerplate. Instead of manually building the authentication provider
in each application, Helidon can discover one, expose it through Service Registry, and let the rest
of the application inject it.

This post walks through the supported authentication variants, which ones need extra modules, and how
to consume the resulting provider in Helidon SE and MP applications.

 * [Where OCI auth lives in Helidon 4.4](#where-oci-auth-lives-in-helidon-44)
 * [Base module and configuration sources](#base-module-and-configuration-sources)
 * [Built-in authentication methods](#built-in-authentication-methods)
 * [Principal-based methods and extra dependencies](#principal-based-methods-and-extra-dependencies)
 * [Automatic selection and explicit selection](#automatic-selection-and-explicit-selection)
 * [Injecting BasicAuthenticationDetailsProvider](#injecting-basicauthenticationdetailsprovider)
 * [Conclusion](#conclusion)

## Where OCI auth lives in Helidon 4.4

There are currently three relevant pieces in Helidon 4.4.0:

 * `io.helidon.integrations.oci:helidon-integrations-oci`
 * ⚠️ `io.helidon.integrations.oci.sdk:helidon-integrations-oci-sdk-runtime`
 * `io.helidon.integrations.oci.sdk:helidon-integrations-oci-sdk-cdi`

The important one for new Helidon SE code is `helidon-integrations-oci`. It is the current Service
Registry based integration and it is the module this article focuses on.

⚠️ The older `helidon-integrations-oci-sdk-runtime` still exists, but it is deprecated since Helidon
4.1.0 and replaced by `helidon-integrations-oci`.

For Helidon MP, CDI support still lives in `helidon-integrations-oci-sdk-cdi`, so I will return to
that module in the final section when looking at `@Inject BasicAuthenticationDetailsProvider`.

## Base module and configuration sources

The base dependency is straightforward:

```xml
<dependency>
    <groupId>io.helidon.integrations.oci</groupId>
    <artifactId>helidon-integrations-oci</artifactId>
    <version>${helidon.version}</version>
</dependency>
```

Once this module is on the classpath, Helidon can publish
`com.oracle.bmc.auth.BasicAuthenticationDetailsProvider` through Service Registry.

One detail is worth keeping in mind from the start: there are two different configuration worlds.

 * `oci-config.yaml` is Helidon configuration, typically using the `helidon.oci.*` prefix.
 * [`~/.oci/config`](https://docs.oracle.com/iaas/Content/API/Concepts/sdkconfig.htm) is the standard OCI SDK configuration file.

That distinction matters because some methods are driven by `oci-config.yaml`, some by `~/.oci/config`,
and `session-token` can work with either.

The Helidon OCI module reads its configuration from:

 * environment variables
 * system properties
 * `oci-config.yaml` on the filesystem
 * `oci-config.yaml` on the classpath

For `oci-config.yaml` itself, the default lookup is very literal:

 * on the filesystem, Helidon looks for `./oci-config.yaml`, which means `oci-config.yaml` in the current working directory
 * on the classpath, Helidon looks for a resource named `oci-config.yaml`, which in a typical application means `src/main/resources/oci-config.yaml`

If `oci-config.yaml` is not present, Helidon does not fail just because of that. With the default
`authentication-method: auto`, it can still discover credentials from environment variables, system
properties, and the standard OCI SDK configuration file such as
[`~/.oci/config`](https://docs.oracle.com/iaas/Content/API/Concepts/sdkconfig.htm).

A minimal starting point looks like this:

```yaml
helidon.oci:
  authentication-method: auto
```

## Built-in authentication methods

The base `helidon-integrations-oci` module supports three authentication methods out of the box:

| Method | Backing OCI SDK provider | Typical use |
|--------|---------------------------|-------------|
| `config` | [`SimpleAuthenticationDetailsProvider`](https://docs.oracle.com/en-us/iaas/tools/java/latest/com/oracle/bmc/auth/SimpleAuthenticationDetailsProvider.html) | Full application-managed configuration |
| `config-file` | [`ConfigFileAuthenticationDetailsProvider`](https://docs.oracle.com/en-us/iaas/tools/java/latest/com/oracle/bmc/auth/ConfigFileAuthenticationDetailsProvider.html) | Local development or admin tooling using `~/.oci/config` |
| `session-token` | [`SessionTokenAuthenticationDetailsProvider`](https://docs.oracle.com/en-us/iaas/tools/java/latest/com/oracle/bmc/auth/SessionTokenAuthenticationDetailsProvider.html) | Temporary sessions and token-based flows |

### Config

`config` is the most explicit option. All important values are provided through Helidon config, so
the application does not depend on an OCI config file being present.

```yaml
helidon.oci:
  authentication-method: config
  authentication:
    config:
      region: eu-frankfurt-1
      fingerprint: "20:3b:aa:bb:cc:dd"
      tenant-id: "ocid1.tenancy.oc1..aaaa..."
      user-id: "ocid1.user.oc1..aaaa..."
      private-key:
        resource-path: "keys/oci_api_key.pem"
      passphrase: "${OCI_API_KEY_PASSPHRASE}"
```

This method is nice when you want the application to be fully self-contained. The private key can be
provided as a Helidon `Resource`, which means classpath resource, filesystem resource, base64 content,
or plain text content. If you omit the private key entirely, Helidon falls back to
`~/.oci/oci_api_key.pem`.

For the underlying OCI side of this setup, see
[Required Keys and OCIDs](https://docs.oracle.com/iaas/Content/API/Concepts/apisigningkey.htm)
and the
[SimpleAuthenticationDetailsProvider JavaDoc](https://docs.oracle.com/en-us/iaas/tools/java/latest/com/oracle/bmc/auth/SimpleAuthenticationDetailsProvider.html).

You can select `config` explicitly as shown above, but `auto` can also reach it when
`helidon.oci.authentication.config` is present.

### Config File

`config-file` delegates to the standard
[OCI SDK and CLI configuration file](https://docs.oracle.com/iaas/Content/API/Concepts/sdkconfig.htm).

```yaml
helidon.oci:
  authentication-method: config-file
  authentication:
    config-file:
      profile: DEFAULT
      path: "/home/app/.oci/config"
```

If `path` is omitted, the OCI SDK default lookup is used. This is usually the best choice for local
development, developer laptops, bastion hosts, or scripts running with a conventional `~/.oci/config`
layout.

This can also be used through `auto`. If Helidon finds a usable OCI config file and no higher-priority
method wins first, it can resolve `config-file` without forcing `authentication-method`.

The related OCI SDK type is
[ConfigFileAuthenticationDetailsProvider](https://docs.oracle.com/en-us/iaas/tools/java/latest/com/oracle/bmc/auth/ConfigFileAuthenticationDetailsProvider.html).

### Session Token

`session-token` uses
[`SessionTokenAuthenticationDetailsProvider`](https://docs.oracle.com/en-us/iaas/tools/java/latest/com/oracle/bmc/auth/SessionTokenAuthenticationDetailsProvider.html).
Helidon enables it in two ways:

 * you configure `helidon.oci.authentication.session-token`
 * or the OCI config file contains `security_token_file`

An explicit `oci-config.yaml` setup can look like this:

```yaml
helidon.oci:
  authentication-method: session-token
  authentication:
    session-token:
      region: eu-frankfurt-1
      fingerprint: "20:3b:aa:bb:cc:dd"
      tenant-id: "ocid1.tenancy.oc1..aaaa..."
      user-id: "ocid1.user.oc1..aaaa..."
      private-key-path: "/home/app/.oci/sessions/DEFAULT/oci_api_key.pem"
      session-token-path: "/home/app/.oci/sessions/DEFAULT/token"
      session-lifetime-hours: 8
      refresh-period: PT55M
```

This method is useful when you want short-lived credentials instead of a long-lived API signing key.
If you already have a normal OCI config file with `security_token_file`, Helidon can build the
provider from that as well.

The OCI configuration file format for token-based sessions is described in the
[SDK and CLI Configuration File](https://docs.oracle.com/iaas/Content/API/Concepts/sdkconfig.htm)
documentation.

Again, explicit selection is optional. `auto` can reach `session-token` too, either from
`helidon.oci.authentication.session-token` or from an OCI config file that contains
`security_token_file`.

## Principal-based methods and extra dependencies

The principal-based variants are intentionally split into extra modules. The base
`helidon-integrations-oci` module does not force all of them on every application.

| Method | Extra artifact | Availability check |
|--------|----------------|--------------------|
| `instance-principal` | `helidon-integrations-oci-authentication-instance` | OCI instance metadata service must be reachable |
| `resource-principal` | `helidon-integrations-oci-authentication-resource` | `OCI_RESOURCE_PRINCIPAL_VERSION` must be set |
| `oke-workload-identity` | `helidon-integrations-oci-authentication-oke-workload` | OKE service account CA certificate file must exist |

### Instance Principal

For Compute instances running on OCI, add:

```xml
<dependency>
    <groupId>io.helidon.integrations.oci.authentication</groupId>
    <artifactId>helidon-integrations-oci-authentication-instance</artifactId>
    <version>${helidon.version}</version>
</dependency>
```

You can select it explicitly, but `auto` can also reach it when the module is present and the OCI
instance metadata service is available:

```yaml
helidon.oci:
  authentication-method: instance-principal
```

This provider becomes available only when Helidon can reach OCI Instance Metadata Service. In other
words, putting the dependency on the classpath is necessary, but not sufficient. The code still checks
that it is actually running in an OCI instance environment.

On the OCI side, the best background reading is
[Calling Services from an Instance](https://docs.oracle.com/iaas/Content/Identity/callresources/callingservicesfrominstances.htm)
and the
[InstancePrincipalsAuthenticationDetailsProvider JavaDoc](https://docs.oracle.com/en-us/iaas/tools/java/latest/com/oracle/bmc/auth/InstancePrincipalsAuthenticationDetailsProvider.html).

### Resource Principal

For OCI Functions and similar resource-principal environments, add:

```xml
<dependency>
    <groupId>io.helidon.integrations.oci.authentication</groupId>
    <artifactId>helidon-integrations-oci-authentication-resource</artifactId>
    <version>${helidon.version}</version>
</dependency>
```

And configure:

```yaml
helidon.oci:
  authentication-method: resource-principal
```

At runtime, Helidon checks for `OCI_RESOURCE_PRINCIPAL_VERSION`. If that environment variable is not
present, this method is considered unavailable and will not produce a provider.

`auto` can also reach this method. The explicit setting is only needed if you want strict,
fail-fast selection instead of discovery.

For the OCI side of the setup, see
[Accessing Other Oracle Cloud Infrastructure Resources from Running Functions](https://docs.oracle.com/iaas/Content/Functions/Tasks/functionsaccessingociresources.htm)
and the
[ResourcePrincipalAuthenticationDetailsProvider JavaDoc](https://docs.oracle.com/en-us/iaas/tools/java/latest/com/oracle/bmc/auth/ResourcePrincipalAuthenticationDetailsProvider.html).

### OKE workload identity

For workloads running on OKE, add:

```xml
<dependency>
    <groupId>io.helidon.integrations.oci.authentication</groupId>
    <artifactId>helidon-integrations-oci-authentication-oke-workload</artifactId>
    <version>${helidon.version}</version>
</dependency>
```

And configure:

```yaml
helidon.oci:
  authentication-method: oke-workload-identity
```

This module is slightly different from the other two. Under the hood it also pulls the OCI SDK addon
for OKE workload identity, so you do not have to wire that dependency separately in your application.

Availability is detected through the Kubernetes service account CA certificate path. Helidon checks
`OCI_KUBERNETES_SERVICE_ACCOUNT_CERT_PATH` first and otherwise falls back to
`/var/run/secrets/kubernetes.io/serviceaccount/ca.crt`.

This one is reachable through `auto` as well, provided the module is present and the workload identity
environment is actually available.

The OCI platform side is documented in
[Granting Workloads Access to OCI Resources](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contenggrantingworkloadaccesstoresources.htm).

### Shared tuning for principal-based providers

Instance principal, resource principal, and OKE workload identity all reuse the common `helidon.oci`
settings below:

```yaml
helidon.oci:
  authentication-timeout: PT10S
  imds-detect-retries: 3
  federation-endpoint: "https://auth.eu-frankfurt-1.oraclecloud.com/"
  imds-base-uri: "http://169.254.169.254/opc/v2/"
  tenant-id: "ocid1.tenancy.oc1..aaaa..."
```

You normally do not need all of them, but it is good to know that the principal-based builder
providers share this customization surface.

## Automatic selection and explicit selection

By default, Helidon uses:

```yaml
helidon.oci:
  authentication-method: auto
```

With `auto`, Helidon tries the available authentication methods and uses the first one that can
actually build a `BasicAuthenticationDetailsProvider`.

In practice, the built-in ordering in Helidon 4.4 looks like this:

 1. `config`
 2. `session-token`
 3. `config-file`
 4. `resource-principal`
 5. `instance-principal`
 6. `oke-workload-identity`

This is convenient when you want the same application to work on a laptop and then in OCI, but when
the deployment target is known, I usually prefer an explicit `authentication-method`. It removes
guessing and makes failures much easier to understand.

Explicit selection is also strict by design. If you choose `instance-principal`,
`resource-principal`, or any other concrete method and that provider cannot be created in the current
environment, Helidon fails instead of silently falling back to something else.

⚠️ One more detail: if you read older OCI CDI examples or older internals in the Helidon source tree, the
strategy names are not completely identical. The current Service Registry based integration uses names
such as `instance-principal` and `session-token`, while the older CDI strategy layer still refers to
things like `instance-principals`, `session-token-builder`, and `session-token-config-file`. That does
not change how the new SE module works, but it is useful context when comparing examples across
generations of the integration.

## Injecting BasicAuthenticationDetailsProvider

Once Helidon has resolved the authentication method, the useful part begins: the resulting
`BasicAuthenticationDetailsProvider` can be injected and reused across OCI SDK clients.

### Helidon SE constructor injection

With the current Service Registry based integration, the cleanest option is constructor injection:

```java
import com.oracle.bmc.auth.BasicAuthenticationDetailsProvider;
import com.oracle.bmc.objectstorage.ObjectStorageClient;
import io.helidon.service.registry.Service;

@Service.Singleton
class BucketService {
    private final BasicAuthenticationDetailsProvider auth;

    @Service.Inject
    BucketService(BasicAuthenticationDetailsProvider auth) {
        this.auth = auth;
    }

    ObjectStorageClient client() {
        return ObjectStorageClient.builder().build(auth);
    }
}
```

That keeps OCI authentication discovery in the framework layer and leaves your application code with a
simple dependency on the provider contract.

### Static lookup with Services.get(...)

For code that is not itself a managed service, Helidon also exposes the provider through static Service
Registry lookup:

```java
import com.oracle.bmc.auth.BasicAuthenticationDetailsProvider;
import io.helidon.service.registry.Services;

BasicAuthenticationDetailsProvider auth =
        Services.get(BasicAuthenticationDetailsProvider.class);
```

That is handy in `main` methods, integration bridges, or migration code. In normal declarative SE
services I still prefer constructor injection, but `Services.get(...)` is perfectly valid when you
need an escape hatch.

### Helidon MP CDI injection

For Helidon MP, use the CDI extension:

```xml
<dependency>
    <groupId>io.helidon.integrations.oci.sdk</groupId>
    <artifactId>helidon-integrations-oci-sdk-cdi</artifactId>
    <version>${helidon.version}</version>
</dependency>
```

Then the provider can be injected as a CDI bean:

```java
import com.oracle.bmc.auth.BasicAuthenticationDetailsProvider;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class BucketService {
    @Inject
    BasicAuthenticationDetailsProvider auth;
}
```

That is the MP story in Helidon 4.4.0: the new Service Registry based module is the main path for SE,
while MP still uses the CDI integration module for direct `@Inject` support.

## Conclusion

Helidon 4.4.0 gives OCI authentication a clean, modular shape:

 * `config`, `config-file`, and `session-token` are available directly from `helidon-integrations-oci`
 * `instance-principal`, `resource-principal`, and `oke-workload-identity` are opt-in add-ons
 * once resolved, the provider is available as `BasicAuthenticationDetailsProvider`

For local development, `config-file` is usually the easiest start. For application-owned secrets,
`config` keeps everything under Helidon control. For OCI-native deployments, use the matching
principal-based module and set `authentication-method` explicitly.

The nice part is what happens afterwards: your code no longer cares how authentication was discovered.
It just injects `BasicAuthenticationDetailsProvider` in SE or MP, or looks it up with `Services.get`
when needed.
