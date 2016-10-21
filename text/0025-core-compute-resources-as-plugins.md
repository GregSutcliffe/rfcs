- Feature Name: Core Compute Resources as Plugins
- Start Date: 2016-10-21
- RFC PR: (leave this empty)
- Issue: (leave this empty)

# Summary
[summary]: #summary

Core Compute Resources should move to plugins

# Motivation
[motivation]: #motivation

Core has been set up to allow compute resources as plugins for some while, and
we already have several examples of this in use (Xen, Azure, DigitalOcean,
etc). This has brought benefits:

* able to release independatly of Core
* make use of the modular Fog gems that are now available
* separate logging of issues
* dedicated maintainers who know the compute resource well

In addition, it allows the core project to mark a particular resource as
unmaintained when necessary (as happened with Xen). 

By comparison, some of the core compute resources are not well maintained, but
users have an expectation that they work, because we ship them. By moving them
to plugins, we gain the ability to properly support the maintainers or mark
them as curerntly in need of a maintainer. It's also easier for new
contributors to work on a specific resource they know, without having to learn
the whole core codebase.

Further, we can also consider using different plugins for different versions of
an API - this may be particularly helpful for different OpenStack releases,
since this seems to move quite quickly.

# Detailed design
[design]: #detailed-design

It may not be feasible to move *all* resources to plugins right away. For each
compute resource, we should:

* Identify at least one maintainer for the new plugin
* Create a new repo `theforeman/foreman-<resource>`
* Move the appropriate code from Foreman Core to the plugin
* Release the plugin
* Send a PR to Core to delete the unused code (and maybe Fog dependencies)

At this point, the plugin can now iterate on it's own release cycle.

There may, of course, be issues where Core is not sufficiently
flexible/extensive. These can be tackled in the usual way, via Redmine and PRs.

# Drawbacks
[drawbacks]: #drawbacks

The only drawbacks are:

1) finding interested parties to maintain the plugins - but this is no
different to finding people to maintain the code in core (which is not working)
2) more packaging work - but we already package the compute resources
separately anyway. This does not add much more.

# Alternatives
[alternatives]: #alternatives

The alternative is to leave the compute resources in core, which for the
motivation above is undesirable.

# Unresolved questions
[unresolved]: #unresolved-questions

One option is to version the namespace in a pluign (e.g. Foreman::Libvirt::V2).
This would allow the new plugin to co-exist with the core provider until
feature parity is reached. This does leave the question of whether to rename
the namespace once the core provider is deleted, however.
