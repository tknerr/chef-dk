# ChefDK Policyfile  README

## What's this Policyfile Stuff?

First of all, the Policyfile feature is a work in progress. As of ChefDK
0.5.0 and Chef Server 12.1.0, native APIs are enabled by default,
allowing you to safely experiment with the feature without impacting
existing production systems. It's recommended that you upgrade to these
versions before experimenting with Policyfiles.

That said, it's possible that features you need to be successful in
production are not implemented yet. If you try it out and have any
feedback, first check out whether it's listed in the "Known Limitations"
section below. If your idea/need isn't listed there, or if you encounter
a bug, file an issue at https://github.com/opscode/chef-dk/issues

## Ok, I've Been Warned. What Is It?

One last warning: This section of the document describes features that
don't exist yet as if they do exist. The "Known Limitations" section
lists the features that are not yet complete.

The Policyfile is a Chef feature that allows you to specify precisely
which cookbook revisions `chef-client` should use, and which recipes
should be applied, via a single document. This document is uploaded to
the Chef Server, where it is associated with a group of nodes. When
these nodes run `chef-client` they fetch the cookbooks specified by the
policyfile, and run the recipes specified by the policyfile's run list.
A given revision of a policyfile can be promoted through deployment
stages to safely and reliably deploy new configuration to your
infrastructure.

A `Policyfile.rb` is a Ruby DSL where you specify a `run_list` and tell
ChefDK where to find the cookbooks. It looks like this:

```ruby
# Policyfile.rb
name "jenkins"

default_source :community

run_list "java", "jenkins::master", "recipe[policyfile_demo]"

cookbook "policyfile_demo", path: "cookbooks/policyfile_demo"
```

When you run a `chef install`, ChefDK caches any necessary cookbooks and
emits a `Policyfile.lock.json` that describes the versions of cookbooks
in use, a hash of the cookbooks' content, the cookbooks' sources, and
other relevant data. It looks like this (content snipped for brevity):

```json
{
  "revision_id": "288ed244f8db8bff3caf58147e840bbe079f76e0",
  "name": "jenkins",
  "run_list": [
    "recipe[java::default]",
    "recipe[jenkins::master]",
    "recipe[policyfile_demo::default]"
  ],
  "cookbook_locks": {
    "policyfile_demo": {
      "version": "0.1.0",
      "identifier": "f04cc40faf628253fe7d9566d66a1733fb1afbe9",
      "dotted_decimal_identifier": "67638399371010690.23642238397896298.25512023620585",
      "source": "cookbooks/policyfile_demo",
      "cache_key": null,
      "scm_info": null,
      "source_options": {
        "path": "cookbooks/policyfile_demo"
      }
    },
    "java": {
      "version": "1.24.0",
      "identifier": "4c24ae46a6633e424925c24e683e0f43786236a3",
      "dotted_decimal_identifier": "21432429158228798.18657774985439294.16782456927907",
      "cache_key": "java-1.24.0-supermarket.chef.io",
      "origin": "https://supermarket.chef.io/api/v1/cookbooks/java/versions/1.24.0/download",
      "source_options": {
        "artifactserver": "https://supermarket.chef.io/api/v1/cookbooks/java/versions/1.24.0/download",
        "version": "1.24.0"
      }
```

You can then test this set of cookbooks in a VM or cloud instance. When
you're ready to run this policy on a set of nodes attached to your
server, you run the `chef push` command to push this revision of the
policy to a specific `policy group`. Policy groups are containers for
nodes that all use the same revision of a given policy. For example a
node may be configured to use the policy name "webapp" or "database" and
belong to a `policy group` like "staging" or "prod-cluster-1". Each
policy group may have a different revision of each policy, so that the
"webapp" policy may have a different set of cookbooks and different run
list between the "staging" and "prod-cluster-1" policy groups.

The `chef push` command will also upload cookbooks to a new cookbooks
storage API which stores them according to their identifiers, which in
this case are SHA-1 hashes of the cookbooks' content (these are the
`identifier` fields in the above example.

When `chef-client` runs, it reads its policy name and policy group from
configuration, and requests the current policy for its name and group
from the server. It then fetches the cookbooks from the new storage API,
and then proceeds as normal.

## Policyfile Syntax

_Note:_ skip to the next section if you're only interested in learning
the "big picture" design of this feature and the motivation for creating
it.

A `Policyfile.rb` is the file where you name your policy, define a
`run_list` for `chef-client` to use to converge your systems, and tell
ChefDK where to find the cookbooks needed for your run list. It has four
methods:

### `name "NAME"` (required)

