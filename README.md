# Deployment Workflow

## Problem Description

Suppose RG Bank (a hypothetical entity) uses Puppet to manage their Business Operations application infrastructure. This infrastructure consists of baseline configuration and 11 individual applications, some single-tier apps, some multi-tier apps. These applications are developed on RG Bank owned and managed infrastructure called Lab. Two separate Managed Service Providers (MSPs) then consume RG Bank's Puppet code to deploy and manage the apps.

RG Bank has XX deployment tiers, or environments, which changes are pushed to in order as part of the deployment process. These deployment tiers are:

1. development
2. test
3. prestage
4. stage
5. production

Each of the 11 applications RG Bank manages have their own Software Development Lifecycle (SDLC) cadence. Changes to each app are first deployed to development. Those changes are then promoted to test, to prestage, and so on until reaching production.

Different applications are developed and deployed at widely varying cadences. Baseline configuration changes to services like ntp and dns move very quickly through development all the way to production. Changes to one of the apps, FTO, move more slowly though. Changes to this app might be left in the "test" deployment tier for a long time to bake-in, and make sure there are no problems after long periods of operation.

Frequently, there is a need for changes to baseline configuration or rapidly-developing apps to come up from behind an existing change already baking in the test deployment tier, and pass through on to prestage or production.

Puppet manages the configuration on all the servers in every deployment tier. The traditional r10k workflow does not account for multiple parallel apps represented in the codebase being deployed to SDLC deployment tiers at differing rates. An r10k control repo for which each git branch is a deployable, testable version of the code is a great _development_ workflow. It is not a great _deployment_ workflow for enterprises whose deployment tiers are not single-threaded.

Dealing with multi-threaded deployment tiers in r10k today starts to quickly require a very high level of proficiency with git. It becomes an exercise of re-writing history and keeping track of which threads have been weaved into which deployment tapestry. For the non-expert user this can be fraught with uncertainty or frustration.

At its root, this difficulty comes from a conflation of development with deployment.

See Appendices A and B for example of common approaches customers have tried to use to work through this problem.

## Mental Model for Deployment

This is a Puppet workflow thought experiment.

What if Git branches didn't control deployment of Puppet code?

Consider this idea instead. A version-centric (not necessarily git-centric) workflow exists that is ONLY about defining and working with deployments of Puppet code.

The model for how to think about Puppet code deployments is broken down below in terms of _things_ and _actions_.

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

Deployments are all about making changes, or promoting change. The fact that a module is independently promotable is very important. It means module change promotion rates for modules are decoupled from each other. One module (A) may promote a change from "test" to "staging" once a week, while another module (B) pushes changes through from "test" to "staging" once a day. Changes made to module (B) do not stack up behind changes to module (A) in a single queue, nor does one change advancing past another require git wizardry to accomplish. Each module has its own linear change history and any version of that module may be pushed or promoted to any deployment at any time.

A list of modules is defined as part of a deployment and each module in that list has:

* A name. The name of the module
* A source. e.g. git://github.com/puppetlabs/puppetlabs-ntp.git
* A version reference. Can be static (like a release version, sha, tag), or dynamic (like a branch)

Note that because the version reference can be dynamic, it's possible for a deployment to be dynamic.

#### Core

The _core_ of a deployment is similar to a module in that it is an independently promotable unit of change. The core has a distinction though in that there may only be one core in a deployment.

#### Code

A particular definition of core and modules, when referred to as a whole, may be referred to as the code.

### Interesting Actions

Basic CRUD actions can be implied based on the thing descriptions above. Here I'll describe some of actions most interesting from a workflow perspective, and why they're interesting.

#### Create (deployment)

Create a new deployment from scratch. This is an unusual action to take and would typically only be done as part of setting up a promotion workflow for the first time.

#### Duplicate (deployment)

