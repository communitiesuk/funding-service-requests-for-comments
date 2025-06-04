```
Title: Authorisation model and implementation
Owner: Marc Usher
Collaborator(s): Platform Team
Created on: 2025-05-28
Status: Final 
Finalised on: 2025-06-04
```

## Overview
As part of the new Funding Service platform — Deliver Grant Funding, for internal users, and Access Grant Funding, 
for external and end users —  we need a robust and reliable approach to authorisation. 

The platform will serve a number of different users (Funding Service admins, grant teams, other government departments
(OGDs), assessors, grant recipients at both pre and post-award, Section 151 Officers). There is an established need 
for these users to have different levels of permissions within the platform and for certain actions and flows to be 
restricted to certain roles within the platform, within an organisation or within an individual grant. There is also a 
need and desire to allow collaborative submissions of data (though audit trails etc. of shared submissions is not in
the scope of this ADR), and a likely need for organisations to be able to allocate individuals to specific sections of 
an application or monitoring data for collaborative submissions of data. This will require an organisation view of users
which our current digital services do not have.

As we think about some level of self-service in the future of the platform there is also the desire to allow certain 
existing users to invite others to the platform, to lessen the onboarding burden within the Funding Service.

This ADR specifically focuses on authorisation, not authentication, and the thinking behind how we want to manage user
roles and permissions within the platform.

## What is the current state?

The current user management for applicants is based on a single-user model, which while allowing a user to complete an 
application, does not have any strong concept of a user’s organisation. This limits us from implementing a range of 
functionality such as:

* Allowing users to view or collaborate on data submissions from the same organisation, including at different stages
such as between pre and post-award
* Limiting submissions per organisation (for example, enforcing only one submission per Local Authority or organisation)
* From a reporting side, allowing us to track which organisations from an expected list have submitted a monitoring
return
* Having a stronger concept of organisations also supports data standards, as organisations can be strongly linked to 
known identifiers, enabling easier place-based reporting

On the Assessment side we are constrained by needing to request from IT a new Security Group and Roles for each grant 
that we onboard. This means currently putting a fund into assessment will currently always require developer input to 
configure this element. Whilst we have requested more control over this process from IT, it has not yet been granted 
several months since the initial request.

### Delta's model
Delta already has a strong concept of organisations and roles (such as a Section 151 Officer within an organisation), 
as well as the ability to assign organisations to take part in specific collections.

The Delta model is also very self-serve and built on a level of trust:
* Users self-select their roles in a profile management type view
    * Admins can assign themselves and other users to Access Groups and Roles; standard users can only assign themselves 
    to certain roles.
* Delta knows you are part of a certain Organisation based on email domain
* Users and Organisation owners do not get any email notifications when they or other users select/deselect roles. The 
only email notifications around roles/permissions are for S151 Officers
* If an organisation has no S151 Officers, then new/existing users can self-select that role. Otherwise, the existing
S151 can nominate delegated approvers. This generally isn't self-service and requires some manual intervention
* If a User doesn't have the relevant email domain but are the S151/other key role for that Organisation (eg. a 
consultancy or senior representative from a different Local Authority) then it's a manual process to add them, check 
their credentials and legitimacy of the request, all of which is quite time consuming but a necessary audit and due
diligence process. 
* If a user has a set role, they have that for all the collections/Access groups that they're a part of or have access 
to - a role is set for the user, not for the collection/organisation.

