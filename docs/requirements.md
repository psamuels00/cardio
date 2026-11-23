# Cardio Application Requirements

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Document purpose](#document-purpose)
- [Application intro](#application-intro)
  - [Features](#features)
- [Users and authentication](#users-and-authentication)
  - [Admin users](#admin-users)
  - [Sample user](#sample-user)
  - [User settings](#user-settings)
- [Domain concepts](#domain-concepts)
  - [User](#user)
  - [Measurement](#measurement)
  - [Shared access link](#shared-access-link)
  - [Saved view](#saved-view)
  - [Session](#session)
- [User interface](#user-interface)
  - [Website domain](#website-domain)
  - [Graph page, before authentication](#graph-page-before-authentication)
  - [Data page, before authentication](#data-page-before-authentication)
  - [Transfer page, before authentication](#transfer-page-before-authentication)
  - [Graph page, after authentication](#graph-page-after-authentication)
    - [Notes](#notes)
  - [Data page, after authentication](#data-page-after-authentication)
  - [About page](#about-page)
  - [Transfer page](#transfer-page)
  - [Shared access link page](#shared-access-link-page)
  - [Admin users page](#admin-users-page)
- [Captured views](#captured-views)
  - [Bookmarks](#bookmarks)
  - [Shared access links](#shared-access-links)
  - [PDF files](#pdf-files)
- [Import and export](#import-and-export)
- [Testing](#testing)
- [Spec/implementation details](#specimplementation-details)
  - [Technology decisions](#technology-decisions)
  - [Shared view session](#shared-view-session)
  - [Supabase integration](#supabase-integration)
  - [Terraform](#terraform)
  - [Testing](#testing-1)
- [Open issues](#open-issues)

<!-- END doctoc -->


## Document purpose

This document will form the basis for development of a spec from which the
Cardio application will be implemented.


## Application intro

The name, Cardio, is short for cardiovascular. The primary purpose of the Cardio
app is to make it as easy as possible to track your blood pressure and heart
rate taken manually at arbitrary times. Another significant purpose is to make
it easy to share views of the data with someone.

The main display has a form to enter new readings: systolic, diastolic, and
heart rate; and a graph of recent measurements.


### Features

The user will be able to

- register, sign in, manage account settings, and sign out
- enter blood pressure and/or heart rate measurements
- see a graph of supplied data over a selectable period of time, including last
  7 days, last 30 days, last 60 days, last 90 days, and since an arbitrary date
- see trend lines on the graph
- update the browser URL so the current view can be bookmarked
- save the current view in a PDF
- view the data in a table format
- edit or delete previous entries, including capture timestamps
- generate a time-limited, shared access link
- view and manage shared access links

An admin user will be able to

- view users
- enable/disable a user
- change a user's role
- enable/disable new user creation


## Users and authentication

The application will support multiple users. Access requires authentication. A
user session remains authenticated for a day, by default, unless the user
explicitly signs out. See also Shared view sessions below.


### Admin users

A user may be designated as an admin. Cardiovascular data is not tracked for an
admin user. An admin user is only used for administration purposes, including
the following:

- view and manage all the users
- pause and resume acceptance of new registrations

For each user, the admin can enable or disable the user.

Instead of showing no data, the standard user pages have a message indicating
the signed in user is an admin, and there is nothing to show or, for the
transfer page, nothing to transfer, or something like that. There should be a
link to the Admin users page.


### Sample user

A sample user named Mr Cardio will be created and seeded with sample data. This
user will be read-only through the user interface, and will require no
authentication to access. It is this user’s data which is displayed on the Web
page when the session is not authenticated.

In order to make the sample data appear to be recent, all the persisted
timestamps will be shifted in the front-end by some amount that grows by a day
each day.


### User settings

A settings page will allow the user to change their email address, display name,
and password. The user can also delete their account, which will delete all
their data, including measurements, saved views, and shared access links. The
user will be prompted to confirm the deletion, and will be warned that it is
permanent and cannot be undone.

The settings page is available for admin users, but not the sample user.


## Domain concepts

The application data model is built on the following concepts:

- User: Someone who owns cardiovascular data and uses the application.
- Measurement: A recorded observation, containing one or more values and a
  timestamp.
- Shared access link: A mechanism for granting temporary, read-only access to
  some user's data.
- Saved view: A particular presentation of measurements: date range, visible
  series, and other graph settings.
- Session: An authenticated user session or a temporary guest session
  established through a shared link.


### User

Represents a person who has an account in Cardio.

Attributes:

- id
- email
- display_name
- role (standard user or admin)
- enabled
- created_at
- updated_at

Rules:

- A user must have a unique email address.
- A user can change their email address, but it must remain unique.
- A user may change their display name.
- A standard user owns their measurements, saved views, and shared access links.
- Authentication credentials and password management are handled separately,
  potentially by Supabase Auth.
- A disabled user cannot access Cardio, even if they have an existing
  authenticated session.
- A user may delete their account. In this case, their data, including
  measurements, saved views, and shared access links, will be deleted as well.
- An admin user does not own or track cardiovascular measurements.
- An admin can change a user's role between standard and admin, and can enable
  or disable a user.


### Measurement

Represents a blood pressure and/or heart rate reading taken at a particular
time.

Attributes:

- id
- user_id
- measured_at
- systolic
- diastolic
- heart_rate
- created_at
- updated_at

Rules:

- A measurement belongs to one user.
- Systolic and diastolic values must be supplied together.
- Heart rate is optional.
- At least one of the following must be supplied: a blood pressure reading or a
  heart rate reading.
- Measurements can be entered for a time in the past, but not for a time in the
  future.
- The measured_at timestamp represents when the reading was taken, not when it
  was entered, and support precision to the second.
- A user cannot have multiple measurements with the same measured_at timestamp.
- Measurements can be edited or deleted by their owner.
- Creation and update of a measurement is assumed to be done by the user who
  owns it.
- Measurements are not directly accessible to other users.
- Values must be valid integers within acceptable physiological ranges
  - systolic:
    - not (50 - 260): invalid, rejected
    - not (85 - 180): warning, but accepted
  - diastolic:
    - not (30 - 150): invalid, rejected
    - not (50 - 120): warning, but accepted
  - heart rate:
    - not (20 - 240): invalid, rejected
    - not (40 - 185): warning, but accepted


### Shared access link

Represents a URL that grants temporary, read-only access to a user's data
without requiring the guest to authenticate.

Attributes:

- id
- owner_user_id
- name
- share_token (or token hash)
- saved_view_id (or equivalent initial view context)
- created_at
- expires_at
- revoked_at
- last_used_at

Rules:

- A shared access link belongs to one user.
- The token is an opaque credential; it does not encode permissions directly.
- Access is granted only while the link is valid, unexpired, and not revoked.
- The owner can name, rename, extend, revoke, and delete links.
- A guest cannot create or manage shared access links.
- A guest has read-only access.
- A guest has access to all measurements.
- A guest can use all the normal graph controls to select what is displayed.
- A guest cannot change the initial view associated with the access token.
- A guest cannot import or export data.
- Access and permitted data are determined server-side.
- Use of a link is logged.
- Every use of a link is logged, including the source IP and access time.
- An expired link can be extended, making it active again.
- The owner can change the view associated with an existing link.
- Revocation immediately terminates an active guest session. At the very least,
  this means the user is no longer authenticated to fetch data from Supabase. If
  not too complex, the user will be immediately signed out of any active session
  and their browser will reload the page they are on.
- After a shared access link is used, the browser URL will be updated to not
  display the access token, even though it may still be in effect


### Saved view

Represents a particular presentation of a user's measurement data, including the
parameters and selections that define a Graph page.

Attributes:

- id
- owner_user_id
- name
- date_range (or start_date and end_date)
- show_systolic
- show_diastolic
- show_heart_rate
- show_trend_lines
- created_at
- updated_at

Rules:

- A saved view belongs to one user.
- A saved view determines which measurements and series are displayed and over
  what period.
- A saved view does not contain copies of the measurements themselves.
- A saved view can be used to recreate a Graph page.
- A saved view must be associated with a shared access link.
- Users cannot explicitly list, create, name, edit, and delete saved views.
- Saved views are completely distinct from bookmarks, which are simply URLs that
  code view parameters.
- A saved view cannot be shared by more than one link.
- A saved view captures only graph settings, not also table settings and other
  display preferences


### Session

Represents a period during which a browser has been granted access to Cardio,
either through normal user authentication or through a shared access link.

Attributes:

- id
- user_id (for an authenticated user session)
- shared_access_link_id (for a guest session)
- created_at
- expires_at
- last_activity_at
- revoked_at (or equivalent invalidation state)

Rules:

- An authenticated session is associated with one user.
- An authenticated user session normally lasts one day following the most recent
  activity, unless the user explicitly signs out.
- A guest session is established by presenting a valid shared access link.
- A guest session inherits the read-only permissions and data scope of its
  shared access link.
- A guest session cannot be used to create or manage shared access links or to
  import or export data.
- A session must not grant access after it has expired or been revoked.
- Disabling a user invalidates their ability to access Cardio, including through
  an existing session.
- Signing out immediately invalidates the session server-side.
- Revoking a shared access link invalidates guest sessions already established
  from it
- Extending a shared access link also extends existing guest sessions

Open questions:

- Are sessions persisted server-side, or are they represented by client-side
  tokens validated by the backend?


## User interface

In the layouts below, {Cardio} refers to text or an image. This is in the header
at the top of the page. Bracketed text implies a link, eg: [graph] is a link
using a disabled color on the Graph page. An exception is [select][v], which
indicates a drop-down select list.


### Website domain

Eventually, a domain name will be purchased and used for the Cardio application,
configured in GitHub Pages. Some available names include:

- mybp.health
- mybloodpressure.health
- arterialpressure.com
- damnbloodpressure.com


### Graph page, before authentication

    {Cardio} Blood Pressure Tracking     [graph] [data] [about] [signin/register]
                                                                       [bookmark]

    <sample graph, with controls>
    <sample new entry form, not submittable>


### Data page, before authentication

    {Cardio} Blood Pressure Tracking     [graph] [data] [about] [signin/register]

    <sample data table, with read-only controls>


### Transfer page, before authentication

    {Cardio} Blood Pressure Tracking     [graph] [data] [about] [signin/register]

    <explanation that this is where you can import and export csv>


### Graph page, after authentication

    {Cardio} Blood Pressure Tracking           [graph] [data] [about] [select][v]
                                                                       [bookmark]

    <graph, with controls>
    <new entry form>

The select list options include

- import/export
- shared access link
- signout

For an admin user, it includes

- admin: users
- signout

Following authentication, the main display will highlight a single graph
combining systolic, diastolic, and heart rate measurements, as well as a form
that can be used to supply a new measurements.

The graph will cover a number of days defaulting to 7, selectable using a link
below the graph labeled “last 7”, “last 30”, “last 60”, or “last 90”; or by
supplying a date in a text box labeled “since” with a default, grayed out value
such as 2026-11-21 representing a year ago.

Above the graph will be checkboxes, arranged horizontally, to separately
show/hide the systolic, diastolic, heart rate, and trend lines, all enabled by
default. The range covered by the graph will reflect these switches.

Below the graph is a form with three large, side-by-side text fields, labeled
above with small text, into which can be entered a new reading: systolic,
diastolic, and heart rate. All values are validated as integers. Any value that
can not represent an integer unambiguously will be red. Diastolic and systolic
must be supplied together, but both diastolic/systolic and heart rate are
optional. If at least one of the two is supplied, and all values are valid, a
submit button becomes enabled.

A link labeled “bookmark,” in the upper-right corner of the page, will update
the browser URL to include parameters making it possible to bookmark the current
view.

Selections made on the Graph page will be retained for the duration of a
session. After navigating away from the Graph page, the settings will be
re-applied upon return.

#### Notes

If a bookmark is used, requiring redirect to authentication, the target page is
displayed rather than this one after authentication. If a shared access link is
used, the target page is immediately displayed.


### Data page, after authentication

This page is the same as the Graph page, except that the data table is displayed
instead of the graph and new entry form.


### About page

    <header>

    <explanation of Web site purpose>
    <safe contact info>


### Transfer page

    <header>

    Import or Export data
    <explanation of expected/generated csv format>
    <sample data in csv format>
    <local file selection and submit option>
    <option to download all data>


### Shared access link page

    <header>

    <table of shared access links>


### Admin users page

    <header>

    <switch to enable/disable new registrations>
    <table of users>


## Captured views

There are three ways to capture a view of the data: bookmarks, shared access
links, and PDF files.


### Bookmarks

Bookmarks are used to return to a Graph page with the current parameters and
selections in place. Normally, the URL for the Cardio app is simple. If
included, parameters can customize the view automatically.

Every Graph page has a link labeled “bookmark” in the upper-right corner of the
page. When selected, the browser URL is updated to include the relevant
parameters. Then the page can be bookmarked using the standard Browser feature
for bookmarking a page.

Bookmarks are only for the Graph page, and only for the current user. They are
not shared. Neither the bookmarks, nor their generation, is tracked in the
database.


### Shared access links

A shared access link combines the concept of a bookmark with a temporarily
pre-authenticated session. One or more shared access links can be created. After
a shared access link is created, it can be shared with anyone.

The link will take the guest directly to the linked page and allow them to
navigate around the site in read-only mode until the token expires. They cannot
view or create shared access links, nor can they import or export data.

A separate page, available to the owner only, will be used to manage shared
access links, including the ability to name/rename, revoke, extend, or delete
them, as well as to view for each one, a log of its use, which shall include an
access time and source IP address.


### PDF files

Any view of the data on a Graph page can be captured as a PDF for download. This
will include the graph with a title and legend. It may also include the data the
graph is based on in a table below the graph.

Neither the PDF file, nor its generation, is tracked in the database.


## Import and export

A data transfer page will allow the user to import or export data in CSV format.
For import, the user can upload a local file. For export, the user will be able
to download a file.

An import operation will only proceed after the input file has been validated as
valid. The operation is idempotent, de-duping entries found to be redundant.


## Testing

There will be unit testing of all the Javascript logic placed in the page. If
possible, Supabase Edge function code will be unit tested as well.


## Spec/implementation details

Most of what is in this section does not belong in a requirements document and
will be moved out.


### Technology decisions

The application will be managed as a GitHub project. It will be developed with
the assistance of GitHub Copilot in PyCharm. Workflows will be used to test code
changes on feature branches, and deploy to GitHub Pages for public access when a
PR is merged. Terraform will be used as well for infrastructure updates.

Two environments will be supported: dev and prod. The dev environment is local
only, except the GitHub workflow for feature branch pushes will plan and apply
Terraform for the dev environment. The prod environment infrastructure is
updated when a feature branch is merged.

The data is stored in Supabase, using a separate project, each with its own
database, for each environment, and edge functions for simple logic, like
validating an access token. Terraform will be used to create an organization and
the two projects in Supabase. Creation and management of schema will be done by
the Supabase CLI, including migrations.

All the Web pages are static and contain logic in Javascript to update the
display and access the database via API. Tailwind CSS is used for styling the
page. Alpine JS is used for dynamic behaviors.

In a nutshell, Cardio is a data-backed Web app, publicly available through
GitHub Pages, serving static files with Javascript accessing data in Supabase
via its API, and with the database created and maintained in a GitHub workflow
running Terraform and the Supabase CLI for either a dev or prod environment.


### Shared view session

For the underlying technical implementation, use these terms:

- Feature: Shared View Sessions
- URL: Shared Access Link
- Credential: Share Token
- Authorization model: Time-limited, read-only guest access
- Initial state: Shared View Context

The token will authorize a scope of access, rather than encoding permissions
directly. This way, the token can be revoked, expired, or modified in its scope
server-side without changing the URL.

A Supabase Edge Function could validate the token and return the permitted data.
The browser would never need privileged database credentials.


### Supabase integration

Store Supabase access tokens and database passwords in GitHub Actions secrets or
environment secrets.

Use different Supabase project URLs and publishable keys for each environment.

We will use the Supabase CLI to create and manage the schema.

Take advantage of Supabase backups before production migrations.


### Terraform

The state files for both dev and prod will be stored in HCP Terraform, using a
free account. HCP Terraform will also be used to plan and apply. These actions
will be orchestrated from a GitHub workflow. The HCL will include a
`{ cloud { ... } }` block to delegate command execution and state management to
HCP Terraform remotely. The workflow will install terraform in the runner and,
then use standard commands like terraform init, terraform plan, and terraform
apply -auto-approve.

Each environment will have its own HCP workspace: cardio-dev and cardio-prod.


### Testing

Integration testing will be done using Playwright, which can run in a GitHub
workflow on feature branch pushes.


## Open issues

- The requirements are intentionally vague with respect to user registration and
  authentication, both the mechanism and the UI. I’m hoping to find support for
  this from Supabase or some other 3rd party, something with little impact on
  the Cardio schema. Otherwise, we will have to define all the usual
  requirements, like register, sign in, sign out, change password, forgot
  password.
- Should I use a static Web site builder, like Eleventy?
- Is there a simple way to retain an archive of deleted users' data?
