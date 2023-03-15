---
title: "Moving Auth Out of Apps and Into Envoy to Reduce Change Vectors"
date: 2023-03-15T07:44:37-04:00
draft: false
---

## TL;DR

Moving authn/authz out of applications and into Envoy filters reduces cost and risk and improves observability and resilience.

## Problem: Each app has an auth config

The initial design decision to put authentication and authorization in each app is so common that many just do it without any conscious thought.

In the pre-microservices world, where most/all of the components of the app were in the app, there was not a major downside to this.

In a microservices world, the authn/authz *decision accumulation* can lead to situations where it is hard, risky, and/or expensive to make changes and do the right thing.

### A change in the authentication or authorization requirements leads to updating each app

#### Sample C# code for in-app auth

In .NET 7, the configuration (when only one scheme is used) is much better than in previous versions. 

For authentication, one simply calls `AddAuthentication().AddJwtBearer()` and then adds an `Authentication` object in `appsettings.json`.

To require certain traits in the JWT, one can add an `AuthorizationPolicy` via the `AuthorizationPolicyBuilder`.

<div class="codeblock-label">Program.cs fragment</div>

```c#
WebApplicationBuilder builder = WebApplication.CreateBuilder(args);

// ...

builder.Services.AddAuthentication().AddJwtBearer();
builder.Services.AddAuthorization(options => { options.AddPolicy("user", policyBuilder => policyBuilder.RequireClaim("user")); });

// ...
```
<div class="codeblock-label">appsettings.json fragment</div>

```json
{
  "Authentication": {
    "Schemes": {
      "Bearer": {
        "Authority": "http://localhost:8180/realms/master",
        "ValidAudiences": [ "demo-app" ],
        "ValidIssuer": "http://localhost:8180/realms/master",
        "RequireHttpsMetadata": false
      }
    }
  }
}
```

An `Authorize` attribute is then added to the appropriate controller or method.

<div class="codeblock-label">Authorize attribute on controller method</div>

```c#
[Authorize]
[HttpGet("")]
public IEnumerable<WeatherForecast> Get()
```

This is quick to develop. If there are many applications that are all doing the same thing, the cost of *initial* implementation is nominal. Unfortunately, when this design decision accumulates over multiple applications, it can become hard to make changes without a service interruption or wasteful cost.

#### Time spent updating, building, testing, deploying

##### Issuer, Audience, and/or Authority update = time_to_update_1 * N

If the change is simply to the issuer (`iss`), the audience (`aud`) or the authority, the time spent updating, testing, building, and deploying will be a linear multiple of the number of applications that need updated. So, if you have 10 apps, then the time is 10x; if you have 100, it is 100x -- this is wasteful. 

If the old auth source is still available while the change is made, and if the apps can tolerate other apps in the ecosystem using a different auth setup, then the risk per deployment is on par with a normal rolling update to just one app.

If, however, the apps pass the JWT upstream as they interact with other services, then you will have an **outage**, (potentially) an **SLA violation**, and a situation where you cannot test the changes *until all apps have deployed*.

##### JWT property change = &Sigma;(variable_time_per_app)

If the change is to some trait within the JWT payload and your app cares about it, then the changes you need to make are to the startup `authz` code, and/or the controller/method attributes.  So, the cost is the time to identify each place in code that needs updated in each application, the time to make and test the changes, and the rollout time.

## Solution: Envoy Filters