See a more comprehensive overview of [Delta's roles and permissions model on Confluence](https://mhclgdigital.atlassian.net/wiki/spaces/DT/pages/114262071/Permissions+Review).

## Why should we change?

As established in the overview, the current User Management and Access Control functionality in the existing services
limits us from implementing a number of features which are known user needs. Being able to implement these would be a
significant value-add from the new platform and should allow for further iteration and refinement going forward as the
platform and the needs of its users develop. We don't want to build ourselves into a corner where the technological
constraints lead to a hard no on implementing new features and functionality.

## How should we address this?

While we would continue to use Entra ID to authenticate internal users and other authentication methods (magic links,
federate with Delta’s existing user login, potentially GOV.UK OneLogin), we feel implementing our own Role-Based Access 
Control for user permissions would enable us to remove the current dependency on IT and have greater granular control
over permissions within the platform to meet the user needs outlined above.

### Model

Controlling User Management and Access Control ourselves is not an insignificant task, and one that needs to have a clear
model and concept. We also have the added complication that our model would ideally work for external users (ie. grant
applicants/recipients) and internal users (grant team members, OGDs, Funding Service admins).

Our database already includes the concept of Users, Grants and Organisations (though as of the time of writing our
Organisation model is very light-touch, but will be fleshed out later as we think through reference data and the
[Organisation data standard](https://communitiesuk.github.io/funding-service-data-standards/organisation.html)).

Creating a single UserRole table which acts as a join table between these three tables, with nullable `organisation_id` 
and `grant_id` columns, allows us to have very granular permissions at a full platform level, organisation level or grant level.

![Access Control Model](/images/userrole-model.png)

In the example model above, Roles are a set list controlled by an enum which can easily be extended but keeps roles to
a known quantity.

You then have the ability to define roles and permissions by using the nullable columns:

| Role type | Description | Database row |
| --- | --- | --- |
| Funding Service administrator | Full platform admin priveleges | ADMIN role with no `organisation_id` or `grant_id` |
| MHCLG Organisation admin | An internal user who needs full overview of all MHCLG grants | ADMIN role with no `grant_id` |
| External Organisation admin | An external user who needs full overview of that Organisation's applications/monitoring returns | ADMIN role with no `grant_id` |
| Organisation member | A user who needs read-only access to everything attached to an organisation | MEMBER role with no `grant_id` |
| Grant admin | A user who needs top-level priveleges for a specific grant, either internal eg. grant team admin or external eg. someone with oversight of an entire application or monitoring return | ADMIN role with both an `organisation_id` and `grant_id` |
| Organisation S151 Officer | An Organisation's S151 Officer | S151_OFFICER role with an `organisation_id` |
| Grant editor | An internal user who will need to edit Grant details or Grant collection information, or an external user who can edit parts of a grant application or monitoring return | EDITOR role with both an `organisation_id` and `grant_id` |
| Grant Assessor | An internal user assigned to assess individual grant applications or review other submitted data | ASSESSOR role with an `organisation_id` and `grant_id`|

This also allows developers to produce easy, reusable checks for permissions levels and only need to check one table based on the active user's
information.

For the purposes of the initial MVP, for Deliver Grant Funding (internal facing) we're catering for Funding Service administrators and Grant team members (who will have read-only access to one or more Grants). As with the examples given above, the building blocks are in place to easily extend this.

There are a couple of product decisions which will need to be made:
* how far the concept of 'self-service' goes, ie. can organisation or grant administrators invite other users?
* how visible will the information be? Do we default to locked-down visibility based on explicit permissions (ie. only people explicitly linked with a grant can see that Grant's information) or do we have a more open approach (eg. Organisation admins can see all information attached to that Organisation?), but the above model should allow us to cater for either approach.

### Organisation concept

We would need some concept or ability to associate users with an organisation, eg. based on their email domain, allowing
them to self-select, a more manual administrator task etc.

For internal MHCLG users who authenticate with Entra ID (Funding Service administrators, grant team members, some 
internal assessors), this will be able to be done programmatically on login. For grant applicants/recipients
(including Section 151 Officers) this won't be possible, so following Delta's pattern of using email domains to detect
organisation is a sensible starting point. 

There are known edge cases (such as the S151 Officer case outlined above in the Delta section, consultants filling out or submitting monitoring returns on behalf of multiple Local Authorities, or grant recipients who do not have an obvious email domain eg. a community group grant applicant with a gmail email address) which will need to be accounted for. Some of this could be manually done by Funding Service 
admins within the platform or require some level of requested organisation information on that user's first login, other instances (such as S151 Officers) could be handled by getting them to authenticate via Delta and receiving organisation information back with that authentication request.

Using email domain is a sensible first pass at detecting user organisations without manual intervention, but we will need to users being attached to multiple organisations and the ability to override the detected organisation.

If Organisation becomes a core concept in the views or grants that are visible to a user, we can also think about an approach
similar to GOV.UK Notify which has a simple 'switch organisation/service' option in the header.

### Learnings
As mentioned above there are a number of lessons we're taking from other services. With Delta, we're keen to allow
authentication via Delta for some key user groups further down the line (including returning relevant user information
with the response such as Organisation and Section 151 Officer role), but also learning from their patterns and approach.

Above we've also mentioned GOV.UK Notify - their model of an Organisation (which can have overall administrators), with
Services (ie. analogous to our concept of 'Grants') sitting below and users being able to be assigned to one or more Organisations
and to one or more Services is something we have tried to embody in our own model as it keeps the concepts simple and
easy to parse for developers, designers and users. As an existing pattern used by a large-scale government service this
should also mean developers and some users have a level of familiarity with this pattern which will lead to easier
adoption of the model in the platform.

## What alternatives are we discarding?

* The current system - we know this is overly restrictive and not fit for the user needs we have, as well as being a very
manual process that relies heavily on IT
* The full Delta model (flat permissions and self-serve) - there are a number of patterns we can reuse from Delta's roles and permissions model but we know that even if we move to a more self-serve permissions model later there will be some level of control still needed, so what we end up with will likely be between these two options
* Off the shelf/third-party platform (eg. [Amazon Cognito](https://aws.amazon.com/cognito/)) - this was considered but in the end decided it wasn't what we needed for now for a number of reasons:
    * It adds an additional layer of complexity and dependency to the platform and makes us reliant on another third-party system that would slightly force us to work within its own framework
    * Building out the Access Control ourselves lets us keep things simple and controlled in the short term to build for MVP with our specific use-cases in mind while also making sure the solution is as future-proof as possible
    * We are currently quite reliant on Azure for permissions/role-based access as well as authentication, and there are a lot of pain-points with this (having to add people to AD groups etc.), so keeping authentication with AD and then having more in-platform control of the roles and permissions seemed like a sensible approach
This may be something we need to revisit as the complexity grows, but this also meets our team principles of being pragmatic - "right-sizing" solutions and avoiding over-engineering solutions - and keeping things simple and maintainable as much as possible.

## Who will be affected?

This will be a significant change in our how our service operates, so will affect all internal and external users as well as the Funding Service team (eg. developers, onboarding etc.). However, with some users having familiarity with Delta and Notify, this should give them an existing level of familiarity with this approach.

## Who will benefit?

Hopefully everyone - it will be less painful than the current system, give us greater flexibility and control over user management and access control that will allow us to meet our user needs for the platform, and give greater visibility and audit trails across the board.

## What are the key risks to manage or mitigate?

* Role complexity in the codebase - we preach simplicity and maintainability so we don't want the number of roles to grow and become unwieldy leading to confusing permissions checks within the codebase. Everything needs to be well defined and managed.
* Organisation reference data - the question of Organisation reference data (particularly regarding grant recipient organisations such as Local Authorities) is still an open question as to where this will sit and how we will pull it into the platform
* Balance of control vs visibility - particularly as the number of roles grow and we move towards some level of self-service, we'll need to be sure that people don't see data or information they shouldn't but also that user and role management does not become onerous
