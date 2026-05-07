---
v: 3

title: "Agent Directory"
abbrev: AD
docname: draft-jimenez-agent-directory-latest

category: std
stream: IETF
area: Applications
workgroup: DAWN

venue:
  mail: TBD
  github: jaimejim/draft-jimenez-agent-directory

author:
- name: Jaime Jimenez
  org: Ericsson
  email: jaime@iki.fi

normative:
  RFC3986:
  RFC6750:
  RFC8615:
  RFC9110:
  RFC9176:
  RFC9457:

informative:
  RFC6690:
  RFC6763:
  I-D.pioli-agent-discovery:
  I-D.mp-agntcy-ads:
  I-D.narajala-ans:
  I-D.hood-independent-agtp:
  I-D.zyyhl-agent-networks-framework:
  I-D.cui-dns-native-agent-naming-resolution:
  I-D.mozleywilliams-dnsop-dnsaid:
  I-D.ietf-wimse-arch:
  I-D.ietf-wimse-identifier:
  MCP-Registry:
    title: "Model Context Protocol Registry"
    target: https://github.com/modelcontextprotocol/registry
    date: 2025
  ANP-Discovery:
    title: "ANP Agent Discovery Protocol Specification"
    target: https://agentnetworkprotocol.com/en/specs/08-anp-agent-discovery-protocol-specification/
    date: 2024
  MCP:
    title: "Model Context Protocol Specification"
    target: https://spec.modelcontextprotocol.io/
    date: 2025
  A2A:
    title: "Agent-to-Agent Protocol"
    target: https://a2aprotocol.ai/
    date: 2025
  Azure-API-Center:
    title: "Agent registry in Azure API Center"
    target: https://learn.microsoft.com/en-us/azure/api-center/agent-to-agent-overview
    date: 2026
  ERC-8004:
    title: "ERC-8004: Trustless Agents"
    target: https://eips.ethereum.org/EIPS/eip-8004
    date: 2026
  AgentRegistry:
    title: "agentregistry"
    target: https://aregistry.ai/
    date: 2026

--- abstract

This document defines the Agent Directory (AD), a service where agents register their identity, capabilities, and reachable endpoints and where clients discover them by capability. The AD adapts the Constrained RESTful Environments (CoRE) Resource Directory (RFC 9176) from constrained IoT devices to software agents, reusing its registration, lookup, and soft-state lifecycle model. The AD uses HTTP and JSON. MCP and A2A define how agents interact; this document defines how they are found.

--- middle

# Introduction {#introduction}

The CoRE Resource Directory {{RFC9176}} defines a service for discovering resources hosted by other web servers, particularly in networks where direct discovery is impractical. Endpoints register links describing their resources, and clients look them up later. Registrations are soft state with a configurable lifetime. The RD provides both resource-level and endpoint-level lookup.

A similar discovery problem arises for software agents. LLM-based assistants, tool-calling agents, and multi-agent systems are deployed without fixed addresses and speak different interaction protocols (MCP {{MCP}}, A2A {{A2A}}, gRPC). Direct enumeration of candidate endpoints does not scale beyond a single administrative domain.

Individual agents may publish their own metadata at well-known endpoints (such as the A2A Agent Card at `/.well-known/agent-card.json`). However, per-agent metadata requires knowing the agent's URL before you can discover what it offers. The AD aggregates per-agent metadata into a queryable service. Clients discover agents by capability without prior knowledge of their addresses.

Like the CoRE RD, the AD has a small footprint: it can run as a cloud service, as a sidecar alongside an agent orchestrator, on an edge gateway, or on a constrained device. Deployment profiles range from cloud services to constrained devices colocated with sensors and actuators.

The lookup interface is a small set of HTTP GET requests with a fixed set of query parameters. Clients include both conventional applications, which use structured filters, and LLM-based clients, which can additionally match on the natural-language descriptions and tags carried in registrations. The interface is small enough to be described as a tool in an MCP server or function-calling schema.

This document specifies the Agent Directory (AD). The core mapping to CoRE RD concepts is:

| CoRE RD Concept | AD Equivalent |
|---|---|
| Endpoint (EP) | Agent |
| Resource link | Capability / tool description |
| Registration resource | Agent registration resource |
| Endpoint lookup | Lookup, agent view (`view=agent`) |
| Resource lookup | Lookup, capability view (`view=cap`) |
| Lifetime (lt) | Registration lifetime |

Agents register with the AD by sending a POST request with a JSON document describing their capabilities. Registrations are soft state and expire unless refreshed.

## Terminology {#terminology}

{::boilerplate bcp14-tagged-bcp14}

This specification makes use of the following terminology:

{:vspace}
Agent:
: An autonomous or semi-autonomous software entity that can initiate and receive interactions, expose capabilities (tools, skills, APIs), and optionally collaborate with other agents.

Agent Directory (AD):
: An HTTP service that stores agent registrations and provides discovery and lookup interfaces.

Capability:
: A discrete function, tool, or skill that an agent exposes. Capabilities are described as entries within an agent's registration.

