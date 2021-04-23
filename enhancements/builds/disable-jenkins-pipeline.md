---
title: disable-jenkins-pipeline
authors:
  - "@adambkaplan"
reviewers:
  - @gabemontero
  - Akram
approvers:
  - "@bparees"
  - Jessica Forrester
  - "@sbose78"
creation-date: 2021-04-21
last-updated: 2021-04-23
status: provisional
see-also: []
replaces: []
superseded-by: []
---

# Disable Jenkins Pipeline Strategy

**Note - this enhancement is provisional and is intended to solicit feedback**

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Operational readiness criteria is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

OpenShift 3.0 included a way to invoke a Jenkins pipeline from an OpenShift build.
The status of the build on the OpenShift cluster would reflect the state of the pipeline in Jenkins.
This was the first feature to support continuous integration/delivery processes on OpenShift.

As OpenShift and Kubernetes have evolved, so has cloud-native CI/CD.
Tekton and its downstream distribution - OpenShift Pipelines - provide a way of running CI/CD processes natively on Kubernetes.
The Jenkins pipeline strategy was officially deprecated in OpenShift 4.1, but at that time there was no meaningful replacement.
We now have that meaningful replacement with the general availability of OpenShift Pipelines.
It is time to disable OpenShift's direct integration with Jenkins.

## Motivation

Distrbuting and supporting Jenkins directly has always been challenging for OpenShift.
Jenkins is based in Java and is extended by a convoluted ecosystem of plugins, which are a constant source of bugs and CVEs.

Jenkins itself is not native to Kubernetes and does not use a Java runtime that has been optimized for cloud-native environments (i.e. Quarkus).
As a result, it has poor resilience when run on Kubernetes.
The Jenkins templates do not run Jenkins in a highly available mode, leaving its primary instance vulnerable to downtime from slow restarts.
The Jenkins container image is also very large - it is in fact the largest image in the OpenShift payload today.
Removing Jenkins from the OpenShift payload has been a longstanding goal that can reduce the time needed to pull the OCP payload at install/updgrade time.

Red Hat does not have a strong connection to the Jenkins community, and has marginal influence when it comes to requested features and patches.
Recently Red Hat tried to engage with a third party to develop an open source Kubernetes operator for Jenkins.
That effort unfortunately fell apart, leading Red Hat to fork the operator in an attempt to continue its development.
No other companies or individuals have contributed to the operator since, and its current form does not provide measurable value beyond what is currently available with the Jenkins templates.

With the upcoming GA release of OpenShift Pipelines, we would like to decouple Jenkins from OpenShift.
This would allow bug fixes in our Jenkins distribution to be released on its own cadence.
Customers can continue to deploy Jenkins on OpenShift via its ImageStream and associated templates.
In this fashion, Jenkins becomes just another application on OpenShift and does not retain any preferential status.

### Goals

* Disable the integration points between Jenkins and OpenShift.
* Move Jenkins out of the OCP Payload
* Document how JenkinsPipeline strategy builds can be migrated off of OpenShift.

### Non-Goals

* Migrate Jenkins users to OpenShift Pipelines/Tekton.
* Stop distributing the Jenkins image and our associated OpenShift plugins.

## Proposal

### User Stories

As a developer using Jenkins on OpenShift
I want to continue using my Jenkinsfiles
So that I can continue to use Jenkins for my CI/CD processes

As an OpenShift release engineer
I want to remove Jenkins from the OCP payload
So that I can reduce the size of the payload
And release Jenkins fixes on their own cadence

### Implementation Details/Notes/Constraints [optional]

Deprecation warnings for the Jenkins Pipeline Build Strategy will need to be updated in the next OpenShift release (OCP 4.N).
Since Jenkins integrations have been officially been officially deprecated for well over one year, these warnings need to be strengthened so customers are fully aware that all direct Jenkins integrations will be removed.
Like the use of other deprecated APIs, OpenShift must also alert users if a Jenkins pipeline build strategy is used in a Build or BuildConfig.

In the OCP 4.(N+1) release, builds which use the Jenkins pipeline build strategy should fail immediately.
All documentation related to Jenkins pipeline builds will need to be removed, and replaced with a statement that the Jenkins pipeline strategy has been disabled.
The updated documenation should include instructions on how to move a Jenkinsfile off of OpenShift.

Ideally, the deprecation warnings are strengthened in OCP 4.9.
Jenkins pipeline builds will then be disabled in OCP 4.10 (our next Extended User Support release).

#### Jenkins Pipeline Build Strategy

