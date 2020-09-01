# Aggregated Reporting API

This is a proposal for a new Web Platform API that allows for collapsing
information across multiple sites into a single, privacy preserving
report. This is made possible by a write-only per-origin data store that
flushes data to a reporting endpoint after reaching aggregation
thresholds across many clients. That is, data is only reported if it is
sufficiently aggregated across browser users, using a server-side
aggregation service.

Note: We are still working on ideas for how to make aggregate
measurement work. This is just a strawman interface proposal. Most
likely an API like this would be backed by something like the
[Aggregation Service](https://github.com/WICG/conversion-measurement-api/blob/master/SERVICE.md)
proposal.

Motivation
==========

This API, like the [Conversion Measurement
API](https://github.com/csharrison/conversion-measurement-api),
intends to provide a well-lit path for ads measurement without needing
to use consistent cross-site identifiers like third party cookies. This
allows ad measurement in a much more privacy preserving manner.

In this case, the API provides critical functionality to measure the
*reach* of a particular ad campaign (how many distinct users saw the
ad). This functionality is useful for other types of third party widgets
as well.

Simple Sample Strawman Usage
============================

The API is built around a Javascript layer similar to `localStorage`. It
explicitly allows third party iframes to use this for third party
storage.

Example: Reporting total ad views for a campaign
------------------------------------------------

**On every impression related to campaign-123:**

```
// Add an entry to the storage with id 'campaign-123' if it doesn’t
// already exist.

const entryHandle = window.writeOnlyReport.get('campaign-123');

// Each entry supports simple string kv pairs. If the attribute is
// already present, it will be overridden. This can be used e.g.
// for demographic slices.
entryHandle.set('country', 'usa');

// Entry attributes support appends. If the attribute does not exist, is
// equivalent to |set|. This allows us to count in unary.
// Multiple visits will look like “11111..."
entryHandle.append(“visits”, “1”);

// Entries can be configured to report after a given time. After the
// time has passed, entries are queued for reporting, become immutable,
// and removed from the entry table. The UA will add additional
// randomized delay on reporting for privacy reasons (up to a e.g. day).
// After a report is configured for reporting it cannot be altered by
// subsequent calls to reportAfter.
entryHandle.reportAfter(2 * kMsecPerDay);

// Entries can optionally be set to expire without reporting if
// reportAfter is not called. All entries have a default expiry
// of seven days, with a max expiry of a month.
entryHandle.expireAfter(7 * kMsecPerDay);
```

This snippet will end up sending the following report, assuming the
aggregation service has seen > T identical reports:
```
{
 'entryName': 'campaign-123',
 'country': 'usa',
 'visits': '1'
}
```
Using this data on the server side, ad tech can find distributions of ad
views across all their users, for a given reporting window.

Example: Reach measurement for an ad campaign
---------------------------------------------

The reach of a campaign is the number of unique clients that saw
impressions for it. This can easily be done with this API, via keying
aggregated reports off of a campaign id.

**On every impression related to campaign-123:**
```
const entryHandle = window.writeOnlyReport.get('campaign-123');

// Add any demographic slices you want or know in the current
// context.
entryHandle.set('country', 'usa');

// Add a date field, so there’s no confusion with regard to
// reporting delays.
entryHandle.set('date', new Date().toDateString());

// Every night, queue a report per user that saw the campaign at
// least once, with their demographic information, as long as
// there are enough identical reports.
entryHandle.reportAfter(msecFromNowUntilMidnight());
```

Example: Number of different domains a 3p widget is encountered on, per user
----------------------------------------------------------------------------

### If you can recognize the same user over time on the same domain:

**On every impression related to widget-123:**
```
const entryHandle = window.writeOnlyReport.get('widget-123');

// Filter out repeat views on this domain using first party state.
if (!haveSeenWidgetOnThisDomainSinceLastReport('widget-123')) {
  entryHandle.append('distinct-domains', '1');
}

entryHandle.reportAfter(2 * kMsecPerDay);
```

### If you cannot recognize the same user over time on the same domain:

Some browsers may limit read/write access to storage inside cross-domain
iframes, making the filtering from the previous solution unavailable.
Here is another technique.
```
const entryHandle = window.writeOnlyReport.get('widget-123-domains');

// Record this domain

entryHandle.add('viewed-on-' + document.location.ancestorOrigins[0],
                '1');

entryHandle.reportAfter(2 * kMsecPerDay);
```

Because of the thresholding requirement, this will only generate reports
about sufficiently common sets of domains. If your widget is embedded in
too many different domains for that to be useful, consider replacing
each domain with e.g. `hash(domain) % 100`.

Advanced Example: Calibrating a frequency capping model
-------------------------------------------------------

“Frequency capping” an ad campaign is a feature in which a given ad
campaign can be shown to a particular person only a certain limited
number of times in some time period. (Both advertisers and people who
see ads are happier if they don't see the same ad too often as they
browse the web.)

Typically this is done via third party cookies, to keep track of a
views-per-ad count across publisher sites. However, when third-party
state is unavailable or otherwise undesirable, this feature could be
approximated by using per-publisher frequency caps, along with some
global, aggregated data for calibration.

Suppose you wanted to model frequency caps as:

```
fcap_domainX = fcap_global *
  (typical #ads seen per user on domain X / 
   typical #ads seen overall, for users who visited domain X)
```

Then this API could provide the distribution you need for the
denominator of that fraction, for every domain where (enough) users see
the ad campaigns:

```
const thisDomain = document.location.ancestorOrigins[0];

const thisDomainMod = someHash(thisDomain) % 100;

for (let i = 0; i < 100; i++) {
  const entryHandle = window.writeOnlyReport.get(
    'report-for-domain-mod-' + i);

  entryHandle.append('ads-seen-on-all-domains', '1');

  entryHandle.expireAfter(msecUntilMidnight() + 5 * kMsecPerMinute);

  if (i === thisDomainMod) {
    entryHandle.set('this-domain-is-' + thisDomain, '1');
    entryHandle.append('ads-seen-on-this-domain-mod', '1');
    entryHandle.reportAfter(msecUntilMidnight());
  }
}
```
This will result in daily reports showing a site someone visited plus
the total number of ads they saw across all sites. There will be some
reports from users who visited multiple sites with the same
hash-mod-100, which can be recognized by having multiple
"this-domain-is-..." lines in the report; the number 100 can be changed
if these collisions cause too many problems.

Reporting
=========

At specific time intervals, the browser will queue all entries in
storage for reporting. The API itself must ensure somehow that the
revealed data is private in some way. This could be achieved by
integrating with the [Aggregation Service](https://github.com/WICG/conversion-measurement-api/blob/master/SERVICE.md)
proposal in some way, by forming aggregation keys from the serialized
reports.

Restrictions for Performance and Privacy
========================================

Limit on number of pending reports
----------------------------------

Pending reports take up storage on the client’s device, so there should
be some limits on the total storage this API can use per origin.

Limit on number of reports per time period
--------------------------------------------------

Additionally, some restriction on the number of reports per time period
seems reasonable, to put a limit on the rate of data about any user that
can be learned.

Note that for a use case like reach measurement, the demand for this API
is at least the number of ads seen per time period per origin, if every
campaign wants to implement this kind of measurement.