Registration:
: The act of an agent (or a commissioning tool acting on its behalf) submitting its identity, capabilities, and endpoint information to the AD.

Registration Resource:
: The resource stored at the AD as a result of a successful registration. Identified by the URI returned in the Location header of the creation response.

Interaction Protocol:
: The application-level protocol through which an agent can be reached for task execution, such as MCP {{MCP}}, A2A {{A2A}}, or gRPC. HTTP is the underlying transport for most interaction protocols but is not itself an interaction protocol in this context.

# Architecture {#architecture}

{{fig-arch}} shows the AD architecture.

~~~~
                Registration         Lookup
                 Interface         Interface
     +-------+       |                 |
     | Agent |---    |                 |
     +-------+   --- |                 |
                   --|-    +------+    |
     +-------+       | ----|      |    |     +--------+
     | Agent |-------|-----|  AD  |----|-----| Client |
     +-------+       | ----|      |    |     +--------+
                   --|-    +------+    |
     +-------+   --- |                 |
     |  CT   |---    |                 |
     +-------+       |                 |
~~~~
{: #fig-arch title='Agent Directory Architecture' artwork-align="center"}

Agents register their capabilities and endpoints with the AD. Clients (which may themselves be agents) query the AD's lookup interface to discover agents matching desired criteria. A Commissioning Tool (CT) may register agents on their behalf, for example when a platform operator manages a fleet of agents. How a CT proves authority to act on behalf of an agent is deployment-specific and out of scope for this document.

## Principles

The AD operates on the following principles:

* The AD stores information that could otherwise only be obtained by directly querying each agent's metadata endpoint.
* Registrations are soft state. The registering agent (or its CT) MUST periodically refresh each registration before its lifetime expires.
* The AD MUST NOT permit modification of a registration by any entity other than the original registrant or an authorized CT.
* The AD does not proxy agent interactions; it only provides discovery.

## Content Model {#content-model}

An AD contains zero or more agent registrations. Each registration has:

* An agent name ("agent", a Unicode string) unique within the AD instance.
* A registration base URI ("base") {{RFC3986}}, typically the agent's reachable address.
* A lifetime ("lt") in seconds.
* A registration resource location within the AD ("href").
* Optionally, a version string ("version").
* Optionally, a vendor string ("vendor").
* Zero or more capabilities, each described as a JSON object with at minimum a name and a type.

## Federation {#federation}

In deployments that span multiple administrative domains or geographic regions, multiple AD instances may operate independently. Federation allows these instances to share registration information so that a client querying one AD can discover agents registered at another.

This document does not specify a federation protocol. Possible approaches include periodic synchronization between peered AD instances, a publish/subscribe notification mechanism, and a hierarchical model where a root AD aggregates entries from subordinate instances. The soft-state model bounds staleness: federated entries carry the original lifetime and expire without explicit deletion if synchronization lapses.

# AD Discovery {#discovery}

## Well-Known URI

An AD MUST be discoverable at the well-known URI:

    /.well-known/ad

A GET request to this URI returns a JSON document describing the AD's interfaces. The response object contains the following fields:

{:vspace}
registration:
: (string, REQUIRED) Path to the registration endpoint.

lookup:
: (string, REQUIRED) Path to the lookup endpoint.

max_count:
: (integer, REQUIRED) Maximum value the AD accepts for the `count` pagination parameter.

Example:

    GET /.well-known/ad HTTP/1.1
    Host: directory.example.com

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "registration": "/ad/r",
      "lookup": "/ad/l",
      "max_count": 100
    }

## DNS-SD

An AD MAY advertise itself on a local network via DNS-SD {{RFC6763}} using the service name `_ad._tcp`. This provides zero-configuration discovery of a local AD, which is useful on networks where agents are colocated with sensors and actuators.

Several DNS-based mechanisms have been proposed in the DAWN working group for wide-area agent discovery. DNS-AID {{I-D.mozleywilliams-dnsop-dnsaid}} publishes agent metadata under `_agents` subdomains using SVCB records. DN-ANR {{I-D.cui-dns-native-agent-naming-resolution}} defines a resolution layer using FQDNs and SVCB/HTTPS records. These mechanisms handle naming and resolution: mapping an agent name to a network location. The AD handles the layer above: finding agents by capability without prior knowledge of their names. DNS resolves the AD's hostname; the AD carries the capability metadata that DNS cannot efficiently carry.

# Registration {#registration}

## Registration Request

An agent registers by sending a POST request to the AD's registration endpoint. All HTTP interactions with the AD follow the semantics defined in {{RFC9110}}. The request body is a JSON object containing the agent's metadata and capabilities.

    POST /ad/r?agent=summarizer-v2 HTTP/1.1
    Host: directory.example.com
    Content-Type: application/json

    {
      "base": "https://agents.example.com/summarizer-v2",
      "description": "Summarizes documents and extracts named entities",
      "protocols": ["a2a"],
      "capabilities": [
        {
          "name": "summarize",
          "type": "tool",
          "description": "Summarize a document or text passage",
          "input_schema": {
            "type": "object",
            "properties": {
              "text": {"type": "string"},
              "max_length": {"type": "integer"}
            },
            "required": ["text"]
          }
        },
        {
          "name": "extract_entities",
          "type": "tool",
          "description": "Extract named entities from text"
        }
      ],
      "version": "2.1.0",
      "vendor": "Example Corp",
      "identity": "https://registry.example.com/agents/summarizer-v2",
      "identity_type": "aip"
    }

    HTTP/1.1 201 Created
    Location: /ad/r/4521

