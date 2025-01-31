class: center, middle, inverse

# Securing Microservices with Open Policy Agent and Envoy

.right[[**Shane Soh**](https://www.linkedin.com/in/shane-soh/)]
.right[Senior Software Engineer]
.right[Centre for Strategic Infocomm Technologies]
---

# Background

### Who are we? 
**Centre for Strategic Infocomm Technologies (CSIT)** is an agency in the Ministry of Defence that builds technologies
to safeguard the national security interests of Singapore. 

Our team builds **platforms and infrastructure** that support a wide range of mission-critical operations, such as in
counter-terrorism and computer network defence.

---

# Background

### This brings about unique challenges...

* Classified *low-trust* environments where complex access control and audit logging are required for internal APIs 

* Increasingly more microservices but still have many monolithic legacy systems running on VMs

* Need API security that is easy to implement and to reason with

Thus **Open Policy Agent (OPA)** and **Envoy**

---

# What is OPA?

Open Policy Agent is a lightweight general-purpose **policy engine** that lets you specify policy as code and use APIs
to offload policy decision-making from your software.

.center.image-60[![](./images/opa-service.svg)]

---

# What is Envoy

.center.image-40[![](https://www.envoyproxy.io/docs/envoy/latest/_static/envoy-logo.png)]

Envoy is an open source edge and service proxy designed for cloud-native applications.

For our purpose, Envoy is used to delegate authorization decisions to OPA, allowing/denying requests to the service
based on OPA's policy decisions.

---

# Putting Them Together

.center[![](./images/diagram.svg)]

* Teams are expected to deploy Envoy and OPA alongside their service

* All incoming requests first go to Envoy which checks with OPA whether each request should be allowed through

* Service can simply assume any request that arrives is authorized as per OPA policies

---

# Sample Project

.footnote[[github.com/shanesoh/envoy-opa-compose](https://github.com/shanesoh/envoy-opa-compose)]

`docker-compose.yml`
```yaml
...
  envoy:
    build: ./compose/envoy
    ports:
      - "8080:80"
    volumes:
      - ./envoy.yaml:/config/envoy.yaml
    environment:
      - DEBUG_LEVEL=info
      - SERVICE_NAME=app  # should match name of underlying service
      - SERVICE_PORT=80

  app:
    image: kennethreitz/httpbin:latest
...
```

* Built Envoy image does environment variable substitution for `envoy.yaml`
    * To simplify adoption as Envoy config can be rather complex
* httpbin as a mock service

---

# Sample Project

.footnote[[github.com/shanesoh/envoy-opa-compose](https://github.com/shanesoh/envoy-opa-compose)]

`docker-compose.yml`
```yaml
...
  opa:
    image: openpolicyagent/opa:0.26.0-envoy
    volumes:
      - ./policy.rego:/config/policy.rego
    command:
      - "run"
      - "--log-level=debug"
      - "--log-format=json-pretty"
      - "--server"
      - "--set=plugins.envoy_ext_authz_grpc.path=envoy/authz/allow"
      - "--set=decision_logs.console=true"
      - "/config/policy.rego"
...
```

* OPA-Envoy image uses [opa-envoy-plugin](https://github.com/open-policy-agent/opa-envoy-plugin) which extends OPA with
  a gRPC server that implements the Envoy External Authorization (`ext_authz`) API. 

---

# Sample Project

.footnote[[github.com/shanesoh/envoy-opa-compose](https://github.com/shanesoh/envoy-opa-compose)]

`policy.rego`
```c
package envoy.authz

import input.attributes.request.http as http_request

default allow = false

allow = response {
  http_request.method == "GET"
  response := {
      "allowed": true,
      "headers": {"X-Auth-User": "1234"}
  }
}
```
* Policy decision can be boolean or an object (as in this case)
* Toy policy to only allow GET requests and add additional header for `X-Auth-User`

---

# Sample Project

.footnote[[github.com/shanesoh/envoy-opa-compose](https://github.com/shanesoh/envoy-opa-compose)]

```shell
# This is allowed
$ curl -X GET http://localhost:8080/anything
{
  ...
  "headers": {
	...
    "X-Auth-User": "1234", 
	...
  }, 
  ...
  "method": "GET", 
}

# This gets denied
$ curl -X POST http://localhost:8080/anything
```

---

# Sample Project

.footnote[[github.com/shanesoh/envoy-opa-compose](https://github.com/shanesoh/envoy-opa-compose)]

In our actual setup we make authorization decisions primarily using OAuth2/OIDC access tokens

```c
# ...truncated...

default allow = false

token = payload {
  # Access token in `Authorization: Bearer <token>` header
  [_, encoded] := split(http_request.headers.authorization, " ")
  [_, payload, _] := io.jwt.decode(encoded)
}

allow = response {
  # Check access token contains `read` permission to `myapp`
  token.resource_access["myapp"].roles[_] == "read"

  response := {
      "allowed": true,
      "headers": {
        "X-User-Id": token.user_id,
        "X-Given-Name": token.given_name
      }
  }
}
```

---

# Why Do We Like OPA?

--

* Manage policy as code
    * Written in Rego
    * Declarative; relatively easy to read, write and test

--

* Decouple policy decision-making from policy enforcement
    * Authorization logic written outside of service instead of it peppered all over the code
        * Separation of concerns: Developers focus on writing application logic assuming authorization is handled
    * Policies can be reloaded without restarting underlying service
    * Facilitates discoverability, reuse and governance

--

* Centralised management APIs
    * Policy distribution via bundles APIs
        * Especially universal policies
    * Collection of telemetry 
        * e.g. Decision logs (that are clearly separated from application logs)

---

# Why Do We Like Envoy?

* Filters to implement common functionalities
    * External Authorization (`ext_authz`) to OPA
    * But also JWT Authentication (`jwt_authn`) filter to validate access tokens

--

* Purpose-built for sidecar usage
    * Completely agnostic to upstream service
        * With OPA, useful for tacking on access control to services that don't have them
    * Containerised for Docker or k8s
        * At the same time easy to run on the host for legacy applications
        * Possible to auto inject as k8s sidecars 

--

* Future-proofing
    * Currently east-west traffic goes through API gateway which provides functionalities like traffic control and
      logging centrally
    * Eases potential adoption of service mesh: Move functionalities into Envoy sidecars and use Istio as control plane

---
class: left, middle, inverse

# Thank you.

<i class="fab fa-linkedin"></i> [shane-soh](https://linkedin.com/in/shane-soh) /
<i class="fab fa-github"></i> [shanesoh](https://github.com/shanesoh)

### We're Hiring! [csit.gov.sg](https://www.csit.gov.sg)
#### Software Engineer (Infrastructure) [go.gov.sg/csit-swe-infra](https://go.gov.sg/csit-swe-infra)