[Envoy](https://www.envoyproxy.io/) is a high-performance FOSS (free open-source software) proxy.  It helps solve the networking and observability concerns that arise in microservice architectures.

It is an out-of-process solution to the problems described above. In a k8s (kubernetes) environment, it is normally deployed as a sidecar with the app in a pod. If a service mesh is used (e.g. [Istio](https://istio.io/)) is used, Envoy is often configured via yaml `ConfigMap` files that are pushed to the control plane.

It is possible to design and test with Envoy locally before testing the code in a k8s environment with a tool like [Skaffold](https://skaffold.dev/) or [Telepresence](https://www.telepresence.io/).  For an example app, please see [stand-sure/keycloak-local-dev](https://github.com/stand-sure/keycloak-local-dev).

### Authentication (`authn`)

If your app simply needs to verify that the caller has a valid JWT ([JSON Web Token](https://www.rfc-editor.org/rfc/rfc7519)), the JWT Authentication Envoy Filter is appropriate [see [envoy.filters.http.jwt_authn](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/jwt_authn/v3/config.proto)].

In the sample below,

1. The URI of a Keycloak server running locally is used as the Issuer, *viz.* 
     ```yaml
     issuer: http://localhost:8180/realms/master`
     ```
2. The Audience `demo-app` is required
     ```yaml
     audiences:
      - demo-app
     ```
3. The JWT is forwarded upstream by
     ```yaml
     forward: true
     ```
   allowing other services to use the value.

   *This can be useful as a migration strategy since apps that have code that checks the JWT do not need immediate changes.*
4. `payload_in_metadata` puts the token payload into the payload that can be used by other filters (including custom WASM and LUA script filters).
5. Certain Claim values in the JWT are copied to HTTP Headers. *This allows the app to simply grab the value*.

   In a .NET WebAPI, one just decorates the variable in the controller method with `FromHeader(Name = "x-headerName")`, e.g. 
     ```c#
     public IEnumerable<WeatherForecast> GetNoAuth([FromHeader(Name = "x-sub")] string? sub = null)
     ``` 
6. The signature of the JWT is validated using the `remote_jwks.http_uri.uri`, which references a cluster defined in the file [not shown in this article].
7. A match condition is defined to only check the JWT on certain paths.

<div class="codeblock-label">envoy-config.yaml fragment</div>

```yaml
          http_filters:
          - name: envoy.filters.http.jwt_authn
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.jwt_authn.v3.JwtAuthentication
              providers:
                provider1:
                  issuer: http://localhost:8180/realms/master
                  audiences:
                    - demo-app
                  forward: true
                  payload_in_metadata: jwt_payload
                  claim_to_headers:
                    - header_name: x-sub
                      claim_name: sub
                    - header_name: x-scope
                      claim_name: scope
                  remote_jwks:
                    http_uri:
                      uri: http://identity_cluster/realms/master/protocol/openid-connect/certs
                      cluster: identity_cluster
                      timeout: 5s
                    cache_duration:
                      seconds: 300
              rules:
                - match:
                    prefix: /WeatherForecast
                  requires:
                    provider_name: provider1
```

The advantages of this approach are:
1. It can be applied to multiple services in the environment, reducing development time and strengthening security for all services involved.
2. If the issuer, audience, or backend identity service change, the edit is simple, and the change can be applied without rebuilding and redeploying the apps.
3. It improves the DevOps IAC (infrastructure as code) maturity of the apps and environment.

### Authorization (`authz`)

[envoy.extensions.filters.http.ext_authz.v3.ExtAuthz](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/ext_authz/v3/ext_authz.proto) filters allow you to do more complex checks.

In the example below, an external authz gRPC service is listening at port 9191 on a particular address (you would use a service DNS name in k8s).

<div class="codeblock-label">envoy-config.yaml fragment</div>

```yaml
          - name: envoy.ext_authz
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthz
              with_request_body:
                max_request_bytes: 8192
                allow_partial_message: true
              failure_mode_allow: true
              grpc_service:
                google_grpc:
                  target_uri: 192.168.87.219:9191
                  stat_prefix: ext_authz
              transport_api_version: V3      
```

The [OPA Envoy Plugin](https://github.com/open-policy-agent/opa-envoy-plugin) is a version of the [Open Policy Agent](https://www.openpolicyagent.org/) that is designed for `ExtAuthz` usage.

OPA uses a language called `Rego` to answer questions that are asked by sending data to an endpoint.  If you are testing locally, a command like this can be used: 

```bash
curl localhost:8181/v1/data -d @- <<< $(cat sample-input.json)| jq
```

The sample below is for illustration only -- it is **not** production-ready. In the code below:
1. The HTTP request in imported from the input sent by the Envoy filter;
2. `token` splits and decodes the `Authorization` (there are other JWT methods available in OPA that can verify the token; see https://www.openpolicyagent.org/docs/latest/policy-reference/#builtin-tokens-iojwtdecode);
3. `is_token_valid` evaluates if the action is allowed and if the token is valid (`iat` and `exp` are properties in the decoded token -- if `nbf` is present, it should be checked as well);
4. `action_allowed` checks if the method verb and request path match the rule (there can be multiple rules).

<div class="codeblock-label">envoy.authz.rego</div>

```golang
package envoy.authz

import input.attributes.request.http as http_request

default allow = false

token = {"payload": payload} {
    [_, encoded] := split(http_request.headers.authorization, " ")
    [_, payload, _] := io.jwt.decode(encoded)
}

allow {
    action_allowed
    is_token_valid
}

is_token_valid {
  now := time.now_ns() / 1000000000
  token.payload.iat <= now
  now < token.payload.exp
}

action_allowed {
  http_request.method == "GET"
  glob.match("/WeatherForecast*", [], http_request.path)
}

```

The steps to figure out your own Rego are:
1. Get and decode a JWT.
   - Both Insomnia and PostMan are able to fetch JWTs
   - There are FOSS tools for decoding the JWT, e.g. https://github.com/mike-engel/jwt-cli
   - Use the properties of interest in the policy
   - CAUTION: JWTs expire -- you will need to make sure the token lifetime is long enough for you to work (or get a newer token periodically)
2. Go over to https://play.openpolicyagent.org/ to design/test.
3. Paste your token into the following input:

      ```json
         {
           "attributes": {
             "request": {
               "http": {
                 "headers": {
                   "authorization": "Bearer PASTE_TOKEN_HERE"
                },
                "method": "GET",
                "path": "/WeatherForecast/NoAuth"
              }
            }
           }
        }
      ```
   - The JSON pattern here matches the `import` statement above and is controlled by the Envoy filter pipeline.
   - The output should look something like the following fragment:
     ```json
      {
          "action_allowed": true,
          "allow": true,
          "is_token_valid": true,
          "token": {
              "payload": {
                  "aud": [
                      "demo-app",
                      "..."
                  ],
                  "azp": "demo-app",
                  "exp": 1678920645,
                  "iat": 1678899045,
                  "iss": "http://localhost:8180/realms/master",
                  "scope": "email user profile openid",
                  "sid": "f6e7981a-d519-437b-ba1b-bf9efbd2ce4d",
                  "sub": "dddabca3-c344-48e3-9b98-cc492f8baa78",
                  "typ": "Bearer"
              }
          }
      }
     ```
   
   - The outputs of each of the rules is a key.  The filter uses the value of `allow` to evaluate the outcome.
4. Once you have a working rule, you can use it in your local OPA service and test things end-to-end.
   - The `envoy` part of the OPA version is important -- the non-envoy versions often do not have the gRPC endpoint.
   - The HTTP UI (and REST API endpoints) for the OPA server are on port 8181.
   - Metrics and health checks are available on 8282.
   - gRPC is on 9191, which is the port listed in the Envoy filter markup above.
   - `decision_logs.console=true` is a great help for troubleshooting as you develop as it allows you to enter a command like 
       ```shell
       docker compose logs opa --follow
       ```
     and see a fragment like the following in the logs:
       ```json
     {"result":{"envoy":{"authz":{"action_allowed":true,"allow":false}}}}

       ```
   - Save your Rego file in the `policies` directory (note: there are ways to dynamically set policies in OPA in production)

<div class="codeblock-label">docker-compose.yaml fragment</div>

```yaml
  opa:
    image: openpolicyagent/opa:0.50.0-envoy-1
    platform: linux/amd64
    container_name: opa
    restart: always
    ports:
      - "8181:8181"
      - "8282:8282"
      - "9191:9191"
    volumes:
      - ./policies:/policies
    command:
      - "run"
      - "--server"
      - "--log-format=json"
      - "--set=decision_logs.console=true"
      - "--diagnostic-addr=0.0.0.0:8282"
      - "--set=plugin.envoy_ext_authz_grpc.addr=:9191"
      - "--set=plugins.envoy_ext_authz_grpc.query=data.envoy.authz.allow"
      - "/policies"
```

## Summary

Moving authn/authz out of applications and into Envoy filters is straight-forward. Doing so, reduces development time, and improves overall security and resilience.