The query parameters are:

{:vspace}
agent:
: REQUIRED. Agent name. MUST be unique within the AD instance. SHOULD be short and meaningful; the AD MAY enforce a maximum length.

lt:
: Lifetime in seconds (OPTIONAL). Default: 86400 (24 hours). Range: 60 to 4294967295 (2^32-1, matching {{RFC9176}}). The AD SHOULD enforce a maximum lifetime; a recommended default maximum is 604800 seconds (7 days).

The body JSON object contains:

{:vspace}
base:
: (string, REQUIRED) The agent's reachable base URI {{RFC3986}}.

description:
: (string, OPTIONAL) A short human-readable description of what the agent does. SHOULD be one or two sentences. Returned in lookup responses to help clients evaluate agents without additional round-trips.

protocols:
: (array of strings, OPTIONAL) Interaction protocols supported. The following values are defined by this document: "mcp" (Model Context Protocol), "a2a" (Agent-to-Agent Protocol), "grpc" (gRPC). Implementations SHOULD use lowercase short names and MAY append a version (e.g., "mcp/1.0"). A future version of this document may define an IANA registry for protocol identifiers.

capabilities:
: (array of objects, OPTIONAL) The agent's capabilities. Each object MUST have "name" (string) and "type" (string) fields. The "type" field indicates the kind of capability (see {{capability-types}}). Additional fields such as "description" (string), "tags" (array of strings), "input_schema" (object), and "output_schema" (object) are OPTIONAL. When present, input_schema and output_schema SHOULD be JSON Schema objects. Capability names MUST be unique within a registration.

version:
: (string, OPTIONAL) The agent's version identifier.

vendor:
: (string, OPTIONAL) The organization or individual that provides the agent.

identity:
: (string, OPTIONAL) A URI pointing to the agent's identity metadata document. This document describes the agent's verified identity, trust posture, owner binding, or credentials in a structured format. The AD stores and returns this URI without interpreting its contents.

identity_type:
: (string, OPTIONAL) The schema or format of the identity document referenced by `identity`. Values include "aip" (Agent Identity Profile), "oidc" (OpenID Connect Discovery), "wimse" (WIMSE workload identity), and "did" (Decentralized Identifier Document). Clients that recognize the type can dereference and validate the identity document; clients that do not recognize the type SHOULD ignore both fields.

When present, the `identity` field lets a client verify an agent beyond trusting the AD. For example, a client discovering a financial-analysis agent can fetch its AIP document to check owner binding, attestation state, and credential lifetime before invoking the agent. The AD stores the URI and does not interpret it; trust decisions remain the client's. Clients MUST NOT treat the presence of an `identity` field as proof of identity without verifying the referenced document.

### Capability Types {#capability-types}

The "type" field in a capability object indicates the kind of function the agent exposes. The following initial values are defined:

{:vspace}
tool:
: A discrete, stateless function that accepts structured input and returns structured output.

skill:
: A higher-level, potentially multi-step behavior that may involve internal reasoning, planning, or coordination. Unlike a tool, a skill may maintain conversational state.

resource:
: A data source or content provider that the agent can expose for reading.

prompt:
: A reusable prompt template that the agent can execute with supplied parameters.

This list is extensible. Implementations MAY use additional type values. The remaining types mirror those defined by MCP {{MCP}}.

## Registration Semantics

Registration is idempotent on the agent name; a POST with the same agent name acts as an upsert. This follows the RFC 9176 pattern of POST-to-collection for registration rather than PUT to a canonical agent URL. The AD stores the `agent` value from the query parameter as part of the registration resource and includes it in all response representations.

* If no registration exists for the agent name, a new registration resource is created. The AD returns 201 (Created) with a Location header pointing to the registration resource.
* If a registration already exists for the agent name and the request comes from the same authenticated entity, the registration is replaced with the new content. The AD returns 200 (OK) with the Location header of the existing registration resource.
* If a registration already exists for the agent name but is owned by a different authenticated entity, the AD returns 409 (Conflict).

The response body for both 201 and 200 is empty. Clients that need the full representation SHOULD send a GET request to the URI in the Location header. The AD MAY grant a lifetime shorter than requested; the granted lifetime MUST be indicated in the response via a `lt` field in a JSON body or via the Location header's associated resource.

## Reading a Registration {#registration-read}

