# Newman 2.x User Guide

## Overview
Newman 2.x is one component within the overall CCCPO program.  It provides a management system that allows Comcast personnel to create and maintain sets of customers associated with campaigns.  It also provides a scheduling system for dispatching TV alerts to those customers.  Finally, Newman tracks the participation of customers in campaigns and allows remote systems to indicate when customers have completed the campaign workflow.

## Terms and concepts
**account** - a Comcast subscriber's identity within the billing system.  Newman assumes that account numbers are globally unique.

**alert** - a single HTTP transaction sent to TVNS.  Newman is unaware of any downstream infrastructure required to dispatch an alert to a device (e.g. settop).  See https://commons.cable.comcast.com/docs/DOC-15699 for more information.

**campaign** - a named list of Comcast customers and their devices.  Newman assigns a **UUID** to each campaign, and that **UUID** is the campaign identifier.

**CCCPO** - **C**omcast **C**ustomer **C**ampaign **P**rogram **O**perations.  The overall Comcast initiative to which Newman belongs.

**CSV** - **C**omma **S**eparated **V**alues.  A text file format described by https://tools.ietf.org/html/rfc4180.  Although this file format primarily uses commas (,) as field delimiters, other delimiters are supported.  Specifically, Newman supports commas (,), tabs (\t), and pipes (|) as delimiters.  Regardless of the delimiter used, Newman uses the term **CSV** to refer to this file format.

**device** - hardware supplied by Comcast to customers.  Newman does not expect that a device is a settop box.  Rather, Newman's only requirement for a device is that it has a MAC address.

**fulfillment** - Newman uses this term to describe the end of a Comcast subscriber's participation in a campaign.  Newman captures both a date and a code and stores this information on each **ticket**.  Newman imposes no particular set of fulfillment codes.  It is up to remote systems to supply these codes and to interpret their meaning.  Newman simply records these codes for reporting and to indicate when tickets should no longer be used when alerting.

**inventory** - the set of all accounts and devices that Newman knows about.  Newman tracks what it knows about inventory via **tickets**.

**schedule** - a timetable for sending alerts for a specific **campaign**.  Newman allows zero or more schedules per campaign.

**ticket** - tracks the participation of a single Comcast subscriber in a given campaign.  Each ticket is associated with exactly 1 **campaign** and 1 **account**.  Additionally, each ticket may have one or more **devices**.

**UUID** - **U**niversally **U**nique **ID**entifier.  Newman generates Version 4 UUIDs for various domain objects.  See https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_4_.28random.29.  Additionally, Newman formats these identifiers using the RFC4648 variant described at http://tools.ietf.org/html/rfc4648#page-8.  However, since several downstream systems have problems with hyphens (-), Newman replaces those with periods (.).

## Installation
Newman 2.x is installed as an RPM.  The various versions of RPMs can be found in the TeamCity release repository at http://nexus.cvs.ula.comcast.net:8081/nexus/index.html#nexus-search;quick~newman-rpm.

### Requirements
* Newman has been tested on both CentOS 6 and 7.  CentOS is the supported flavor of Unix for running Newman.
* Newman requires the JDK 1.7 or higher.  Installation of the Java runtime is not part of Newman's RPM.
* Newman requires an Elasticsearch 1.7 (or higher).  The template for Newman's Elasticsearch index is supplied in the installation RPM.  Refer to Elasticsearch's user guide for detail on installing this template.

## Security
Newman requires users to log in with their Comcast Cable user credentials.  Newman does not store password information; rather, it delegates to the Comcast Corporate Active Directory to authenticate.

## Importing Data
Newman allows users to upload information to create and update tickets.  Newman can ingest this data as **CSV** files.

There are a few common format rules for any **CSV** file that Newman ingests:

