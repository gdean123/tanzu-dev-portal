---
date: '2021-02-16'
lastmod: '2021-02-26'
subsection: Application Lifecycle
team:
- John Harris
title: Application Lifecycle
topics:
- Kubernetes
weight: 17
oldPath: "/content/guides/kubernetes/app-lifecycle.md"
aliases:
- "/guides/kubernetes/app-lifecycle"
level1: Managing and Operating Kubernetes
level2: Preparing and Deploying Kubernetes Workloads
---

The scripts and systems used in the CI/CD pipelines to deploy and update
applications are limited by the Kubernetes resources they can manage.  In many
cases this may be perfectly sufficient.  An update to the image in a Deployment
spec may be all that is required to perform an update of an application, for
example.

However, this model is often insufficient
when dealing with workloads that are stateful, distributed and/or complex.
For example, an application that stores data in a relational database may require
a database schema update in coordination with an upgrade to the application itself.
This kind of coordination is awkward for a pipeline to orchestrate and usually
requires a series of manual operations to perform an application update or
upgrade.

## Operators

A Kubernetes operator is a piece of software that runs in your cluster and
manages the deployment and lifecycle management on your behalf.  In this
example you use a custom resource - an extension to the Kubernetes
API - to define the essential characteristics of your application.  The operator
responds by creating the various Kubernetes resources on your behalf.  When
an app upgrade is needed, the pipeline need only update the custom resource that
represents your application.  The operator contains the logic that allows it to
conduct the upgrade in response.  In the relational database-backed example, it
may put the application into a maintenance mode, initiate a backup of the
database, apply the schema update to the database, upgrade the application
version and restore it to normal functioning mode.

The advantage of using Kubernetes operators is that you can reduce the human
toil from otherwise labor-intensive operations and perform these operations
reliably.  The drawback is that you have another piece of complex software to
develop and maintain.

## Popular Tooling & Approaches

### Kubebuilder

[Kubebuilder](https://github.com/kubernetes-sigs/kubebuilder) is an SDK for
building custom Kubernetes controllers and operators.  It is a CLI you can use
to stamp out boilerplate and scaffolded Go code that is common to all such projects.
The code generated by Kubebuilder includes a Dockerfile, deployment manifests and a
Makefile that facilitates local development.  In addition to the base project
codebase, Kubebuilder can be used to add new APIs as CRDs along with their
corresponding controllers.

Pros:

* is [well documented](https://book.kubebuilder.io/)
* can include admission webhooks for your CRDs
* has experimental support for
  [plugins](https://github.com/kubernetes-sigs/kubebuilder/tree/master/plugins)
  to build operators using alternative patterns
* is maintained and controlled by upstream Kubernetes SIG

Cons:

* requires the Go programming language

### Metacontroller

![metacontroller](images/metacontroller.png)

[Metacontroller](https://github.com/GoogleCloudPlatform/metacontroller) or, the
more actively maintained fork [metac](https://github.com/AmitKumarDas/metac), is
a cluster add-on that runs in your cluster and takes care of the common
operations all controllers and operators must do.  Whereas Kubebuilder stamps
out boilerplate code for these common operations, metacontroller abstracts
those operations away into a separate workload.  Metacontroller manages all
interactions with the Kubernetes API.  The custom controller's logic is
developed in what metacontroller calls the "lambda controller".  The presence
and nature of the lambda controller is defined in a metacontroller custom
resource, for example the "Composite Controller".  In the case of a Composite
Controller, the "parent resource" is the resource that is watched and which
triggers reconciliation.  When a change occurs in the parent resource,
metacontroller calls the lambda controller and passes that parent resource
object as JSON.  The lambda controller responds with the "child resource" as
JSON and metacontroller makes the change to the actual child resource in the API.

Pros:

* lambda controllers can be developed in any language that can expose an HTTP
  endpoint to metacontroller and use JSON
* smaller codebase for your controller logic
* fast to get started - great for prototyping
* can run multiple controllers behind a single metacontroller instance

Cons:

* your controller has the additional dependency and operational overhead of
  metacontroller running in the cluster

### Operator SDK

[Operator SDK](https://github.com/operator-framework/operator-sdk) is a project
originally started at CoreOS that is similar to Kubebuilder in that it uses a
CLI to generate boilerplate and scaffolded code for a new project.  It is a
component of Red Hat's [Operator Framework](https://github.com/operator-framework)
which could make it attractive if you are using other components of that toolkit.
Due to the overlap between Kubebuilder and the Go-based operator support in
Operator SDK, those features will be [merged into
Kubebuilder](https://github.com/kubernetes-sigs/kubebuilder/blob/master/designs/integrating-kubebuilder-and-osdk.md)
in the future.  Operator SDK should only be considered where value can be
derived from the Ansible or Helm features.

Pros:

* allows you to develop operators in Go, Ansible or Helm
* offers integrations with lifecycle manager from the Operator Framework

Cons:

* is managed by Red Hat for their ecosystem so may not fit your needs well if
  not using OpenShift
* the Go-based operator features will be merged into Kubebuilder