An agent or client retrieves a single registration by sending a GET request to the registration resource.

    GET /ad/r/4521 HTTP/1.1
    Host: directory.example.com

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "agent": "summarizer-v2",
      "base": "https://agents.example.com/summarizer-v2",
      "protocols": ["a2a"],
      "capabilities": [
        {
          "name": "summarize",
          "type": "tool",
          "description": "Summarize a document or text passage",
          "input_schema": {
            "type": "object",
            "properties": {
              "text": {"type": "string"},
              "max_length": {"type": "integer"}
            },
            "required": ["text"]
          }
        },
        {
          "name": "extract_entities",
          "type": "tool",
          "description": "Extract named entities from text"
        }
      ],
      "version": "2.1.0",
      "vendor": "Example Corp",
      "href": "/ad/r/4521"
    }

The response includes the full registration content, including capability details (description, schemas) that are omitted from lookup results with `view=agent`. If the registration resource does not exist, the AD MUST return 404 (Not Found).

## Registration Update {#registration-update}

An agent refreshes or updates its registration by sending a POST request to its registration resource (the URI returned in the Location header at creation time).

    POST /ad/r/4521 HTTP/1.1
    Host: directory.example.com

    HTTP/1.1 204 No Content

An empty POST resets the lifetime. The agent MAY include query parameters (lt) to update those values, or a JSON body to replace the capabilities. If the registration has already expired, the AD MUST return 404 (Not Found); the agent must re-register via the registration endpoint.

An agent may update the lifetime via the `lt` query parameter:

    POST /ad/r/4521?lt=3600 HTTP/1.1
    Host: directory.example.com

    HTTP/1.1 204 No Content

If the registration has already expired, the update fails:

    POST /ad/r/4521 HTTP/1.1
    Host: directory.example.com

    HTTP/1.1 404 Not Found
    Content-Type: application/problem+json

    {
      "type": "https://ad.example.com/errors/registration-expired",
      "title": "Registration has expired",
      "status": 404
    }

## Registration Removal {#registration-removal}

An agent removes its registration by sending a DELETE request to its registration resource.

    DELETE /ad/r/4521 HTTP/1.1
    Host: directory.example.com

    HTTP/1.1 204 No Content

The AD MUST automatically remove a registration when its lifetime elapses without a refresh.

# Lookup {#lookup}

The AD provides a single unified lookup endpoint. The `view` query parameter controls how results are grouped:

* `view=agent` (default): returns a list of registered agents matching the query, with capability summaries.
* `view=cap`: returns a list of individual capabilities across all matching agents.

## Agent View {#agent-view}

    GET /ad/l HTTP/1.1
    Host: directory.example.com
    Accept: application/json

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "agents": [
        {
          "agent": "summarizer-v2",
          "base": "https://agents.example.com/summarizer-v2",
          "description": "Summarizes documents and extracts named entities",
          "protocols": ["a2a"],
          "capabilities": [
            {"name": "summarize", "type": "tool"},
            {"name": "extract_entities", "type": "tool"}
          ],
          "href": "/ad/r/4521"
        },
        {
          "agent": "citation-finder",
          "base": "https://agents.example.com/citation-finder",
          "description": "Finds and verifies academic citations",
          "protocols": ["mcp"],
          "capabilities": [
            {"name": "find_citations", "type": "tool"},
            {"name": "check_references", "type": "tool"}
          ],
          "href": "/ad/r/4522"
        }
      ]
    }

Agent view returns capability summary objects (name and type only). Full capability details (description, schemas) are omitted to keep lookup responses compact. Clients that need the full registration SHOULD retrieve the individual registration resource (GET /ad/r/{id}). This two-step pattern mirrors the RD's separation between endpoint lookup and resource lookup.

## Capability View {#capability-view}

Capability view returns individual capabilities with their owning agent's information. Clients use this view to find specific tools or skills across all registered agents.

    GET /ad/l?cap_name=summarize&view=cap HTTP/1.1
    Host: directory.example.com
    Accept: application/json

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "capabilities": [
        {
          "name": "summarize",
          "type": "tool",
          "description": "Summarize a document or text passage",
          "agent": "summarizer-v2",
          "base": "https://agents.example.com/summarizer-v2",
          "protocols": ["a2a"],
          "href": "/ad/r/4521"
        },
        {
          "name": "summarize_meeting",
          "type": "tool",
          "description": "Summarize meeting transcripts",
          "agent": "meeting-assistant",
          "base": "https://agents.example.com/meeting-bot",
          "protocols": ["mcp"],
          "href": "/ad/r/4523"
        }
      ]
    }

Each capability object in the response includes: "name", "type", "description" (if registered), the owning agent's "agent", "base", "protocols", and "href" (the registration resource URI). Clients can use "href" to retrieve the full registration.

## Query Parameters {#lookup-params}

All query parameters are OPTIONAL and act as filters. When multiple filter parameters are present, the AD MUST AND them: only entries matching all specified filters are returned. A request with no filter parameters returns all registrations visible to the requesting client.

{:vspace}
agent:
: Filter by agent name. A trailing asterisk (*) matches any suffix (e.g., `agent=sum*` matches "summarizer" and "summary-bot"). Without an asterisk, the value MUST match exactly. Wildcard matching is only supported for the `agent` and `cap_name` parameters.

