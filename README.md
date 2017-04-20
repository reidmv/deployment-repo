# Deployment Repo

This is a Puppet workflow thought experiment.

What if branches didn't control deployments?

What if instead, a repository exists that is ONLY about defining deployments of Puppet code? A deployment is roughly synonymous with a Puppet environment. A tier is roughly synonymous with a customer environment. Every deployment has a tier, and defines a collection of Puppet code and modules to deploy.

The long-form of "tier" is Deployment Tier.

There don't need to be multiple branches. A file can describe each deployment, and there can be multiple files. All the files exist in the master branch of the "deployment repo".

Each yaml file in this repo represents a deployment. 

To make a new deployment, make a new file.

Pushing a change to the deployment-repo is just like changing configuration on the Puppet master, and triggers updates to the code-dir as necessary to reflect the updated configuration.

## Mental Model

The model for how to think about deployments is broken down below in terms of _things_ and _actions_.

### Things

#### Deployment

A _deployment_ is a deployed set of Puppet code. The long-form of _deployment_ is _Puppet code deployment_. A deployment definition consists of:

* A name. The deployment will be created as a Puppet environment of this name.
* The core Puppet code. A reference to a git repository and version. The contents of the repository should be laid out as a control-repo, or the root directory of a Puppet environment.
* A list of Puppet modules. References to git repositories and versions. Each module referenced will be deployed to the Puppet environment under `modules/<module name>`.
* A _deployment tier_ attribute. This is simply a string value and is used with Hiera in promotion workflows to tie each deployment to an appropriate Software Development Lifecycle (SDLC) phase.

Referencing a Puppet module in the deployment definition is not the only way to include a module in a deployment, though it is the recommended method. Modules may be included in a deployment one of three ways.

* Commit the module code directly to the control repo. E.g. in the `site/` directory, or the `modules/` directory.
* Include the module in the deployment. This is the preferred method of including a module.
* Include a Puppetfile in the control repo, and reference the module there. This method of including a module exists for backwards compatibility, and is not recommended.

Deployments may be created, deleted, updated, or duplicated (create a new deployment using an existing deployment as a template). Deployments must have unique names.

#### Deployment Tier

A _deployment tier_ is a string value attribute every deployment has which will be available to Puppet code as a top-scope variable `$::deployment_tier`. This variable may be used in Hiera hierarchies.

There may exist multiple deployments all at the same _deployment tier_. For example, users may create temporary testing deployments during development, all of which are considered part of the _development_ SDLC phase for the purposes of Hiera data.

In practice it is expected that deployment tiers like "production", "uat", and "staging" will be associated with a small number of long-lived deployments, while deployment tiers like "testing" and "development" may be associated with a fluxuating number of shorter-lived deployments.

#### Module

A _module_ is a Puppet module. However, in the context of a deployment a module is also an independently promotable unit of change.

Deployments are all about making changes, or promoting change. The fact that a module is independently promotable is very important. It means module change promotion rates for modules are decoupled from each other. One module (A) may promote a change from "test" to "staging" once a week, while another module (B) pushes changes through from "test" to "staging" once a day. Changes made to module (B) do not stack up behind changes to module (A) in a single queue, nor does one change advancing past another require git wizardry to accomplish. Each module has its own linear change history and any version of that module may be promoted to any deployment at any time.

#### Core

The _core_ of a deployment is similar to a module in that it is an independently promotable unit of change. The core has a distinction though in that there may only be one core in a deployment.

### Interesting Actions

Basic CRUD actions can be implied based on the thing descriptions above. Here I'll describe some of actions most interesting from a workflow perspective, and why they're interesting.

#### Create (deployment)

Create a new deployment from scratch. This is an unusual action to take and would typically only be done as part of setting up a promotion workflow for the first time.

#### Duplicate (deployment)