This names your policy. Generally you should use a name that reflects
the function that machines using this policy will perform in your
infrastructure, like "jenkins-master" or "chatserver".

### `run_list "ITEM1", "ITEM2", ...` (required)

This is the `run_list` that `chef-client` will use when it applies this
policy to a node (the `run_list` on the node object is ignored when
using policyfiles). At this time, ChefDK does not support roles in the
`run_list`, but support will be added in the future.

### `default_source :SOURCE_TYPE, *args`

This tells ChefDK where to find any cookbooks required by your
`run_list` that do not have specific locations configured by the
`cookbook` method (see below). If you specify a specific source for
every cookbook, then you do not need to configure this.

You may use more than one `default_source` statement if, for example,
you would like to keep your internal cookbooks in one repo and also use
community cookbooks from supermarket. Beware that these sources cannot
have any cookbooks in common, or else `chef` will return an error when
attempting to install the cookbooks, unless you have a `cookbook` entry
for that cookbook that specifies the source (see below).

#### community

This default source pulls cookbooks from either the public supermarket
or a private supermarket install. To use public supermarket:

```ruby
default_source :community
```

To use a private supermarket install:

```ruby
default_source :community, "https://mysupermarket.example"
```

#### chef\_repo

This treats a "monolithic" Chef cookbook repository as an artifact
store.

```ruby
default_source :chef_repo, "path/to/repo"
```

Note that the path argument can be either a path to the top level of
your Chef repo, or to the `cookbooks` directory within your repo.

#### chef\_server

Chef Server cannot be used as a source until the Server implements the
"universe" endpoint.

### `cookbook "NAME" [, "VERSION_CONSTRAINT"] [, SOURCE_OPTIONS]`

The `cookbook` method serves several purposes:

#### Add an Additional Cookbook

It can add a cookbook to the set of cookbooks ChefDK will compile,
when that cookbook isn't needed by the `run_list`.

Example:

```ruby
cookbook "apache2"
```

#### Specify a Version Constraint

It can specify an additional version constraint for the cookbook. You
can use this feature to constrain the versions of the cookbooks at the
top level of your `run_list`.

Example:

```ruby
run_list "jenkins::master"

# Restrict the jenkins cookbook to version 2.x, greater than 2.1
cookbook "jenkins", "~> 2.1"
```

#### Specify an Alternative Source

ChefDK can fetch cookbooks from Supermarket, git, and local disk (other
sources will be added in the future). This allows you to combine
cookbooks from multiple sources into a set of cookbooks that builds your
application.

Examples:

```ruby
cookbook 'my_app', path: 'cookbooks/my_app'
cookbook 'mysql', github: 'opscode-cookbooks/mysql', branch: 'master'
```

### `named_run_list "NAME", "ITEM1", "ITEM2", ...`

Policyfiles provide named run lists as an alternative/replacement for
the override run list feature in Chef Client. There are a variety of use
cases for named run lists, such as running a smaller set of recipes in
order to quickly converge configuration for a single application on a
host, or doing one-time setup tasks. Though useful, so-called "partial
convergence" can lead to configuration drift and other problems, so use
this feature carefully.

In the policyfile DSL, a named run list looks like this:

```ruby
named_run_list :update_app, "my_app_cookbook::default"
```

#### Define Attributes

The Policyfile DSL allows you to define default and override attributes,
using the exact same syntax you use in attributes files in your
cookbooks. When you run `chef-client`, these attributes will be applied
at the "role" precedence level.


```ruby
default["nested"]["attributes"] = "auto-vivify"
override["same"]["with"] = "overrides"
```

### Testing With Test Kitchen

ChefDK now includes a Test Kitchen provisioner, which allows you to
converge VMs using Chef Client in policyfile mode, using Chef Zero to
serve cookbook data. Add the following to your `.kitchen.yml`:

```yaml
provisioner:
  name: policyfile_zero
  require_chef_omnibus: 12.0.0-rc.2
```

## Applying the Policy on a Node

On the node you with to use the policy update the client.rb to include
the following:

```ruby
# Policyfile Settings:
use_policyfile true
# The following is true by default in chef versions >= 12.2.0
# and can be ommitted
policy_document_native_api true

policy_group "#{policy_group}"
policy_name "#{policy_name}"
```

If you are using the compatability mode you should use the following
settings:

```ruby
# Policyfile Settings:
use_policyfile true
policy_document_native_api false
deployment_group "#{policy_name}-#{policy_group}"
```
## Motivation and FAQ

We believe Policyfiles will greatly improve the experience of using Chef
to configure your infrastructure by allowing you to test and promote
your configuration code safely and with a more humane interface.
Policyfiles resolve many real-world problems that users experience with
the Chef workflows that are possible today.