protocol:
: Filter by supported interaction protocol. Exact match.

cap_name:
: Filter by capability name. Trailing asterisk (*) for prefix match; exact match otherwise.

cap_type:
: Filter by capability type (e.g., "tool", "skill"). Exact match.

tag:
: Filter by capability tag. Returns agents (or capabilities) that have at least one capability carrying the specified tag. Exact match. In agent view, the full agent with all its capabilities is returned if any capability matches.

view:
: Result grouping. "agent" (default) groups results by agent; "cap" groups results by individual capability. If omitted, the AD MUST use "agent".

page:
: Page number (zero-based) for pagination. Default: 0.

count:
: Maximum number of results per page. Default: 100. The AD MAY enforce a lower maximum and MUST document its limit in the well-known response.

# Error Responses {#errors}

When the AD cannot fulfill a request, it MUST return an appropriate HTTP status code and SHOULD include a problem details object as defined in {{RFC9457}}. The `type` URIs shown below are illustrative; this document does not register them.

    HTTP/1.1 409 Conflict
    Content-Type: application/problem+json

    {
      "type": "https://ad.example.com/errors/agent-name-taken",
      "title": "Agent name already registered",
      "detail": "Agent name already registered by another entity.",
      "status": 409
    }

The following error conditions are defined:

{:vspace}
400 Bad Request:
: The request body is malformed or missing required fields.

401 Unauthorized:
: The request lacks valid authentication credentials.

403 Forbidden:
: The authenticated entity is not authorized for the requested operation.

404 Not Found:
: The referenced registration resource does not exist.

409 Conflict:
: The agent name is already registered by a different entity.

429 Too Many Requests:
: The client has exceeded the rate limit.

# Security Policies {#security-policies}

The security model is derived from {{Section 7 of RFC9176}} and adapted for agents. Policies for capability attestation and cross-domain trust are out of scope for this document.

## Agent Name and Registration Ownership

The AD MUST ensure that a registering agent is authorized to use the given agent name. The default policy is "First Come First Remembered" as described in {{Section 7.5 of RFC9176}}: accept registrations for any agent name not currently active, and only accept updates from the same authenticated entity. Whether the AD should return 409 (Conflict) or 403 (Forbidden) when a different entity tries to claim an existing name is an open design question; this document uses 409 to signal a naming conflict rather than an authorization failure.

## Capability Confidentiality

The AD MUST enforce access control on lookup results when the operator policy designates some registrations as restricted. Not every client is entitled to see every registration.

## Capability Vetting

The AD SHOULD restrict which capability types and names an agent may claim, based on operator policy. For example, registration of an agent as a "financial-analysis" tool can be limited to agents presenting the corresponding credentials. The policy mechanism is deployment-specific and out of scope for this document.

# Security Considerations {#security-considerations}

## Transport Security

All communication with the AD MUST be protected using TLS.

## Authentication

Agents MUST authenticate when registering. The AD MUST support OAuth 2.0 bearer tokens {{!RFC6750}} or mutual TLS (mTLS). API keys are acceptable for development but do not satisfy the authentication requirement for production use.

Registration requires a verified agent identity: the AD must know who is registering before it can enforce ownership of the registration resource. Bearer tokens prove authorization but not workload identity. For production deployments, the AD SHOULD accept workload identity credentials as defined by the WIMSE working group {{I-D.ietf-wimse-arch}}. The WIMSE architecture treats agents as workloads and defines a `wimse:` URI scheme {{I-D.ietf-wimse-identifier}} that can serve as the agent's authenticated identity at registration time. This provides a cryptographically verifiable binding between the registrant and the registration.

If an agent's credentials are compromised, the attacker can modify the registration (including the `base` URI) until the credentials are revoked. Deployments SHOULD keep credential lifetimes comparable to registration lifetimes, and SHOULD log registration changes for audit.

## Authorization

The AD MUST enforce that only the original registrant (or an authorized CT) can modify or delete a registration. The AD SHOULD implement rate limiting on both registration and lookup endpoints, and MUST enforce limits on registration size and number of capabilities per agent.

## Privacy

Agent metadata can reveal organizational structure: a lookup response listing all agents discloses which capabilities an organization has deployed. The AD SHOULD return only those registrations that the requester is authorized to see. The interaction between scoped responses and federation is out of scope for this document.

## Adversarial Metadata

Capability descriptions and tags are free-form text that LLM-based clients may use as input to their decisions. A malicious registrant can craft descriptions that bias those decisions toward invoking the wrong agent. A malicious registrant can also set `base` to a victim's endpoint, redirecting traffic intended for the registered agent.

Clients SHOULD treat registration metadata as untrusted input. Clients MAY verify a registration by fetching the agent's own metadata at `base` and comparing it with the AD response before invocation.

# IANA Considerations {#iana-considerations}

## Well-Known URI Registration

This document requests registration of the following well-known URI {{RFC8615}}:

{:vspace}
URI suffix:
: ad

Change controller:
: IETF

Specification document:
: \[RFC-XXXX\]

## Media Type

