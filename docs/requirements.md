# Cardio Application Requirements

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Document purpose](#document-purpose)
- [Application purpose](#application-purpose)
- [Features overview](#features-overview)
- [Technology overview](#technology-overview)
- [Users and authentication](#users-and-authentication)
  - [Admin users](#admin-users)
  - [Sample user](#sample-user)
- [Bookmarks](#bookmarks)
- [Shared access links](#shared-access-links)
- [Saved views](#saved-views)
- [Import and export](#import-and-export)
- [User interface](#user-interface)
  - [Page layouts](#page-layouts)
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
- [Testing](#testing)
- [Shared view session details](#shared-view-session-details)
- [Supabase integration details](#supabase-integration-details)
- [Terraform details](#terraform-details)
- [Open issues](#open-issues)

<!-- END doctoc -->

## Document purpose

This document will form the basis for development of a spec from which the
Cardio application will be implemented.

## Application purpose

The name, Cardio, is short for cardiovascular. The primary purpose of the Cardio
app is to make it as easy as possible to track your blood pressure and heart
rate taken manually at arbitrary times.

The main display has a form to enter new readings: systolic, diastolic, and
heart rate; and a graph of recent measurements.

## Features overview

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
- enable/disable new user creation

## Technology overview

The application will be managed as a GitHub project. It will be developed with
the assistance of GitHub Copilot in PyCharm. Workflows will be used to test code
changes on feature branches, and deploy to GitHub pages for public access when a
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
timestamps will be shifted by some amount that grows by a day each day.

## Bookmarks

Bookmarks are used to return to a Graph page with the current parameters and
selections in place. Normally, the URL for the Cardio app is simple. If
included, parameters can customize the view automatically.

Every Graph page has a link labeled “bookmark” in the upper-right corner of the
page. When selected, the browser URL is updated to include the relevant
parameters. Then the page can be bookmarked using the standard Browser feature
for bookmarking a page.

## Shared access links

One or more shared access links can be created. A shared access link combines a
bookmark with a temporarily pre-authenticated session. After a shared access
link is created, it can be shared with anyone.

The link will take the guest directly to the bookmarked page and allow them to
navigate around the site in read-only mode until the token expires. They cannot
view or create shared access links, nor can they import or export data. See
Shared view session details below.

A separate page will be used to manage shared access links, including the
ability to name/rename, revoke, extend, or delete them, as well as to view for
each one, a log of its use.

## Saved views

Any view of the data on a Graph page can be captured as a PDF, which can be
downloaded. This will include the graph with a title and legend. It may also
include the data the graph is based on in a table below the graph.

## Import and export

A data transfer page will allow the user to import or export data in CSV format.
For import, the user can upload a local file. For export, the user will be able
to download a file.

An import operation will only proceed after the input file has been validated as
valid. The operation is idempotent, de-duping entries found to be redundant.

## User interface

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

### Page layouts

In the layouts below, {Cardio} refers to text or an image. This is in the header
at the top of the page. Bracketed text implies a link, eg: [graph] is a link
using a disabled color on the Graph page. An exception is [select][v], which
indicates a drop-down select list.

#### Graph page, before authentication

    {Cardio} Blood Pressure Tracking     [graph] [data] [about] [signin/register]
                                                                       [bookmark]

    <sample graph, with controls>
    <sample new entry form, not submittable>

#### Data page, before authentication

    {Cardio} Blood Pressure Tracking     [graph] [data] [about] [signin/register]

    <sample data table, with read-only controls>

#### Transfer page, before authentication

    {Cardio} Blood Pressure Tracking     [graph] [data] [about] [signin/register]

    <explanation that this is where you can import and export csv>

#### Graph page, after authentication

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

##### Notes

If a bookmark is used, requiring redirect to authentication, the target page is
displayed rather than this one after authentication. If a shared access link is
used, the target page is immediately displayed.

#### Data page, after authentication

This page is the same as the Graph page, except that the data table is displayed
instead of the graph and new entry form.

#### About page

    <header>

    <explanation of Web site purpose>
    <safe contact info>

#### Transfer page

    <header>

    Import or Export data
    <explanation of expected/generated csv format>
    <sample data in csv format>
    <local file selection and submit option>
    <option to download all data>

#### Shared access link page

    <header>

    <table of shared access links>

#### Admin users page

    <header>

    <switch to enable/disable new registrations>
    <table of users>

## Testing

There will be unit testing of all the Javascript logic placed in the page. If
possible, Supabase Edge function code will be unit tested as well.

## Shared view session details

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

## Supabase integration details

Store Supabase access tokens and database passwords in GitHub Actions secrets or
environment secrets.

Use different Supabase project URLs and publishable keys for each environment.

We will use the Supabase CLI to create and manage the schema. Make each
migration reversible by convention, keeping the forward and reverse SQL
together, for example:

    migrations/
      001_create_customers.up.sql
      001_create_customers.down.sql

Supabase's standard migration workflow doesn't automatically recognize .up.sql
and .down.sql pairs, so we need to manage that convention.

Take advantage of Supabase backups before production migrations.

## Terraform details

The state files for both dev and prod will be stored in HCP Terraform, using a
free account. HCP Terraform will also be used to plan and apply. These actions
will be orchestrated from a GitHub workflow. The HCL will include a
`{ cloud { ... } }` block to delegate command execution and state management to
HCP Terraform remotely. The workflow will install terraform in the runner and,
then use standard commands like terraform init, terraform plan, and terraform
apply -auto-approve.

Each environment will have its own HCP workspace: cardio-dev and cardio-prod.

## Open issues

- The requirements are intentionally vague with respect to user registration and
  authentication, both the mechanism and the UI. I’m hoping to find support for
  this from Supabase or some other 3rd party, something with little impact on
  the Cardio schema. Otherwise, we will have to define all the usual
  requirements, like register, sign in, sign out, change password, forgot
  password.
- Should I use a static Web site builder, like Eleventy?