### Focus Workflow on Configuring Machines to do Useful Work

Chef's current tooling (`knife` in particular) maps very closely to Chef
Server's REST API and therefore is centered around manipulating
individual objects and uploading them to the Chef Server. `chef-client`
assembles these pieces at run time (more on that below) to configure a
host to do some useful work for you organization. With the Policyfile
feature, we want to focus the workflow on creating and configuring
entire systems, rather than individual components. For example,
Policfiles describe whole systems and individual revisions of
`Policyfile.lock` documents are uploaded with all required components as
a unit to the Chef Server.

### Code Visibility

In Chef currently, the exact set of cookbooks that a node will apply is
defined by:

* The node's `run_list` property;
* Any roles present in the node's `run_list` or recursively included by
  those roles;
* The environment, which restricts the set of valid cookbook versions
available to a particular node according to a variety of constraint
operators;
* Dependencies specified in cookbook metadata;
* The dependency solver implementation, which tries to pick the "best"
set of cookbooks that meet the environment and dependency criteria.

These conditions are re-evaluated each time `chef-client` runs, so it's
not always easy to tell exactly which cookbooks `chef-client` will run,
or what the impact of updating a `role` or uploading a new cookbook will
be.

The Policyfile feature solves this problem by computing the cookbook set
on the workstation and producing a readable document of the solution.
`chef-client` runs re-use the same precomputed solution until you
explicitly update their specific policy group.

### Role Mutability