This document uses `application/json` for all request and response bodies and `application/problem+json` {{RFC9457}} for error responses.

--- back

# Relationship to Other Work {#related-work}

## CoRE Resource Directory (RFC 9176)

The AD is a direct adaptation of the CoRE RD. The registration, update, removal, and lookup interfaces follow the same patterns. The differences reflect the shift from CoAP and CoRE Link Format {{RFC6690}} to HTTP and JSON: structured capability objects replace flat link attributes, and interaction protocol is advertised as a first-class field.

## Per-Agent Metadata Endpoints

A2A {{A2A}} serves an Agent Card at `/.well-known/agent-card.json`; MCP {{MCP}} exchanges server metadata during initialization. Both describe a single agent and both require the client to already know the agent's URL.

The AD complements these by aggregating metadata from many agents into a queryable registry. Clients perform capability-based search without prior knowledge of agent addresses. Once the AD returns a matching agent's base URI, the client MAY fetch the agent's own metadata endpoint for protocol-specific details before initiating interaction.

## MCP Registry

The Model Context Protocol project maintains a public registry {{MCP-Registry}} where MCP servers are catalogued for discovery by MCP-compatible clients. The registry and the AD address overlapping concerns from different ends: the MCP Registry is scoped to MCP servers and curated by a single project; the AD is protocol-agnostic (MCP, A2A, gRPC, and others are all describable) and is specified here as an open interface that any operator can implement. An agent registered in the MCP Registry can also appear in an AD instance without conflict; the two are complementary rather than substitutable.

## Agent Transfer Protocol (AGTP)

{{I-D.hood-independent-agtp}} proposes an agent-native transport protocol over TCP/TLS or QUIC, with agent identity, session semantics, and authorization expressed at the wire level rather than layered on HTTP. The AD takes the opposite approach, reusing HTTP and JSON as a substrate. The two are not mutually exclusive: an AD instance could register agents reachable over AGTP in the same way it registers agents reachable over MCP, A2A, or gRPC, by carrying the relevant protocol identifier in the `protocols` field.

## Agent Directory Service (ADS)

{{I-D.mp-agntcy-ads}} specifies a system with goals similar to the AD.

ADS takes a fully decentralized approach: content-addressed identifiers (CIDs) for records, a libp2p Kad-DHT for routing, OCI-compliant storage (ORAS/zot) for retrieval, and gRPC for the client interface. It defines OASF, a hierarchical skill taxonomy with 15 top-level categories and 100+ specific skills, used for capability-based routing through the DHT. This provides federation and cryptographic integrity at the cost of requiring DHT peers, OCI registries, and gRPC infrastructure.

The AD makes the opposite trade-off: a single HTTP/JSON service with no additional infrastructure. Federation is not built in (see {{federation}}) and there is no content-addressed integrity model. ADS targets decentralized deployments where no single operator is trusted; the AD targets deployments where a managed directory is acceptable. Interoperability between the two (an AD instance indexing ADS records, or vice versa) is not specified in this document.

## Agent Registration and Discovery Protocol (ARDP)

{{I-D.pioli-agent-discovery}} defines a similar registration and discovery service with a custom `agent:` identifier scheme. The AD differs by reusing standard HTTP URIs for agent identity and by grounding the design in the RD model.

## DNS-Based Agent Discovery

{{I-D.narajala-ans}} (ANS) proposes DNS-based agent discovery with PKI certificates. The AD operates at the application layer and does not require DNS infrastructure changes.

DN-ANR {{I-D.cui-dns-native-agent-naming-resolution}} defines a thin DNS resolution layer using FQDNs and SVCB records, explicitly leaving capability-based discovery to an upper layer. The AD can serve as that upper layer. DNS-AID {{I-D.mozleywilliams-dnsop-dnsaid}} places agent metadata in DNS zones for intra-organization discovery; the AD provides cross-organization lookup. The two approaches are complementary.

## Platform-Specific and On-Chain Registries

General-purpose service registries (Consul, etcd, Kubernetes service discovery) solve service location but do not model agent capabilities, interaction protocols, or soft-state registration semantics. The AD addresses the agent-specific aspects that these systems lack.

Azure API Center {{Azure-API-Center}} provides a centralized agent registry within the Azure ecosystem. The agentregistry project {{AgentRegistry}} offers a SaaS registry with a CLI. ERC-8004 {{ERC-8004}} defines on-chain agent registries on Ethereum.

These systems are each tied to a single provider: a cloud vendor, a SaaS operator, or a blockchain. The AD is specified by this document and can be operated by anyone; clients authenticate to the instance they use.

## Agent Network Protocol (ANP)

ANP's Agent Discovery Protocol {{ANP-Discovery}} defines active discovery via `/.well-known/agent-descriptions` and passive discovery where agents register with a search service. The AD's registration model corresponds to ANP's passive mode but specifies the registration API concretely. ANP uses JSON-LD and DIDs; the AD uses plain JSON and HTTP URIs. The broader ANP framework is described in {{I-D.zyyhl-agent-networks-framework}}.