In OCP 4.N, an alert should be fired if a build is started with the Jenkins Pipeline build strategy.
Likewise, an alert should be fired if there is any BuildConfig which uses the Jenkins Pipeline strategy.
This data is already collected by openshift-state-metrics, therefore creating an alert is a matter of aggregating the right metrics.

In OCP 4.(N+1), the build controller should immediately fail a build which utilizes the Jenkins pipeline strategy with an appropriate failure message.

#### oc <new|start>-build

In OCP 4.N, `oc new-build` should print a warning message if the generated/referenced BuildConfig uses the Jenkins pipeline strategy.
`oc start-build` should print a warning message if a build was started with the Jenkins pipeline strategy.

In OCP 4.(N+1), `oc new-build` should not support the creation of new Builds with the Jenkins pipeline strategy.
`oc start-build` may continue to create builds referencing the Jenkins pipeline strategy, but these builds will ultimately fail.

#### Web Console

The Developer and Admin perspectives in the Web Console should not directly integrate with Jenkins beyond what is possible with an OpenShift API object.
User will continue to be able to deploy Jenkins on OpenShift by instantiating our provided Templates.

#### Documentation

Current Jenkins Pipeline strategy builds will need to add their Jenkinsfile defitions to Jenkins through another means.
Upstream Jenkins has guidance on how to [define a pipeline in source control](https://www.jenkins.io/doc/book/pipeline/getting-started/#defining-a-pipeline-in-scm), which should be the basis on how users should migrate their Jenkins pipelines off of OpenShift.
There are two migration scenarios which need to be considered:

- Migrating a Jenkinsfile defined in the user's git source.
- Migrating an inline Jenkinsfile defined in the user's BuildConfig.

#### Release Engineering

Jenkins is currently built and packaged via the OpenShift ART team.
This process will need to be migrated to CPaaS.

### Risks and Mitigations

**Risk: Not sufficient time to migrate off of Jenkins**

**Risk: Users will continue to start Jenkins pipeline builds**

## Design Details

### Open Questions [optional]

TODO

### Test Plan

**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?
- What additional testing is necessary to support managed OpenShift service-based offerings?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).

#### Removing a deprecated feature

- Announce deprecation and support policy of the existing feature
- Deprecate the feature

### Upgrade / Downgrade Strategy

If applicable, how will the component be upgraded and downgraded? Make sure this
is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to keep previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to make use of the enhancement?

Upgrade expectations:
- Each component should remain available for user requests and
  workloads during upgrades. Ensure the components leverage best practices in handling [voluntary disruption](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/). Any exception to this should be
  identified and discussed here.
- Micro version upgrades - users should be able to skip forward versions within a
  minor release stream without being required to pass through intermediate
  versions - i.e. `x.y.N->x.y.N+2` should work without requiring `x.y.N->x.y.N+1`
  as an intermediate step.
- Minor version upgrades - you only need to support `x.N->x.N+1` upgrade
  steps. So, for example, it is acceptable to require a user running 4.3 to
  upgrade to 4.5 with a `4.3->4.4` step followed by a `4.4->4.5` step.
- While an upgrade is in progress, new component versions should
  continue to operate correctly in concert with older component
  versions (aka "version skew"). For example, if a node is down, and
  an operator is rolling out a daemonset, the old and new daemonset
  pods must continue to work correctly even while the cluster remains
  in this partially upgraded state for some time.

Downgrade expectations:
- If an `N->N+1` upgrade fails mid-way through, or if the `N+1` cluster is
  misbehaving, it should be possible for the user to rollback to `N`. It is
  acceptable to require some documented manual steps in order to fully restore
  the downgraded cluster to its previous state. Examples of acceptable steps
  include:
  - Deleting any CVO-managed resources added by the new version. The
    CVO does not currently delete resources that no longer exist in
    the target version.

### Version Skew Strategy

How will the component handle version skew with other components?
What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- During an upgrade, we will always have skew among components, how will this impact your work?
- Does this enhancement involve coordinating behavior in the control plane and
  in the kubelet? How does an n-2 kubelet without this feature available behave
  when this feature is used?
- Will any other components on the node change? For example, changes to CSI, CRI
  or CNI may require updating that component before the kubelet.

## Implementation History

2020-04-23: Provisional enhancement draft

## Drawbacks

The idea is to find the best form of an argument why this enhancement should _not_ be implemented.

## Alternatives

Similar to the `Drawbacks` section the `Alternatives` section is used to
highlight and record other possible approaches to delivering the value proposed
by an enhancement.

## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new
subproject, repos requested, github details, and/or testing infrastructure.

Listing these here allows the community to get the process for these resources
started right away.
