- Start Date: 2025-08-01
- RFC PR: [#104](https://github.com/inveniosoftware/rfcs/pull/104)
- Authors: Jakob Miesner

# Library KPIs

## Summary
Add endpoints to InvenioILS for KPIs about the performance of the library and extend Invenio Stats.

## Motivation
The CERN Library would like to have two dashboards displaying its KPIs for two different user groups:
 
1. Internal Dashboard
This dashboard is used internally by the librarians to measure the performance of certain processes.
E.g. when a patron requests a document, how much time passes between the patron issuing the request and the patron receiving the book.
It allows the librarians to react to find bottlenecks and track the evolution of the library's performance over time. 

2. Outsider/Stakeholder Dashboard
The Audience of this Dashboard is not the librarians, but rather the patrons and management.
It displays simpler KPIs that show if the library is working well, while being less detailed and technical than the internal dashboard.


## Detailed design
The package `invenio-stats`, which is already used by `invenio-app-ils` will be used to implement the KPIs.
The RFC only covers how api endpoints for the KPIs can be implemented in `invenio-app-ils` and how `invenio-stats` needs to be extended to support the required queries.

In the following, we will specify the requested KPIs and split them up into individual queries, which will be implemented with `invenio-stats`.
`invenio-stats` allows for all queries to be filtered based on a time range specified in the HTTP request (using `start_date` and `end_date`).
Splitting up the requested KPIs into individual queries allows for more flexibility in how the KPIs are later displayed.


To implement some of the KPIs we require stats that are fetched in a periodic fashion (e.g. the evolution of loanable items over a time period).
All previous ILS stats are based on events, e.g. "a record is viewed or "an e-item is downloaded".
Queries that can be implemented by listening to signals are marked with "event: \<description of the signal/event that produces this stat\>".
Queries that are not based on events, but collected in a fixed periodic fashion, are marked with "requires: periodic-event-stats". 
The feature of periodic stats is described in more detail at the end of this section.


By the design of `invenio-stats`, all stats are aggregated.
Currently, this aggregation is always done over a certain `field` (a field to group the documents in the events index by).
Some of our KPIs do not have such a `field`, as all documents should be grouped together.
We thus introduce a change to the `invenio-stats` aggregator `StatAggregator`, which allows for global aggregation.
Queries that require this change are marked with "requires: global-aggregation" and the feature is described in more detail at the end of this section.

### The specific KPIs are implemented as follows
1. Turnover rate of the Library collection:
   1. number of new loans / number of loanable items
       - query - number of new loans:
         - implemented with KPI 3.1 or 4.4
       - query - number of loanable items:
         - requires: 
           - periodic-event-stats
           - global-aggregation
         - aggregate:
           - count of items that have `status`: `CAN_CIRCULATE`
           - daily
           - over no field
       - NOTE: this KPI is described in ISO 11620:2023 A.2.1.1
   2. number of renewals / number of loanable items
       - query - number of renewals: 
         - requires:
           - global-aggregation
         - event: loan renewal
           - listen to existing signal `invenio_circulation.signals.loan_state_changed`
           - create event when renewal, indicated by the transition to `state` `ITEM_ON_LOAN` from `state` `ITEM_ON_LOAN`
         - aggregate:
           - count loan renewals
           - daily
           - over no field
       - query - number of loanable items:
         - see 1.1


2. Average duration of loan:
     1. Average loan durations := loan duration / number of completed loans
          - query - loan duration (= Loan start date - Loan end date)
            - requires:
              - global-aggregation
            - event: loan ends 
              - use signal `invenio_circulation.signals.loan_state_changed`
              - create event when transition is to the `state` `ITEM_RETURNED`
            - aggregate:
              - sum of `end_date - start_date`
              - monthly
              - over no field
         - query - number of completed loans:
           - requires:
             - global-aggregation 
           - event: loan ends  
             - use signal `invenio_circulation.signals.loan_state_changed`
             - create event when transition is to the `state` `ITEM_RETURNED`
           - aggregate:
             - count
             - monthly
             - over no field


3. Availability of requested documents:
     1. Loan creations:
         - query: number of loan creations
           - event: loan creation
             - still unsure how to do this. Look [here for options](#kpi-31---extracting-for-loan-creation-method)
           - aggregate:
             - count
             - daily
             - over composite field `loan_creation_method__document_availability_during_loan_creation`:
               - `loan_creation_method` is an Enum with the following values := 
                 - `self_checkout`
                   - The document is directly borrowed through self checkout
                 - `manually`
                   - the loan was created through the loans page in the backoffice or a patron through the frontoffice (but not self checkout)
                 - `interlibrary_loan`
                   - The loan was created from an interlibrary loan request
               - `document_availability_during_loan_creation` is an Enum with the following values :=
                 - `available`
                   - The loan is done via a loan request and the document is available (one available item with `status` `CAN_CIRCULATE`).
                 - `not_available`
                   - The loan is done via a loan request but the document is not available (no available item with `status` `CAN_CIRCULATE`).
               - copy over the fields `loan_creation_method` and `document_availability_during_loan_creation`, so the query can filter on them individually
                 - e.g. only count loans that were created manually: filter `loan_creation_method=manually`

     2. Average Waiting Time := waiting time between loan (request) creation and loan start time / number of loans started
          - query: waiting time between loan (request) creation and loan start time
            - event: loan starts (use signal `invenio_circulation.signals.loan_state_changed`, indicated by the transition to the `state` `ITEM_ON_LOAN` from any other state)
            - aggregate:
              - sum of waiting time
                - waiting time := loan `start_date` - loan (request) creation date
                  - loan (request) creation date is based on the creation method := 
                    - `self_checkout` and `manually`: use the field `_created` on the loan
                    - `interlibrary_loan`: use either the field
                      - `_created` on the `invenio_app_ils.borrowing_request` that is linked to the loan
              - monthly
              - over field `loan_creation_method__document_availability_during_loan_creation`
                - see 3.1 for definition of `loan_creation_method__document_availability_during_loan_creation`
                  - we have to extract the value of this field from the events index of KPI 3.1, as the information is only valid during loan creation 
                - we also add information about the provider to the event for interlibrary loans, so future aggregations can differentiate the waiting time based on the provider
            - query: number of loans started
              - event: loan starts (use signal `invenio_circulation.signals.loan_state_changed`, indicated by the transition to the `state` `ITEM_ON_LOAN` from any other state)
              - aggregate:
                - count
                - monthly
                - over field `loan_creation_method__document_availability_during_loan_creation`
                  - see 3.1 for definition of `loan_creation_method__document_availability_during_loan_creation`
                    - we have to extract the value of this field from the events index of KPI 3.1, as the information is only valid during loan creation 
                  - we also add information about the provider to the event for interlibrary loans, so future aggregations can differentiate the waiting time based on the provider


4. Number of changes to the Library collections:
    - count number of creations, updates, deletions of all ILS records
      - The following record types were directly requested by the CERN Library, but we will track all record types:
        1. documents
        2. physical items
        3. e-items
        4. loans
      - query: number of creations, updates, deletions of records
        - event: use existing signals from `invenio_records`
          - `after_record_insert`
          - `after_record_update`
          - `after_record_delete`
        - aggregate:
          - count
          - daily
          - over composite field `pid_type__method`:
            - `pid_type` := record pid type (e.g. `docid`, `eitmid`, ...)
            - `method` := C(R)UD method used to modify the record (`create`, `update`, `delete`)
            - example values of the field would be: `docid__create`, `eitmid__update`, ...
            - copy over the `pid_type` and `method`, so the query can filter on them individually
              - e.g. only count creations of documents: filter `pid_type=docid` and `method=create`


5. Patrons:
    1. Number of unique patrons with an activity on their account for a given period (logging in counts as active)
        - query: count distinct patron ids that logged in
          - requires:
            - global-aggregation
          - event: send new signal from `invenio_accounts.views.LoginView::login_user`
          - aggregate:
            - cardinality of distinct patrons (can not use patron id because GDPR)
            - daily 
            - over no field
    2. Loan issued for a period associated with Patron's department
        - query: count loan creations grouped by patron department
          - solved through 3.1
            - add extra aggregation over patron department name
              - is added by event preprocessing and extracted from the field `extra_data` from `invenio_oauthclient.model.RemoteAccount` during the event creation


6. Overdue loans;
    1. Percentage of overdue loans (active overdue loans / active loans)
        - Is already possible with Invenio ILS, but no time series data will be available
          - can be queried from the api by filtering for `is_overdue` and `state`
        - query - number of active loans and active overdue loans:
          - requires: 
            - periodic-event-stats
          - aggregate:
            - number of active overdue and active loans
              - active := `state` is `ITEM_ON_LOAN`
            - daily
            - over field `is_overdue`


7. Purchase orders:
    1. Average Waiting time: waiting time of purchase orders / number of purchase orders that were received
         - query: waiting time of purchase orders
           - requires:
             - global-aggregation
           - event: send new signal from `invenio_app_ils.acquisition.api.Order.update`
             - create event when the `received_date` is set and there is a related literature request
           - aggregate:
             - sum of waiting time:
               - waiting time := order `received_date` - literature request `_created`
             - monthly
             - over no field
               - we also add information about the provider to the event, so future aggregations can differentiate the waiting time based on the provider
         - query: number of purchase orders that were received
           - requires:
             - global-aggregation
           - event: send new signal from `invenio_app_ils.acquisition.api.Order.update`
             - create event when the `received_date` is set
           - aggregate:
             - count
             - monthly
             - over no field
               - we also add information about the provider to the event, so future aggregations can differentiate the waiting time based on the provider
    2. Average ordering time: ordering time of purchase orders / number of purchase orders that were received
         - query: ordering time of purchase orders
           - requires:
             - global-aggregation
           - event: send new signal from `invenio_app_ils.acquisition.api.Order.update`
             - create event when the `received_date` is set
           - aggregate:
             - sum of ordering time:
               - ordering time := `received_date` - `order_date`
             - monthly
             - over no field
               - we also add information about the provider to the event, so future aggregations can differentiate the waiting time based on the provider
         - query: number of purchase orders that were received
           - see 7.1


8. Literature requests:   
    1. Average Efficiency: literature request acceptance waiting time / number of literature request accepted.
         - query: literature request acceptance waiting time
           - requires:
             - global-aggregation
           - event: send new signal from `invenio_app_ils.document_requests.views.py.DocumentRequestAcceptResource.post`
           - aggregate:
               - sum of literature request acceptance waiting time := 
                 - literature request acceptance waiting time := `datetime.now()` when event is created - literature request `_created`
             - monthly
             - over no field
         - query: number of literature requests accepted
           - requires:
             - global-aggregation
           - event: send new signal from `invenio_app_ils.document_requests.views.py.DocumentRequestAcceptResource.post`
           - aggregate:
             - count
             - monthly
             - over no field
    2.  Fulfillment: percentage of accepted vs. declined literature requests (differentiate decline reason 'available in catalogue' from other reasons).
          - Is already possible with Invenio ILS, but no time series data will be available
            - can be queried from the api by filtering for the state and decline reason and looking at the field total of the result
              - e.g.: `<host>/api/document-requests/?q=&sort=-created&page=1&size=15&state=DECLINED&decline_reason=IN_CATALOG`


### Requirements for KPIs

#### Global Aggregation - global-aggregation
Some of our KPIs require aggregating over all documents in the events index, instead of grouping them by a certain field (e.g. "query - number of loanable items")
We thus propose to adapt the `StatAggregator` in `invenio-stats` to allow for aggregations without a field to group by.
This can be done by making the `field` parameter of the `StatAggregator` optional.
If field is set to `None`, the aggregation is done over all documents in the index.

A possible implementation of this can be seen [here](https://github.com/inveniosoftware/invenio-stats/pull/164).

#### Periodic Stats - periodic-event-stats
Some of our KPIs require stats that are not based on events, but rather calculated on a regular basis (e.g. "query - number of loanable items").
We thus propose to add the method `process_periodic_stats` to `invenio-app-ils`, which is called daily by celery-beat.
It receives a list of the names of the stats to be processed and manually emits an event for each of them.


##### config.py
```py
CELERY_BEAT_SCHEDULE = {
    "stats_process_periodic_daily": {
        "task": "invenio_app_ils.stats.event_builders.process_periodic_stats",
        "schedule": crontab(minute=0, hour=3),  # every day, 3am
        "args": [["items-count"]],
    },
}

STATS_EVENTS = {
    "items-count": {
        "templates": "invenio_app_ils.stats.templates.events.items_count",
        "event_builders": ["invenio_app_ils.stats.event_builders.count_items"],
        "cls": EventsIndexer,
        "params": {
            "preprocessors": [
                "invenio_app_ils.stats.processors.add_timestamp_as_unique_id",
            ],
            "double_click_window": 30,
            "suffix": "%Y-%m",
        },
    }
}
```

##### event_builders.py
```py
def process_periodic_stats(stat_names):
    """Process periodic stats."""
    for stat_name in stat_names:
        current_stats.publish(stat_name, [count_items({})])

def count_items(event):
    """Count available items."""
    event.update(
        {
            "timestamp": datetime.datetime.now().isoformat(),
            "available_items_count": current_app_ils.item_search_cls()
            .filter("term", status="CAN_CIRCULATE")
            .count(),
        }
    )

    return event
```

An example implementation of the daily counting of available items ("query - number of loanable items") can be seen [here](https://github.com/inveniosoftware/invenio-app-ils/pull/1246)


## How we teach this

###  Regarding Periodic Stats
We propose the term `periodic` to be added to all code related to this feature (e.g. `process_periodic_stats`).
This makes it clear that these are not event based, but collected on a regular basis.


## Drawbacks

### Periodic Stats
By implementing periodic stats to be added as events, it is easy to run into situations, where invenio-stats always only aggregates one document per time period.
This is because the periodic events in this RFC are aggregations themselves.
The stats `query - number of loanable items` from 1.1 is an example of this.
We count the number of items on a daily basis, resulting in one event/document per day in the events index.
We then later aggregate daily, resulting in one document per day in the aggregations index, which contains the data of only one event.
This is not a problem per se, but can be seen as unclean and a duplication of the data.

### Splitting up KPIs into multiple queries
The KPIs are split up into multiple queries.
While this allows for more flexibility, it also requires the Dashboard to perform some aggregations itself.
This makes it less possible for dashboards that are possibly built in the future on top of `invenio-stats` to display aggregations, to directly display the KPIs, as they have to know how to combine the individual queries into the requested KPIs.
(A possible implementation of such a dashboard was discussed [here](https://github.com/inveniosoftware/product-rdm/discussions/182).)
That said, as the events are stored in `invenio-stats`, it is always possible to create new aggregations for the KPIs in the future, if a dashboard requires it.

## Alternatives

### Not using `invenio-stats`
An alternative to using invenio-stats would be to extend the indices of existing records to include the fields required for the KPIs, such as the `waiting_time` for loans.
This would allow querying the records directly for the KPIs without needing a separate stats index.

However, this approach would introduce many fields to the records indices that do not represent the record itself and serve only as metadata for KPIs.
Additionally, the dashboard would need to send multiple requests to the API, triggering potentially expensive aggregations in the search system.

In contrast, `invenio-stats` already provides partially aggregated data, and a single request can cover multiple queries, reducing the number of API calls and the load on the search system.
### Global Aggregation
We considered creating a new aggregator class `GlobalStatAggregator` that always aggregates over all documents in the index.
But this aggregator would share a lot of code with the existing `StatAggregator`.
Thus, we decided to rather adapt the existing `StatAggregator` to allow for aggregations without a field to group by.

### Periodic Stats
A considered alternative for creating periodic stats would be to create an entire new class of stats in `invenio-stats`.
It involves creating a new indexer class `PeriodicEventsIndexer` that does not consume events from a queue, but rather generates events on a schedule.
This was discarded, as it would introduce unnecessary complexity to `invenio-stats`.

### KPIs

#### KPI 2 and 3.2 - Alternative approach to measure waiting time and loan duration
An alternative approach to measure the waiting time and loan duration would be do count the loans that are respectively waiting or active at the end of each day.
This would give a more direct measure of how many loans are active or waiting over time.
However, this does not allow to compute an average, as it is unsure what divisor to use.

#### KPI 2 and 3.2 - Directly computing the average
For KPI 2 and 3.2, we could also directly compute the average in the stats aggregation, instead of returning the sum and count and letting the dashboard compute the average.
However, this allows for less flexibility in what date granularity the average is computed on (e.g. daily, weekly, monthly).

## Unresolved questions

### Periodic Stats

#### Should the periodic stats safeguard against duplicate entries?
Currently, the periodic stats do not safeguard against multiple events being generated for the same time period.
E.g. if the `process_periodic_stats` method is ran twice in a single day, it will generate two events for the same day.
This can lead to unexpected results, as the aggregations will then aggregate over both events.
A possible solution could be the aggregation of the periodic events always taking the average.
This way, multiple events for the same time period would not change the result of the aggregation.


### KPIs

#### KPI 3.1 - Extracting for loan creation method
It is still not clear what is the best way to listen for loan creations.
A challenge we face is that we have to extract the loan creation method.
The different endpoints called to create a loan are:
  - interlibrary loans `/borrowing-requests/<pid>/patron-loan/create`
  - self checkout `/loans/self-checkout`
  - manual creation `/circulation/loans/request`

We could call a new signal `loan_created` from those endpoints with a parameter indicating the creation method.

Alternatively, we could just listen to the signal `after_record_insert` from `invenio_records`, filter for loans and only during event generation or preprocessing extract the creation method. (Unsure if possible)

#### Median vs. Average
Should the aggregation of the loan duration and waiting time also be done on a median basis?
ISO 11620:2023 recommends the median as a more robust measure for the waiting time.
An additional aggregation could be added.

#### Aggregation period
We aggregate most stats on a daily basis.
An exception of this are the loan durations and waiting times, which are aggregated monthly.
Should we also aggregate those on a daily basis?
In case the median is added as an additional aggregation, this would make it less reliable, as the median of a single day is not very meaningful.
