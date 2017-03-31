# Deployment Repo

This is a Puppet workflow thought experiment.

What if branches didn't control deployments?

What if instead, a repository exists that is ONLY about defining deployments of Puppet code? A deployment is roughly synonymous with a Puppet environment. A tier is roughly synonymous with a customer environment. Every deployment has a tier, and defines a collection of Puppet code and modules to deploy.

The long-form of "tier" is Deployment Tier.

There don't need to be multiple branches. A file can describe each deployment, and there can be multiple files. All the files exist in the master branch of the "deployment repo".

Each yaml file in this repo represents a deployment. 

To make a new deployment, make a new file.

Pushing a change to the deployment-repo triggers a deployment of whatever is changing.

## Feature branches and canary testing

If a single version of from single repo is the source of truth for all deployments (Puppet environments), how does development get done?

Make a new deployment. Choose which existing deployment to base a new one off of, and copy it. Define any differences, commit the file. Use the deployment. When finished, delete the deployment.

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
