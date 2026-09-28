---
title: AI-based access control
sidebar_label: AI-based access control
description: Applies organizational access control governance policy to each onboarded agent and tool.
sidebar_position: 9
---

An access control policy states who may call what. You write that policy in plain language, for a person
to read. A running agentic system needs the same policy as rules that a machine evaluates on each call.
AI-based access control (AIAC) performs the translation. It generates the Rego rules that OPA evaluates,
and it repeats the work each time a service, a role or the policy changes.

:::warning Alpha
This feature is an experiment. It lives in the separate repository
[rossoctl/aiac](https://github.com/rossoctl/aiac), and no Rossoctl installation contains it. The
behaviour, the interface and the set of deployed components will change. Evaluate it on a test cluster.
:::

## The problem that it solves

An agentic system does not make one call. Follow one request from a user:

1. A user invokes an agent.
2. That agent invokes a second agent.
3. The second agent invokes a tool.
4. The tool invokes a third agent.

Each hop is a separate decision to allow or to deny. The number of possible execution paths grows past
the point where a person can list them. Your policy document also changes. Nobody keeps the two in
agreement by hand.

Three problems follow:

- **The rules fall behind the platform.** A new service or a new role arrives with no matching rule,
  because no mechanism applies one.
- **No single document states the intent.** The knowledge of what a role may do spreads over many
  deployments. You cannot read the intent in one place.
- **Each path needs a decision in advance.** A gate answers in milliseconds. It cannot read a policy
  document at the moment of the call.

AIAC decides every path in advance, from one authoritative policy.

## How it operates

AIAC separates three layers. Each layer has one component and one responsibility.

| Layer | Component | What it does |
| --- | --- | --- |
| Policy management | AIAC agent | Translates the policy in plain language into OPA rules. |
| Policy decision | OPA | Evaluates the rules. Decides what the caller may access. |
| Policy enforcement | [AuthBridge](../../security/authbridge.md) | Intercepts the call. Exchanges the token. Holds no rule. |

AuthBridge wraps each workload that you secure with it — every agent, and each tool that you choose. The
wrap gives that workload a gate on both directions of its traffic. AuthBridge runs a chain of plugins on
the inbound traffic, and a second chain on the outbound traffic. Each chain contains the OPA plugin. The
name of the plugin is `opa`.

That plugin decides nothing on its own. It asks OPA, and OPA answers from the Rego rules that AIAC wrote.
When a rule denies the call, the plugin returns the status 403 with this body:

```json
{"error":"policy.forbidden","message":"policy denied","plugin":"opa"}
```

Three events start a translation.

| Event | What AIAC does |
| --- | --- |
| Keycloak registers a client | Onboards the new service. Computes its rules, inbound and outbound. |
| A role changes | Computes the rules for each service that the role reaches. |
| An operator ingests a policy document | Computes the difference against the rules that are current. |

For each event, the agent performs these steps:

1. It receives the event from NATS JetStream. Delivery is durable, so a restart of the agent loses no
   event.
2. It reads the roles, the services and the scopes from Keycloak.
3. It retrieves the part of the policy that applies to the event.
4. A model proposes the rules. A second pass of the model validates them against the policy.
5. The computation engine merges the proposed rules into the policy model that the store holds.
6. The policy writer renders the merged model as Rego into an `AuthorizationPolicy` custom resource, one
   resource for each agent.
7. `bundle-service` composes those resources into one bundle for each pod. The OPA plugin polls that
   bundle.

AIAC reports a conflict. It never resolves one. A role and a scope that carry both a grant and a
prohibition stop the request with the status 422 and a report of every conflict found. No rule changes.
This behaviour has no exception. AIAC applies no precedence and performs no merge, because a conflict is
a fault in the policy that a person must correct.

## Every decision happens before the call

AIAC calls a model when the policy or the platform changes. No model runs when a call arrives. The agent
reasons over each potential interaction on each potential execution path. It writes the result as Rego,
as constraint rules that OPA evaluates later.

Two results follow. AIAC adds nothing to the response time of a request, because OPA evaluates a rule and
calls no model. And you pay for a model on each change of policy, not on each call of an agent.

Your policy is the ground truth for each rule. A second pass of the model validates each proposed rule
against the policy that the agent retrieved for that event.

## How to enable it

You need a Kubernetes cluster with Keycloak, an AuthBridge installation whose OPA plugin is active, and
an endpoint that is compatible with the OpenAI chat interface. AIAC calls that endpoint for each
translation.

AIAC deploys into its own namespace, `aiac-system`, from the manifests in its own repository. For the
build of each image, the two Secrets, and the order to apply the manifests, follow the
[AIAC installation guide](https://github.com/rossoctl/aiac/blob/main/k8s/aiac-deployment-guide.md). That
guide holds the procedure. This page does not repeat it.

Those manifests deploy four components: the interface pod, the event broker, the policy model store and
the agent. The knowledge base that holds the policy has no manifest yet, so a cluster that you stand up
today has no route to ingest a policy document.

<!-- VERIFY: the RAG pod (ChromaDB, RAG Ingest Service, Policy Guardrails Agent) has no manifest under
k8s/ in rossoctl/aiac; PRD section 8 marks rag-statefulset.yaml pending. Update the component list above,
the third row of the trigger table, and the matching limit below when that manifest ships. -->

## What it does not do

- **It enforces nothing.** AuthBridge intercepts each call, and OPA decides it. AIAC writes the rules
  only.
- **It does not manage a user.** AIAC manages roles and scopes. Each rule names a role, so OPA resolves
  the entitlements of a user from the role of that user. AIAC drops an event about a user.
- **It does not resolve a conflict.** It reports each conflict and changes no rule.
- **It does not record the reason for a rule.** A rule holds a role, a scope and an effect. AIAC stores
  no reference to the sentence of the policy that justifies it.
- **It does not review the rules that a person added.** A review of the current entitlements, and a
  request for access from a user, are both planned. Neither one is present.
- **It does not replace the identity of a workload.** Keycloak remains the identity provider, and the
  platform continues to register each agent and each tool as a client.

## Try it

The [AIAC demonstration](https://github.com/rossoctl/aiac/blob/main/demo/assets/INSTALL.md) deploys one
agent and one tool, then drives the onboarding flow from end to end.

## Related pages

- [Identity and trust](../core/identity.md) describes how the identity of a workload becomes access, and
  where roles come from.
- [AuthBridge](../../security/authbridge.md) describes the layer that enforces the decision of OPA, and
  the path of one request.
- [Intent-based access](intent-based-access.md) asks a model about one action, at the moment of that
  action. This feature decides each action in advance, and calls no model on the path of a request.
