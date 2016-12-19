- Feature Name: Popcon-style data aggregation for Foreman
- Start Date: 2016-12-19
- RFC PR: 
- Issue: 

# Summary
[summary]: #summary

Add an set of information to Foreman (similar to the About page) which will be
sent (opt-out available) to the Foreman infra for aggregation. This enables us
to learn more about things like userbase size, how up-to-date the community
stays, plugin / CR popularity, smart-proxy features, and so on.

# Motivation
[motivation]: #motivation

We have a lot of data on the Foreman developer community (via information from
GitHub, Redmine, IRC, and the mailing lists). However, these people have
self-selected to participate in our community, and represent a smaller, and
more vocal, subset of the whole Foreman userbase. We know remarkably little
about the wider community (not even how big it is).

If we are to make decisions that benefit the whole userbase, we need better
data, and from the people who aren't coming to talk to us via the
aforementioned channels. A data upload from running Foreman servers on a
regular, infrequent basis would help us get a clearer picture of the situation

# Detailed design
[design]: #detailed-design

This comes in two parts:

## 1 - Foreman-side sender

Foreman needs to generate and send a payload of information to a specified address. This is mostly what we'd on the About page, plus some extras. As a starting point, I'd suggest:

* UUID: to ensure we update existing installations
* Core version: eg '1.13.2'
* Proxies: an array of hashes of versions and features
* Plugins: an array of hashes of names and versions
* Compute resources: an array of types (not called with uniq, so we get accurate numbers)

This might look like:

```
{
  'uuid': '1234567890abcdef',
  'core_version': '1.13.2',
  'proxies': [
    { 'version': '1.13.0', 'features': ['TFTP','DHCP','DNS'] },
    { 'version': '1.12.1', 'features': ['Puppet', 'PuppetCA'] }
  ],
  'plugins': [
    { 'name': 'foreman_discovery', 'version': '7.0.0' },
    { 'name': 'foreman_templates', 'version': '3.0.0' }
  ],
  'compute_resources': [
    'libvirt',
    'libvirt',
    'digitalocean',
  ]
}
```

This is extensible, the hash have more keys added as necessary.

That seems the minimum to answer some basic questions about the size of the
userbase and where we should focus our efforts, without asking for any
sensitive data. Depending on reception of the feature, we could later extend
this with things like the number of hosts/domains/subnets in use, Operating
System (of Foreman and/or hosts) and so on.

This data can be added to the About page, and a Rake task added to send it to
the Foreman infra via a JSON POST. A cron job can be used to call the Rake task.

**Mitigations**: This has potential to greatly upset users if not done with care.
At least three things should be done:

* Upon upgrade to the version containing the feature, the admin's attention
  should be drawn to the feature (see questions below)
* Attention should also be drawn to the About page, where the admin can see the
  current data which would be uploaded
* A Setting should be added to opt-out of the upload.

Opt-out rather than opt-in is specifically used so that we can have reasonable
confidence in the dataset. Opt-in is likely to have a low take-up, leading to
simply another small slice of self-selecting users.

## 2 - Infra-side

The receiving can be done fairly simply via something like an API-only
Sinatra/Rails app, living in the Foreman infrastructure. It needs only to have
one route:

```
POST /data
```

The first is obviously the receiving URL for the data upload from part 1. This
will need to process the incoming JSON and store it appropriately in a DB. An
index on UUID would clearly be needed so that finding & updating a previous
record is quick. Given the has-many relationship of proxies, plugins, and CRs
in the incoming data, a schema might be:

```
uploads:
- id           [int]
- uuid         [string]
- core_version [string]

plugins:
- id           [int]
- upload_id    [int]
- version      [string]
- features     [string (or maybe hstore...)]

plugins:
- id           [int]
- upload_id    [int]
- name         [string]
- version      [string]

compute_resources:
- id           [int]
- upload_id    [int]
- type         [string]
```

Once the data is stored, calculation and deisplay of the summaries is required.
We are already working on a community dashboard [1] via the ReDash project -
thus, our ReDash instance could be given read-access to the db to generate
interesting graphs from the data.

# Drawbacks
[drawbacks]: #drawbacks

THe biggest drawback is upsetting users. Numerous projects have added
phone-home systems in the past, and often it is met with dismay and unhappiness
in the user base. We can add some extra work to mitigate this (see main
implementation, search for 'mitigation').

# Alternatives
[alternatives]: #alternatives

No alternatives are considered - one could consider the yearly community survey
an alternative, but evidence shows low take-up on that.

Not doing this will extend the status quo - we simply do not know very much
about our community.

# Unresolved questions
[unresolved]: #unresolved-questions

## Choice of app framework / DB

I'm leaning towards Sinatra & DataMapper because I know them best. However,
others with more knowledge are welcome to propose better options - in
particular, the JSON structures shown may lend themselves to a NoSQL setup.

## Drawing admin attention

The best way to *show* users that this new feature is present is to be
determined. Obvious choices would be something in the packaging (so it shows on
YUM/APT upgrade), notification in the installer, and *definitely* something in
the UI (similar to the welcome page). Thoughts on the best way to achieve this
are welcome.

## Using ReDash to display data

ReDash is working well so far, so I've avoided adding GET urls to the app for
returning stats. Other options would be to design an API to return data, so
that others can produce their own graphs.