Create a new deployment based on an existing deployment as a template. This action may occur frequently as part of the development workflow. To create a new module feature a dev deployment might be created as a duplicate of production, and then a development version of module (A) promoted to it. This development deployment would be used by the developer to test and iterate on proposed changes to module (A) until the changes were ready to start promoting through the SDLC change pipeline.

#### Update (deployment)

An update to a deployment is most likely to occur immediately after duplicating a deployment. The reason for this is that outside of that one use case, the promote action is is likely to be preferred over update in a workflow.

Updating a deployment is literally just editing any part of its definition. Core, modules, deployment tier, or name.

#### Refresh (deployment)

Check all the dynamic modules and/or dynamic core in a deployment and if necessary, refresh the deployment on disk (the Puppet environment) with any changes.

The deployment definition doesn't change in a refresh, but if there is new content in any of the dynamic references it has, that new content will be reflected in the deployment after a refresh.

#### Promote (module or core)

This workflow is all about promoting change. _Promote_ is the primary action by which change is propogated from one deployment to another.

To promote a change means to select core and/or module(s) from a deployment (A) and update another deployment (B) such that (B) is made to contain the same object(s) at the same version(s) as exist in (A).

An example promotion might be promoting the bigfix module version 1.2.0 from test to staging. Or, it might be promoting a change in core, 2e73137a, from staging into production.

Importantly, the rate at which changes are promoted for one object is independent of the promotion rate of all other objects.

#### Promote (code)

To facilitate faster or simpler workflows, it is possible to take the code in a deployment (A) and replicate it to deployment (B), fully replacing whatever code definition (B) had previously. This constitutes promoting the code as a unit of change, rather than the core or a module.

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

## Implementation

This could be implemented as an API-driven service, part of Code Manager. It could also be implemented as a kind of control repo, where a single flat file represents each deployment. All that can be figured out. A sample flat-file implementation is included in this repo.

## Objectives

This opportunity is seeking a way to

* Make DEPLOYING something an intentional action separate from DEVELOPING it
* Provide a workflow for deploying different apps to different deployment tiers at different rates WHILE
* Providing a versioned, single source of truth about what deployments exist and which versions of decoupled modules have been deployed to them AND
* Providing a way to stage and make atomic deployments involving more than one decoupled module WITH
* A SIMPLE git workflow

R10k alone provides only a single-threaded deployment workflow. Our customers need multi-threaded deployments.

# Appendixes

## Appendix A: Feature branch merge deployment strategy

An example approach to the problem statement that some customers have explored today.

This approach uses Git to keep features in branches, independently referencable until they've been merged into every deployment environment.

The downside to this approach is it is heavier on the use of Git, and when things go wrong they have to be fixed using Git.

### Merging strategy

* Ordered, named branches are maintained in Git where each branch corresponds to a SDLC stage
    * There are _n_ total SDLC branches in the ordered SDLC promotion path
    * "integration" is the 1st branch
    * "production" is the _n_-th branch
* Features are developed in feature branches forked from production
* Feature branches are iteratively merged into the ordered SDLC branches, starting with integration
    1. The feature branch is merged into integration: `git checkout integration; git merge feature`
    2. The feature branch is merged into the integration+1 SDLC branch: `git checkout [integration+1]; git merge feature`
    3. The feature branch is merged in order into each of the remaining non-production SDLC branches
    4. The feature branch is merged into production
* If changes are made to a feature branch before it is merged into production, its promotion target resets back to integration
* Feature branches are deleted after being merged into production
* After a feature branch has been merged into integration it should not deleted until merged to production. If the feature is abandoned, the feature branch commits must all be reverted (in the feature branch), and the "abandoned" feature branch then promoted through to production.

### Actions

#### Promote a feature branch

Promoting a feature branch means merge it to the next SDLC branch. The "next" SDLC branch is the first SDLC branch, starting from integration, which does not contain the feature branch's HEAD commit.

#### Make a change to a feature branch