1. Empty lines or blank lines (i.e. lines that consist only of whitespace) are permitted.
2. The pound sign (#) indicates a comment.  A comment must be the only thing on its line.  Field names or data and comments cannot be on the same line.
3. A header specifying the field names for each line is required as the first non-empty, non-blank line.  This header specifies the order of fields for each data line that follows.
4. Field names are case insensitive.
5. Each non-empty, non-blank line after the header is assumed to contain one or more fields.  The fields must be in the same order as the header.  However, there can be fewer fields than the header, in which case Newman assumes the remaining fields have a blank value for that line.

There are two flavors of importing that Newman supports.  First, _inventory_ files contain metadata about devices and associated accounts that participate in campaigns.  Second, _fulfillment_ files inform Newman that accounts are no longer participating in campaigns and the reasons for the discontinued participation.

### Common fields for both inventory and fulfillment

**ACCOUNT_NUMBER** - the Comcast billing account number.  This field is always **required**.

**DEVICE_ADDRESS** - The MAC or unit address.  This field is required for inventory, but is optional for fulfillment.

**CAMPAIGN_ID** - The campaign identifier.  This field is always required.  However, for inventory, if the user is uploading via the campaign upload form, this field can be omitted since Newman will assume its value is the identifier of the campaign shown in the web interface.

### Inventory
Once a campaign has been created, a user may upload data to Newman via the campaign interface.  Each line of a **CSV** inventory file represents a single **device**.  To specify multiple devices for a **ticket**, supply (1) line for each **device** and repeat the account-related fields.

Any fields that Newman does not recognize in an _inventory_ file are stored as attributes on the corresponding tickets.  This allows arbitrary metadata, such as division or market, to be stored within Newman.  These attributes can then be used when scheduling alerts to refine the customers that are alerted.  Because of this behavior, Newman does not have the notion of fields that are specific to _inventory_ files, as any field Newman encounters will be stored on the appropriate tickets.

### Fulfillment
A RESTful endpoint for updating tickets is supplied for the purpose of allowing remote systems to update tickets that have been fulfilled.

#### Fulfillment-specific fields

**FULFILL_DATE** - The ISO8601 date on which fulfillment occurred.  If this field is omitted or is blank, Newman assumes its value is the current date.

**FULFILL_ACTION** - The textual reason for fulfillment.  This text can be any value, e.g. a code like IVR or a phrase like "Phase 1 Complete".  Newman does not impose any special values on this field.  All that matters to Newman is that there is a non-blank value for this field to indicate that a ticket should no longer receive alerts.  If this field is omitted or is blank, Newman uses a default value of "newman".

## Sending alerts
Newman implements a scheduling system which automates the sending of alerts.  Newman sends these alerts to TVNS via CodeBig.

** IMPORTANT: Newman does not know about downstream hardware and software, such as MDS.  Newman cannot deduce errors from any other systems except CodeBig and TVNS. **

### Scheduling
Newman allows each campaign to have zero or more schedules.  Each schedule is a timetable for sending alerts to the accounts associated with that campaign.  Schedules come in two basic flavors.  **If no active schedules are associated with a campaign, Newman will not send alerts for that campaign.**

_Recurring_ schedules can be set to occur at a certain time each day, optionally excluding certain days of the week.  "Every day at 6:00pm", "Every weekday at 4:35am", and "Monday at 7:30pm" are examples of supported _recurring_ schedules.

_One-time_ schedules can be used to send alerts once at some point in the future.  "January 7th at 4:50pm" is an example of a _one-time_ schedule.

### Customizing the number of alerts per schedule

Newman requires a limit on the number of customers per run of a schedule.  Anytime when sending alerts, Newman will send alerts to no more than this number of customers.  However, since a customer may have more than one device, the actual number of alerts sent to TVNS will likely be higher than this value.

A _reminder period_ can be set on any campaign which is global for all schedules associated with that campaign.  Newman uses this value to determine how often to resend alerts.  For example, if a _reminder period_ of 2 days is used, then a _recurring schedule_ set to run every day will only resend alerts to customers that did not receive alerts on the previous day.  The _reminder period_ value also applies to _one-time_ schedules.  If no _reminder period_ is set, then Newman never resends alerts to customers.

### Retries

Alerts may fail for any number of reasons.  Newman will retry each alert a maximum of twice, which is a globally configurable option, before giving up and considering that alert to be a failure.

### Who gets alerted?

When it is time for Newman to send alerts, Newman makes an effort to satisfy the customer limit associated with the schedule.  First, Newman attempts to alert customers that have never been alerted.  Next, if the customer limit has not been reached, Newman will attempt to resend alerts to customers, subject to the _reminder period_.