Create a new deployment based on an existing deployment as a template. This action may occur frequently as part of the development workflow. To create a new module feature a dev deployment might be created as a duplicate of production, and then a development version of module (A) promoted to it. This development deployment would be used by the developer to test and iterate on proposed changes to module (A) until the changes were ready to start promoting through the SDLC change pipeline.

#### Update (deployment)

An update to a deployment is most likely to occur immediately after duplicating a deployment. The reason for this is that outside of that one use case, the promote action is is likely to be preferred over update in a workflow.

Updating a deployment is literally just editing any part of its definition. Core, modules, deployment tier, or name.

#### Promote (module or core)

This workflow is all about promoting change. _Promote_ is the primary action by which change is propogated from one deployment to another.

To promote a change means to select core and/or module(s) from a deployment (A) and update another deployment (B) such that (B) is made to contain the same object(s) at the same version(s) as exist in (A).

An example promotion might be promoting the bigfix module version 1.2.0 from test to staging. Or, it might be promoting a change in core, 2e73137a, from staging into production.

Importantly, the rate at which changes are promoted for one object is independent of the promotion rate of all other objects.

#### Promote (deployment)

To facilitate faster or simpler workflows, it is possible to take a whole deployment (A) and replicate its core and modules to a deployment (B), fully replacing whatever core and module definitions (B) had previously. This constitutes promoting the deployment as a single unit of change.

#### Compare (deployment)

Because changes for core and modules are moving through the promotion pipeline (deployments) at independent rates, it is important to be able to see at a glance what the differences are between deployments. To see a list of what changes are in progress, and where each change is at in the deployment pipeline.

### Where the term "Deployment" comes from

The choice of the term "deployment" is inspired by the design of an old VMware product, vFabric Application Director. 

* [vFabric Application Director web interface docs](http://pubs.vmware.com/appdirector-5.2/index.jsp#com.vmware.appdirector5.2.using.doc/GUID-DE9B73D5-E6BE-4BDD-8F19-57244B044F1F.html)
* [vFabric Application Director managing deployments docs](http://pubs.vmware.com/appdirector-5.2/index.jsp#com.vmware.appdirector5.2.using.doc/GUID-F43C713F-F151-4D78-A1A3-2CC8740045C2.html)

![vFabric Application Director](http://pubs.vmware.com/appdirector-5.2/topic/com.vmware.appdirector5.2.using.doc/GUID-1FCB49B4-5FF4-4F8E-9F78-C3279DA43347-low.png)

The idea is that a deployment is just an instance of something. It is not unique. "This is my code deployment. There are many like it, but this one is mine."

A deployment can have lifecycle actions applied to it, such as modules promoted to it, or updates made to it, and it has a history. But in the end it is easily replicable and could even be considered disposable, recreatable.

A deployment is a named, parameterized instance of _something else_ (in this case Puppet code), includes lifecycle actions, and has a history.

## WHY?

Today there is no clear line of deliniation between version of code and deployment.

If everything is in the control repo, you can't deploy different apps to differrent deployment tiers at different rates. Every change goes into the same queue, and is in line BEHIND the change in front of it regardless of dependency.

If you de-couple a module from the control repo using the Puppetfile by setting `:branch => :control_branch`, deployment of that module has been 100% decoupled from teh control repo and you have two potential problems.

1. The final gate to deploying a change to the decoupled module is committing code to a branch in that module. This may be ok, or it may be a problem.
2. There is no single, versioned history of what permutation of different versions of Puppet code and modules was deployed to a given Deployment Tier at any given point in time. You can't easily reproduce a deployment.
3. It is difficult to make an atomic deployment involving more than one decoupled module.

This thought experiment is seeking a way to

* Make DEPLOYING something an intentional action separate from DEVELOPING it
* Provide a workflow for deploying different apps to different deployment tiers at different rates WHILE
* Providing a versioned, single source of truth about what deployments exist and which versions of decoupled modules have been deployed to them AND
* Providing a way to stage and make atomic deployments involving more than one decoupled module WITH
* A SIMPLE git workflow