"Fixing up" a feature branch is making any new commit to it. An implication worth noting is that making a change to a feature branch which has already been promoted "resets" that feature branch's "next" SDLC branch back to integration.

#### Resolve a merge conflict

Unfortunately, resolving merge conflicts may still be necessary on occasion. When this happens it will likely require git work and human judgement to resolve. It is difficult to suggest an algorithmic approach to resolution.

## Appendix B: Profile versions deployment strategy

An example approach to the problem statement that some customers have explored today.

A common strategy is to build parameterized classes and use Hiera data to activate different configurations based on a node's SDLC tier. In this model, Puppet code contains simultaneously multiple different configurations, and which is deployed is selected by Hiera.

The downside to this approach is that the code becomes more complicated.

### Puppet Code

```puppet
class profile::fto (
  Integer[1, 2] $profile_version = 2,
) {

  case $profile_version {
    1: { include profile::fto::v1 }
    2: { include profile::fto::v2 }
    default: { fail("cannot configure ${title} profile_version ${profile_version}."
  }

}
```

### Hiera

Hiera contains per-SDLC environment settings indicating which profile versions should be used for each application. E.g.

*deployment\_tier/production.yaml:*

```yaml
---
profile::fto::profile_version: 1
```

*deployment\_tier/development.yaml:*

```yaml
---
module::profile_fto::version: "1.1.0"
module::profile_fh::version: "4.0.0"
module::profile_bigfix::version: "2.0.0"
```

### Create a new profile version

1. Create a feature branch for the new profile change
2. Modify the profile
    * Edit the `profile_version` parameter validation to include the next version number
    * Edit the `profile_version` parameter's default value to be the newest valid version number
    * Edit the case statement, adding the new version number and `include` block
    * Edit the `profile_version` parameter validation and case statement, removing the oldest version(s) that are no longer deployed in any SDLC environment
3. Modify hiera
    * Edit `common.yaml` and set the relevant `*::profile_version` parameter to the _currently deployed_ version number. If multiple versions are currently deployed set it to the lowest currently deployed number
    * Edit each `deployment_tier/*.yaml` file and make sure the relevant `*::profile_version` parameter is set to the correct value for that SDLC environment
4. Merge the feature branch

### Promotion workflow

1. Create a feature branch for the new version deployment
2. Edit the `deployment_tier/*.yaml` for the SDLC environment you want to promote a version to and set the relevant `*::profile_version` parameter
3. Merge the feature branch
his could be implemented as an API-driven service, part of Code Manager. It could also be implemented as a kind of control repo, where a single flat file represents each deployment. All that can be figured out. A sample flat-file implementation is included in this repo.

## Appendix C: WHY?

Today there is no clear line of deliniation between version of code and deployment.

If everything is in the control repo, you can't deploy different apps to differrent deployment tiers at different rates. Every change goes into the same queue, and is in line BEHIND the change in front of it regardless of dependency.

If you de-couple a module from the control repo using the Puppetfile by setting `:branch => :control_branch`, deployment of that module has been 100% decoupled from the control repo and you have two potential problems.

1. The final gate to deploying a change to the decoupled module is committing code to a branch in that module. This may be ok, or it may be a problem.
2. There is no single, versioned history of what permutation of different versions of Puppet code and modules was deployed to a given Deployment Tier at any given point in time. You can't easily reproduce a deployment.
3. It is difficult to make an atomic deployment involving more than one decoupled module.

This thought experiment is seeking a way to

* Make DEPLOYING something an intentional action separate from DEVELOPING it
* Provide a workflow for deploying different apps to different deployment tiers at different rates WHILE
* Providing a versioned, single source of truth about what deployments exist and which versions of decoupled modules have been deployed to them AND
* Providing a way to stage and make atomic deployments involving more than one decoupled module WITH
* A SIMPLE git workflow

## Appendix D: File-based Sample

The rest of the files in this repo represent how the mental model described might look if it were implemented as a set of flat configuration files.