## Agent Identity Profiles

The AD's `identity` and `identity_type` fields provide an extension point for linking registrations to external identity metadata. The Agent Identity Profile (AIP) is one such format, defining a JSON document that describes an agent's owner binding, capabilities, attestation state, trust posture, and credential lifecycle. Other formats include OpenID Connect Discovery documents and WIMSE workload identity endpoints. The AD does not mandate any particular identity schema; it provides the pointer so that clients can verify agents through whatever trust framework their deployment requires.

# Examples {#examples}

## Discovery Flow {#discovery-steps}

Discovering an agent through the AD involves four steps: locating the directory, discovering its interfaces, querying for a matching capability, and contacting the agent directly.

~~~~
 Client                          AD                         Agent
   |                              |                            |
   |  1. Obtain AD URL            |                            |
   |  (DNS-SD/config/.well-known)  |                            |
   |                              |                            |
   |  2. GET /.well-known/ad ---->|                            |
   |  <--- lookup interface URL   |                            |
   |                              |                            |
   |  3. GET /ad/l?               |                            |
   |     cap_name=summarize       |                            |
   |     &view=cap -------------->|                            |
   |  <--- matching agents + base |                            |
   |                              |                            |
   |  4. Interact with agent using protocol from lookup -----> |
   |  <--- Response ------------------------------------------ |
~~~~
{: #fig-discovery-steps title='Discovery Flow' artwork-align="center"}

The steps in detail:

Step 1: Obtain the AD URL.

The client already knows the URL of an Agent Directory.
This may have been learned through a DNS-SD browse for
`_ad._tcp`, a `.well-known/ad` query to a known host,
or simply through static configuration.

    https://ad.example.com

Step 2: Discover the AD's lookup interface.

The client queries the well-known endpoint to learn the available
interfaces.

    GET https://ad.example.com/.well-known/ad HTTP/1.1

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "registration": "/ad/r",
      "lookup": "/ad/l",
      "max_count": 100
    }

In some deployments the lookup interface path is already known
or follows a convention, in which case this step can be skipped.

Step 3: Search for an agent with the desired capability.

The client queries the lookup interface with `view=cap`.

    GET https://ad.example.com/ad/l?cap_name=summarize&view=cap
    HTTP/1.1

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "capabilities": [
        {
          "name": "summarize",
          "type": "tool",
          "description": "Summarize a document or text passage",
          "agent": "summarizer-v2",
          "base": "https://agents.example.com/summarizer-v2",
          "protocols": ["a2a"],
          "href": "/ad/r/4521"
        }
      ]
    }

The response includes the agent's base URI and supported
protocols, giving the client enough information to contact the
agent directly.

Step 4: Interact with the agent.

The lookup response indicated that `summarizer-v2` is reachable
over A2A. The client fetches the agent's A2A Agent Card to
obtain the full task interface before invocation.

    GET https://agents.example.com/summarizer-v2/.well-known/agent-card.json
    HTTP/1.1

    HTTP/1.1 200 OK
    Content-Type: application/json

    { ... A2A Agent Card ... }

The subsequent task invocation is defined by the interaction
protocol (A2A in this case, MCP or gRPC in others) and is out
of scope for this document.

## Enterprise Agent Portfolio

A Publisher at `example.com` operates three agents behind a single AD.

Step 1: Agents register with the AD.

    POST /ad/r?agent=ticket-classifier HTTP/1.1
    Host: ad.example.com
    Content-Type: application/json

    {
      "base": "https://agents.example.com/ticket-classifier",
      "description": "Classifies incoming support tickets.",
      "protocols": ["mcp"],
      "capabilities": [
        {"name": "classify_ticket", "type": "tool"},
        {"name": "suggest_priority", "type": "tool"}
      ],
      "vendor": "Example Corp"
    }

    HTTP/1.1 201 Created
    Location: /ad/r/201

    POST /ad/r?agent=knowledge-lookup HTTP/1.1
    Host: ad.example.com
    Content-Type: application/json

    {
      "base": "https://agents.example.com/kb",
      "description": "Searches internal knowledge base.",
      "protocols": ["mcp"],
      "capabilities": [
        {"name": "search_kb", "type": "tool", "tags": ["nlp", "search"]}
      ],
      "vendor": "Example Corp"
    }

    HTTP/1.1 201 Created
    Location: /ad/r/202

    POST /ad/r?agent=order-router HTTP/1.1
    Host: ad.example.com
    Content-Type: application/json

    {
      "base": "https://agents.example.com/order-router",
      "description": "Routes orders to fulfillment systems.",
      "protocols": ["a2a"],
      "capabilities": [
        {"name": "route_order", "type": "tool"}
      ],
      "vendor": "Example Corp"
    }

    HTTP/1.1 201 Created
    Location: /ad/r/203

