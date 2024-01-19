---
title: Expose Open Policy Agent/Gatekeeper Constraint Violations with Prometheus and Grafana
tags: ['kubernetes','open policy agent', 'opa','prometheus','grafana']
status: draft
---

# Expose Open Policy Agent/Gatekeeper Constraint Violations for Kubernetes Applications with Prometheus and Grafana

*TL;DR: In this [blog post](https://itnext.io/expose-open-policy-agent-gatekeeper-constraint-violations-with-prometheus-and-grafana-6b7ac92ea07f), we talk about a solution which gives platform users a succinct view about which Gatekeeper constraints are violated by using Prometheus & Grafana.*

**[Andy Knapp](https://www.linkedin.com/in/andy-knapp-6a72b6108/) and [Murat Celep](https://www.linkedin.com/in/muratcelep/) has worked together on this blog post.**

Application teams that just start to use Kubernetes might find it a bit difficult to get into it as Kubernetes is a quiet complex & large ecosystem (see [CNCF ecosystem landscape](https://landscape.cncf.io/)). Moreover, although Kubernetes is starting to mature, it's still being developed very actively and it keeps getting new features at a faster pace than many other enterprise software out there. On top of that, Kubernetes platform deployments into the rest of a company's ecosystem (Authenticaton, Authorization, Security, Network,storage) are tailored specifically for each company due to the integration requirements. So even for a seasoned Kubernetes expert there are usually many things to consider to deploy an application in a way that it fulfills security, resiliency, performance requirements. How can you assure that applications that run on Kubernetes keep fulfilling those requirements?

## Enter OPA/Gatekeeper

[Open Policy Agent](https://www.openpolicyagent.org/) (OPA) and its Kubernetes targeting component [Gatekeeper](https://github.com/open-policy-agent/gatekeeper) gives you means to enforce policies on Kubernetes clusters. What we mean by policies here, is a formal definition of rules & best practices & behavior that you want to see in your company's Kubernetes clusters. When using OPA, you use a Domain Specific Language called [Rego](https://www.openpolicyagent.org/docs/latest/policy-language/) to write policies. By doing this, you leave no room for misinterpretations that would occur if you tried to explain a policy in free text on your company's internal wiki.

Moreover, when using Gatekeeper, different policies can have different enforcement actions. There might be certain policies that are treated as **MUST** whereas other policies are treated as **NICE-TO-HAVE**. A **MUST** policy will stop a Kubernetes resource being admitted onto a cluster and a **NICE-TO-HAVE** policy will only cause warning messages which should be noted by platform users.

In this blog post, we talk about about how you can:

- Apply example OPA policies, so-called Gatekeeper constraints, to K8S clusters
- Expose prometheus metrics from Gatekeeper constraint violations
- Create a Grafana dashboard to display key information about violations

If you want to read more about enforcing policies in Kubernetes, check out [this article](https://itnext.io/enforcing-policies-in-kubernetes-c0f6192bd5ca).

## Design

![System design](https://raw.githubusercontent.com/mcelep/opa-scorecard/master/system_logical.png)

The goal of the system we put together is to provide insights to developers and platform users insights about OPA constraints that their application might be violating in a given namespace. We use Grafana for creating an example dashboard. Grafana fetches data it needs for creating the dashboard from Prometheus. We've written a small Go program - depicted as 'Constraint Violation Prometheus Exporter' in the diagram above - to query the Kubernetes API for constraint violations and expose data in Prometheus format.
Gatekeeper/OPA is used in [Audit](https://open-policy-agent.github.io/gatekeeper/website/docs/audit) mode for our setup, we don't leverage Gatekeeper's capability to deny K8S resources that don't fulfill policy expectations.

### OPA Constraints

Every company has its own set of requirements for applications running on Kubernetes. You might have heard about the *production readiness checklist* concept (in a nutshell, you want to create a checklist of items for your platform users to apply before they deploy an an application in production). You want to have your own *production readiness checklist* based on Rego and these links below might give you a good starting point for creating your own list:

- [Application Readiness Checklist on Tanzu Developer Center](https://tanzu.vmware.com/developer/guides/kubernetes/app-enhancements-checklist/)
- [Production best practices on learnk8s.io](https://learnk8s.io/production-best-practices)
- [Kubernetes in Production: Readiness Checklist and Best Practices on replex.io](https://www.replex.io/blog/kubernetes-in-production-readiness-checklist-and-best-practices)

Bear in mind that, you will need to create a OPA policies off of your production readiness checklist and you might not be able to cover all of the concerns you might have in your checklist using OPA/Rego. The goal is to focus on things that are easy to extract based on Kubernetes resources definitions, e.g. number of replicas for a K8S Deployment.

For our blog post, we will be using the open source project [gatekeeper-library](https://github.com/open-policy-agent/gatekeeper-library) which contains a good set of example constraints. Moreover, the project structure is quite helpful in the sense of providing an example of how you can manage OPA constraints for your company: Rego language which is used for creating OPA policies should be unit tested thoroughly and in  [src folder](https://github.com/open-policy-agent/gatekeeper-library/tree/master/src/general), you can find  pure rego files and unit tests. [The library folder](https://github.com/open-policy-agent/gatekeeper-library/tree/master/library/general) finally contains the Gatekeeper constraint templates that are created out of the rego files in the src folder. Additionally, there's an example constraint for each template together with some target data that would result in both positive and negative results for the constraint. Rego based policies can get quite complex, so in our view it's a must to have Rego unit tests which cover both **happy & unhappy** paths. We'd recommend to go ahead and fork this project and remove and add policies that represent your company's requirements by following the overall project structure. With this approach you achieve compliance as code that can be easily applied to various environments.

As mentioned earlier, there might be certain constraints which you don't want to directly enforce (MUST vs NICE-TO-HAVE): e.g. on a dev cluster you might not want to enforce **>1 replicas**, or before enforcing a specific constraint you might want to give platform users enough time to take the necessary precautions (as opposed to blocking their changes immediately). You control this behavious using [`enforcementAction`](https://open-policy-agent.github.io/gatekeeper/website/docs/violations#dry-run-enforcement-action). By default, `enforcementAction` is set to `deny` which is what we would describe as a **MUST** condition.
In our example, we will install all constraints with the **NICE-TO-HAVE** condition using `enforcementAction: dryrun` property. This will make sure that we don't directly impact any workload running on K8S clusters (we could also use [`enforcementAction: warn`](https://open-policy-agent.github.io/gatekeeper/website/docs/violations#warn-enforcement-action) for this scenario).

### Prometheus Exporter

We decided to use [Prometheus](https://prometheus.io/) and [Grafana](https://grafana.com/) for gathering constraint violation metrics and displaying them, as these are good and popular open source tools.

For exporting/emitting Prometheus metrics, we've written a small program in Golang that uses the [prometheus golang library](https://github.com/prometheus/client_golang). This program uses the Kubernetes API to discover all constraints applied to the cluster and export certain metrics.

Here's an example metric:

```
opa_scorecard_constraint_violations{kind="K8sAllowedRepos",name="repo-is-openpolicyagent",violating_kind="Pod",violating_name="utils",violating_namespace="default",violation_enforcement="dryrun",violation_msg="container <utils> has an invalid image repo <mcelep/swiss-army-knife>, allowed repos are [\"openpolicyagent\"]"} 1
```

Labels are used to represent each constraint violation and we will be using these labels later in the Grafana dashboard.

The Prometheus exporter program listens on tcp port `9141` by default and exposes metrics on path `/metrics`. It can run locally on your development box as long as you have a valid Kubernetes configuration in your home folder (i.e. if you can run kubectl and have the right permissions). When running on the cluster a `incluster` parameter is passed in so that it knows where to look up for the cluster credentials. Exporter program connects to Kubernetes API every 10 seconds to scrape data from Kubernetes API.

We've used [this](https://medium.com/teamzerolabs/15-steps-to-write-an-application-prometheus-exporter-in-go-9746b4520e26) blog post as the base for the code.