Roles are currently global objects and changes to existing roles are
applied immediately to all nodes that contain that role in their
`run_list` (either directly or via another role's `run_list`). This
means that updating existing roles can be very dangerous, so much so
that many users advocate abandoning them entirely.

The Policyfile feature improves the situation in two ways. Firstly,
roles are expanded at the time that the cookbook set is computed (i.e.,
the `chef install` step). Roles never appear in the
`Policyfile.lock.json` document. As a result, roles are "baked in" to a
particular revision of a policy, so that changes to a role can be tested
and rolled out gradually. Secondly, Policyfiles offer an alternative
means of managing the `run_list` for many nodes at once, since there is
a one-to-many relationship between policies and nodes. Therefore users
can, if desired, stop using roles without needing to use role cookbooks
as a workaround for managing the `run_list` of their nodes.

### Cookbook Mutability

The Chef Server currently allows an existing version of a cookbook to be
mutated. While this provides convenience for users who upload
in-development cookbook revisions to a Chef Server (this is common among
beginners and some Ci patterns), it offers the same problems as Role
mutability. Some users account for this by following a rigorous testing
process so that only fully integrated (i.e., all contributors' changes
are merged) and well tested cookbooks are ever published to the Chef
Server. While this process enforces good development habits, it is not
appropriate for everyone, and should not be a prerequisite for getting
safe behavior from the Chef Server.

The Policyfile feature solves this issue by using a new Chef Server
cookbook publishing API which does not provide cookbook mutability. In
order to avoid name collisions, cookbooks are stored by name and an
arbitrary ID, which is computed from the content of the cookbook itself.

One pesky example of name/version collisions is when users need to
temporarily use a fork of an upstream cookbook. Even if the user
contributes their change and the maintainer is very responsive, there
may be a period of time where the user needs to use their fork in order
to make progress. However, this presents a versioning quandry: if the
user doesn't update the version, they must overwrite the existing copy
of the cookbook on their server. Contrarily, if they do update the
version number, they might conflict with the version number of a future
release, which they could only fix by overwriting the newer version on
their server. Using content-based IDs with sourcing metadata makes this
use case easy.

#### But Opaque IDs are Confusing!

It's definitely true that such opaque IDs are less comfortable than the
name, version number scheme that users are used to. In order to
ameliorate the problem:

* When working with the `Policyfile.rb`, you deal with cookbooks
mostly in terms of names and version numbers. For cookbooks from an
artifact service like supermarket, you use names and version constraints
like you're used to; for cookbooks from git, you use branch/tag/revision
as you're used to; for cookbooks from disk, you specify paths. The
opaque IDs are mostly behind the scenes.
* Extra metadata about cookbooks is stored and included in API
responses, as well as the `Policyfile.lock.json`. This includes the
source of the cookbook (supermarket, git, local disk, etc.) and the
upstream ID of the cookbook (such as git revision); for cookbooks loaded
from local disk, the Policyfile implementation detects if they are in a
git repo and collects the upstream URL, current revision ID, whether the
repo is dirty, and whether the commits are pushed to the remote.
* Cookbooks uploaded to the new cookbook storage API can have extended
SemVer version numbers with prerelease sections, like `1.0.0-dev`.

### Limit Expensive Computation on Chef Server

In order to determine the cookbook set for a given chef-client run, Chef
Server has to load dependency data for all known versions of all
cookbooks, and then run an expensive (NP expensive) computation to
determine the correct set. Moving this computation to the workstation
and running it less frequently makes Chef Server more efficient.

### Where Does the Policyfile Live?

At the moment, we see three main ways to organize your Policyfiles:

* Store the Policyfile and related cookbooks in the same repository as
the application you're deploying. If you're deploying custom
applications written in-house and your software developers are
comfortable working with Chef, you can put the Policyfile in the same
repo as the application's source code and version everything together.
* Store the Policyfile with a cookbook. If you're following the single
cookbook per repo workflow, you can include the Policyfile in the
highest-level cookbook's (i.e., the cookbook ultimately responsible for
deploying a server's primary application) repository.
* Store all of your Policyfiles in a single directory. This is likely to
be the most common way to use Policyfiles with the monolithic cookbook
repo workflow.

### Am I Going to be Forced to Use This?

This is being rolled out by adding new functionality to Chef Client,
ChefDK, and Chef Server **alongside** the existing functionality. There
are no plans to remove or even deprecate the existing APIs. The plan is
to get people to switch by offering an alternative with both more safety
and more freedom than the current situation.

That said, if adoption is large enough then eventually removing the
server-side dependency solver will be considered.

### Does this Replace Berkshelf?

The Policyfile is definitely a replacement for the "environment
cookbook" pattern in Berkshelf. It also provides a dependency solver and
fetcher (thanks to some code from berks), so it may replace some other
berkshelf use cases. However it is much less opinionated than Berkshelf,
and may not replace Berkshelf for all use cases, so we'll have to see
how things turn out.

### Does this Force Me to Use the Single Cookbook per Repo Thing?

No. The "megarepo"/"monolithic repo" workflow is be supported. All of
the policyfile based commands in ChefDK support an optional
policyfile name as an argument, so you can add a `policies/` directory
to your Chef Repo (see also "Am I Going to be Forced to Use This?"
above).

Users who use the "megarepo" workflow may see some benefit to using
separate per-cookbook repos for third-party cookbooks, but this will be
optional and users can convert from vendor branches piecemeal if they
decide to do so.

### Do I Have to Change My Workflow to Use This?

The answer to this depends on how you define "workflow." As noted above,
you can choose to have a chef-repo or not, and you can fetch third party
cookbooks using the Policyfile or an out of band mechanism (vendor
branches). You and your team can decide to publish only completely
integrated "release" cookbooks to the server if that works for you, but
you can also safely publish development versions of cookbooks to the
server without risk of mutating the production versions and without
needing a versioning scheme (devodd and friends) to workaround cookbook
mutability issues.

That said, the mechanics of how you get configuration code from your
workstation to production will be different. In particular, when using
the Policyfile feature in the recommended way, you cannot publish an
updated cookbook or role and have it applied immediately to all
machines. Tools that use the old APIs will need to be updated.

### Are Policyfiles Versioned?

Policyfiles are versioned, but the versioning mechanism is different
than SemVer depedency resolution--it's more like git branching.

Each time you create or update a Policyfile lock document, ChefDK
generates a `revision_id` based on the lock content (again this is a
SHA1 hash) and inserts this into the resulting JSON.

Additionally, the Chef Server has a new concept called a _policy group_,
which is a collection of nodes. In practice, it will probably be common
to name each policy group after a deployment phase in your workflow,
e.g., "dev", "stage", "prod", though in more complex environments, users
may have "canary" groups for testing code changes or segment production
into clusters to control the pace of code roll-out.

Each policy group can have one active revision of a given policy name.
Thus, a user may generate revision "123abc" of the "webapp" policy, and
apply it to the "dev" policy group. When everyone is satisfied that this
revision works correctly, it may be applied to "stage" and then "prod".
Development then continues, and a user generates "456def" of the
"webapp" policy. When this is applied to "dev", "stage" and "prod" will
still use revision "123abc" of the "webapp" policy until someone updates
those.

### What About Environments?

Environments are not supported when using Policyfiles. Policyfiles
already provide dependency locking, using a much more convenient
mechanism.

Policyfiles do not provide a direct replacement for environment
attributes. For now, there are few ways to replace them:

#### Nested Attributes in the Policyfile

Put all the attributes you want in your Policyfile, but nest them one
level deeper, e.g.,

```ruby
default["production"]["app_setting"] = "prod value"
default["dev"]["app_setting"] = "dev value"
```

Then you adjust your cookbook code to hoist these values up.

The primary benefit of doing it this way is that the structure of your
attributes and your cookbook code are versioned together. For example,
if you need to change a value from a `String` to an `Array` or a `Hash`,
these attributes will be incompatible with the old code. But by updating
your cookbook set and attributes together, you can avoid exposing code
to incompatible attributes structures.

Another benefit is that your code controls which subset of attributes to
"hoist," so you can describe multiple concerns, such as lifecycle, data
center, etc. all in one place.

The major downsides are:

* Cannot update attributes everywhere immediately. This is the cost of
  having "versioned" attributes structures.
* Have to write code to hoist the attributes, which implies you need a
  wrapper cookbook when using this approach with community cookbooks.
* Might have to duplicate information across several policies.

#### Use Data Bags Instead

An alternative is to use data bag items and set whatever attributes you
need to the data bag item's content.

This has the following benefits:

* Data updates immediately when the data bag is uploaded.
* Can model multiple concerns such as lifecycle, data center, etc. by
  using different data bags
* Can share data bags between lots of policies

And the following downsides:

* Have to create your own versioning system if you want versioning, or
  be careful to maintain compatibility.
* Need to write wrapper cookbook code to assign attributes from the data
  bag item.

#### Search or Service Discovery

Many of the attributes you need to configure based on "environment"
describe how services integrate themselves with other components in
their stack. Search or a service discovery tool can help accomplish this
in a more dynamic way.

## Compatibility Mode

As of Chef Server 12.1.0, and ChefDK 0.5.0, the new Policyfile native
APIs are enabled by default. This is the safest and recommended way to
try Policyfiles.  If you are unable to upgrade your Chef Server, you can
operate ChefDK and Chef Client in a compatibility mode that uses
existing Chef Server APIs to demonstrate the Policyfile behavior. This
mode of operation can be dangerous if used in an existing Chef Server
organization. See the caveats below before using it.

### Cookbook Artifact Storage

In compatibility mode, ChefDK must implement content-hash-based storage
of cookbooks using the existing `/cookbooks` endpoint. To do so, it maps
hash IDs to `X.Y.Z` version numbers. While this works to demonstrate the
Policyfile behavior, it is certainly a kludge. If you are trying the
Policyfile feature in compatibility mode, beware:

* Cookbooks uploaded by the policyfile commands will have very large
version numbers with no sort order. Any `chef-client` that is not
operating in policyfile mode will prefer these cookbooks to ones
uploaded normally unless you are dilligent about using environment
version constraints.
* The `/cookbooks` endpoint is not designed to be used this way, so it
doesn't show you the "real" version numbers or additional metadata about
these cookbooks. While we have plans to make arbitrary cookbook IDs
easier to manage in the final implementation, there's little we can do
about it in the exiting API.

### Policyfile Storage

In compatibility mode, ChefDK uses data bag items to store
`Policyfile.lock.json` documents. To minimize the chance of conflict
with other data bag items, ChefDK stores all of these documents in the
"policyfiles" data bag; individual `Policyfile.lock.json` revisions are
given IDs of the form `$policyname-policygroup`.

## Known Limitations

The implementation of the Policyfile feature is still incomplete. This
is a (possibly not complete) list of planned features and use cases that
currently aren't implemented/supported.

### Conservative Updating

Individual cookbooks in a `Policyfile.lock.json` cannot be upgraded. You
have to recompute the entire thing from the `Policyfile.rb`. Support for
this will be added.

### Role Support

Roles currently cannot be used in the `Policyfile.rb` run list, but will
be supported in the future.

### Multiple Run List Support

ChefDK now provides a named run list feature which will provide roughly
the same functionality as Chef Client's override run list option,
however Chef Client's policyfile mode does not yet support the feature.
A future update to Chef Client will allow users to specify a named run
list to run instead of the primary run list.

### Private Cookbook Hosting

You can host your cookbooks privately if you run a private supermarket
instance. We would also like to add the ability to host your cookbooks
directly on a Chef Server, using the existing `/cookbooks` endpoint.
This requires implementation of a `/universe` endpoint on the Chef
Server, as described here:
https://github.com/opscode/chef-rfc/blob/master/rfc014-universe-endpoint.md

### Native API Incomplete

The native Policyfile APIs are not complete. For example, there is not
yet an API to delete policy revisions or policy groups. Also, the node
object is not yet Policyfile-aware, so you cannot easily list nodes by
policy group membership. Future versions of the Chef Server will add
these features.