Step 2: A client queries for MCP agents.

    GET /ad/l?protocol=mcp HTTP/1.1
    Host: ad.example.com
    Accept: application/json

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "agents": [
        {
          "agent": "ticket-classifier",
          "base": "https://agents.example.com/ticket-classifier",
          "description": "Classifies incoming support tickets.",
          "protocols": ["mcp"],
          "capabilities": [
            {"name": "classify_ticket", "type": "tool"},
            {"name": "suggest_priority", "type": "tool"}
          ],
          "href": "/ad/r/201"
        },
        {
          "agent": "knowledge-lookup",
          "base": "https://agents.example.com/kb",
          "description": "Searches internal knowledge base.",
          "protocols": ["mcp"],
          "capabilities": [
            {"name": "search_kb", "type": "tool"}
          ],
          "href": "/ad/r/202"
        }
      ]
    }

## Filtered Lookup with Pagination

A client enumerates MCP tool capabilities in pages of two.

    GET /ad/l?protocol=mcp&cap_type=tool&view=cap&count=2&page=0 HTTP/1.1
    Host: ad.example.com
    Accept: application/json

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "capabilities": [
        {
          "name": "classify_ticket",
          "type": "tool",
          "agent": "ticket-classifier",
          "base": "https://agents.example.com/ticket-classifier",
          "protocols": ["mcp"],
          "href": "/ad/r/201"
        },
        {
          "name": "suggest_priority",
          "type": "tool",
          "agent": "ticket-classifier",
          "base": "https://agents.example.com/ticket-classifier",
          "protocols": ["mcp"],
          "href": "/ad/r/201"
        }
      ]
    }

    GET /ad/l?protocol=mcp&cap_type=tool&view=cap&count=2&page=1 HTTP/1.1
    Host: ad.example.com

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "capabilities": [
        {
          "name": "search_kb",
          "type": "tool",
          "agent": "knowledge-lookup",
          "base": "https://agents.example.com/kb",
          "protocols": ["mcp"],
          "href": "/ad/r/202"
        }
      ]
    }

## Registration Conflict

A second entity attempts to register an agent under an already-claimed name.

    POST /ad/r?agent=ticket-classifier HTTP/1.1
    Host: ad.example.com
    Authorization: Bearer <token-of-other-entity>
    Content-Type: application/json

    {
      "base": "https://attacker.example.org/ticket-classifier",
      "protocols": ["mcp"],
      "capabilities": [{"name": "classify_ticket", "type": "tool"}]
    }

    HTTP/1.1 409 Conflict
    Content-Type: application/problem+json

    {
      "type": "https://ad.example.com/errors/agent-name-taken",
      "title": "Agent name already registered",
      "detail": "Agent name 'ticket-classifier' already registered.",
      "status": 409
    }

# Tooling and Integration {#tooling}

The AD uses `application/json` for all request and response bodies (and `application/problem+json` for errors). No custom media types or content negotiation is required.

The HTTP/JSON interface is usable from command-line tools (`curl` and a JSON parser), shell scripts, and CI/CD pipelines without specialized client libraries. The lookup interface can also be exposed as an MCP tool, so that an MCP-compatible client performs agent discovery through its existing tool-calling mechanism.

# Implementation Status

{::boilerplate rfc7942info}

## ad.jaime.win

Organization:
: Ericsson

Implementation:
: [https://ad.jaime.win](https://ad.jaime.win)

Description:
: A public AD instance deployed as a Cloudflare Worker with a Durable Object for persistent storage. Implements the registration, lookup, and soft-state lifecycle interfaces defined in this document. Source code is available at the GitHub repository linked from the deployment.

Level of Maturity:
: Prototype

Coverage:
: All features specified in this document: well-known URI discovery, agent registration (create, update, replace, delete), agent lookup with filtering and pagination, capability lookup with filtering and pagination, soft-state expiry, and RFC 9457 error responses.

Version Compatibility:
: draft-jimenez-agent-directory-00

Licensing:
: MIT

Contact:
: Jaime Jimenez (jaime@iki.fi)

# Acknowledgments

The AD design is directly based on the CoRE Resource Directory {{RFC9176}} by Christian Amsüss, Zach Shelby, Michael Koster, Carsten Bormann, and Peter van der Stok.

# Mapping from CoRE RD to AD {#mapping}

The following table summarizes the conceptual mapping between CoRE RD concepts and their AD equivalents.

| CoRE RD (RFC 9176) | AD | Notes |
|---|---|---|
| Endpoint (EP) | Agent | Software entity that registers |
| Resource link | Capability | Structured JSON instead of link-format |
| /.well-known/core | /.well-known/ad | Discovery entry point |
| Registration (POST /rd) | Registration (POST /ad/r) | Same soft-state model |
| Registration resource | Registration resource | Same lifecycle |
| Lifetime (lt) | Lifetime (lt) | Same semantics, different default |
| Endpoint name (ep) | Agent name (agent) | Same uniqueness constraint |
| Resource lookup | Lookup view=cap (GET /ad/l?view=cap) | Structured query instead of link-format filter |
| Endpoint lookup | Lookup view=agent (GET /ad/l) | Same pattern, unified endpoint |
| Commissioning Tool (CT) | Commissioning Tool (CT) | Third-party registration |
| application/link-format | application/json | Representation format |
