---
layout: post
title: Designing to Support Post-Mortem Debugging, Part 2
author: E. Harold Williams
author-image: hwilliams.jpg
summary: The premise of my last post was that production systems will always have bugs and that it is important to design a system that makes it possible to research bugs after the fact. I looked at the practices we used to capture and report on exceptions thrown by a running system.

---

The premise of [my last post](///article/Post-Mortem-Debugging-Part1) was that production systems will always have bugs and that it is important to design a system that makes it possible to research bugs after the fact. I looked at the practices we used to capture and report on exceptions thrown by a running system.

Exceptions are often the easiest kinds of bugs to fix. Some errors don’t throw exceptions. Business logic errors occur either because a specification was incorrect, or because the code incorrectly implemented that specification. Sometimes we find that things break because services we access through an API start returning different results that our code interprets successfully, but incorrectly.

The common thread of these scenarios is that the system does not complain at all, but simply does the wrong thing. Perhaps inventory is not offered that should be available. Perhaps it is offered at the wrong price. In general, these must detected by a human and sent to a developer to investigate.

## The Best Case Scenario - “Here’s how to make it happen”

When told of a bug, a programmer first question is “How can I reproduce that?”. We generally believe that if someone can give us a set of steps to show the errant behavior, we can fix it. For example:

* Search for a room in this market
* Notice that this hotel does not display the required resort fee

Once we have this set of steps, we set about reproducing the bug on our development systems where we can use breakpoints step through suspect portions of the code.

### Cloning a Client’s System in a Development Environment

As described in my last post, however, our system is highly configurable and part of reproducing a bug sometimes requires setting configurations in the proper manner. If we find that things appear to work properly in the development environment, it may be because we have configured our environments differently.

Since our application has thousands of configurations, and since it is not clear, especially to support and QA, which configurations might affect the faulty behavior, the most fool-proof approach is to clone the client database for use on the development system.

However, client databases can be extremely large and the there are both time and space considerations when copying them to a developer's machine. (Security considerations are minor since sensitive data is encrypted using a hardware encryption device that is not available outside the production environment.) To facilitate copying databases, we have used schemas (in the Postgres database) to partition our tables into two major categories.

We have a booking schema, that holds all the booking history of our system. This would include bookings and the series of modifications they might undergo, payment histories, accounting, and various audit trails. The size of the tables in this schema grow rapidly as customers shop and book. A typical client’s database is about 400 GB.

Our other schema holds configuration data. This includes configurations (basically a key/value map), product descriptions, agency and agent definitions, contracts that define how costs should be converted to points in our loyalty module, and many more things that are used to “set up” the system for use. The size of the configuration schema is not insignificant, but dramatically smaller than the booking schema. A typical client may have about 15 GB in the configuration schema.

![Client DB](/images/client-db-img.png)

In general, administrators managing the system make changes in the configuration schema while customers or agents purchasing travel products make changes in the booking schema. Tables in the booking schema can refer to tables in the configuration schema, but not vice-versa. With this setup, it is possible to write scripts that copy just the configuration schema from the database and we have created scripts that create a mirror of the client production system in the developers test environment. 

The ability to clone just the configuration portion of a client database and mirror the client’s environment on a production machine has been key to quickly diagnosing and fixing bugs.

## The Worst Case Scenario - “It happens occasionally, but I don’t know why.”

Our support and QA teams do their best to tell us how to reproduce bugs, but there are times where bugs are intermittent and we just can’t figure out how to reproduce it. To tackle these bugs, we log information about the customer interaction that the developer thinks might be useful for debugging and save it for later. 

Logs are invaluable for intermittent bugs, but also extremely useful for even ordinary bugs. When used to supplement a bug report, the developer can often find details about the scenario that were forgotten or unknown to the reporter.

A couple definitions are useful for talking about how we manage logs.

* A *booking* is what our system is designed to create. Customers search for travel inventory, investigate interesting options, and create a booking that makes their reservations. Bookings can evolve over time as changes are made or payments are recorded.
* A *session* is created when a customer comes to our system to search for travel options. Typically, customers do multiple searches to investigate travel options and occasionally make bookings. Sessions timeout after a period of inactivity.

### What Goes Into a Log?
There are a few rules about what we put into a log file:

* A description of the context for the log (machine name, client name, session key, active agent
* Configurations accessed and their values
* All interactions with external systems (usually XML requests and responses)
* The stack trace of any exceptions

Beyond these, a log’s contents are somewhat ad-hoc. Initially the programmer decides what intermediate states or decision points might be interesting to record. As the system is used, the logging needs become more well understood and additional logging is added and noise is eliminated.

Here is an example of a log entry made at the completion of a booking. Part of booking is optionally sending a confirmation email based on relatively complex logic:

```
2013-03-20 06:03:02,627 WARN  [com.ezrez.web.authorize.AjaxPresenter] /*******************************
2013-03-20 06:03:02,627 WARN  [com.ezrez.web.authorize.AjaxPresenter]  * SENDING confirmation email and push pnr for 1198336 because...
2013-03-20 06:03:02,627 WARN  [com.ezrez.web.authorize.AjaxPresenter]  *    page: /authorize/ajax/get_payment_ajax.cfm
2013-03-20 06:03:02,627 WARN  [com.ezrez.web.authorize.AjaxPresenter]  *    isBookingSuccess: true
2013-03-20 06:03:02,627 WARN  [com.ezrez.web.authorize.AjaxPresenter]  *    isPendingPaymentRedirect: false
2013-03-20 06:03:02,627 WARN  [com.ezrez.web.authorize.AjaxPresenter]  *    isAddTo: false
2013-03-20 06:03:02,627 WARN  [com.ezrez.web.authorize.AjaxPresenter]  *    isDeclinedForFraud: false
2013-03-20 06:03:02,627 WARN  [com.ezrez.web.authorize.AjaxPresenter]  *    isSubsequentPayment: false
2013-03-20 06:03:02,627 WARN  [com.ezrez.web.authorize.AjaxPresenter]  *    hasModificationWorkspace: false
2013-03-20 06:03:02,627 WARN  [com.ezrez.web.authorize.AjaxPresenter]  */
```

This type of logging is useful to developers assigned to research a bug who might not be familiar with this portion of the system and also to support personnel investigating customer requests, e.g., “Why didn’t you send a confirmation email for that booking?”

Pricing is a critical part of our system so we log extensive price information for each component booked. The following snippet of a log file an example. Some of this information is transient and will not be saved in the database but, in case a post mortem investigation is required, it is available.

```
2013-03-20 06:02:44,016 INFO  [com.ezrez.core.model.pricing.PriceServiceImpl] 
Calculated Supplier Price:
	. Currency: USD
	. CurrencyType: SUPPLIER
	. ProductType: ROOM
	. PassThrough: false
	. SoldAt: 315.19
	. TaxTotal: 33.62
	. Cog: 313.87
	. Net: 280.25
	. TotalRounding: 0.0
	. BookingFeeFromPassThroughMarkup: 0.0
	. BaseCommission: 0.0
	. ExtraCommission: 0.0
```

```
Currency Configuration:
	. SystemCurrency: USD
	. SupplierCurrency: USD
	. CustomerCurrency: USD
	. AgencyCurrency: USD
	. ForeignExchangeRateSnapshotId: 0
	. AgencyCurrencyPrecision: 0.01
	. CustomerCurrencyPrecision: 0.01
	. SystemCurrencyPrecision: 0.01
	. SupplierToAgencyExchangeRate: 1.0
	. SupplierToCustomerExchangeRate: 1.0
	. SupplierToSystemExchangeRate: 1.0
	. AgencyToSystemExchangeRate: 1.0
	. CustomerToSystemExchangeRate: 1.0
```

```
Calculated Redeem Price:
	. getContractName: Contract Segment 1
	. getRuleName: Segment 1
	. getAwardDesignator: null
	. getContractToCustomerExchangeRate: 1.0
	. getMaxPoints: 33800
	. getMaxPointsTaxValue: 33.62
	. getMaxPointsValue: 315.19
	. getMinPointsSpend: 5000
	. getOverageFactor: 0.009953102
	. getPercentageFloorPoints: 11500
	. getPointsToCustomerExchangeRate: 0.009328448
	. getShortfallFactor: 0.009798057
	. getTierUpperRangeInCustomerCurrency: 325.0
	. getAllocationMaxPoints: 33800
	. getShortfallNotRoundableInCustomerCurrency: 0.0
	. getShortfallRoundableInCustomerCurrency: 0.0
	. getShortfallTaxInCustomerCurrency: 0.0
	. getRoundingIncrementFromLoyaltyRule: 50
	. getTaxIncludedInConversionFromLoyaltyRule: true
```

```
Intermediate Price Calculations:
	. TotalBeforeTax: 281.57
	. Markup: 1.3199999999999932
	. TaxOnNet: 33.62
	. TaxOnMarkup: 0.0
	. Overprice: 0.0
	. ReductionFactor: 0.0
```

```
CRS Price:
	. ProductType: ROOM
	. SupplierCurrency: USD
	. Net: 280.25
	. Tax: 33.62
	. InclusiveOfAllCharges: true
	. Published: false
	. CanMerchantPublished: false
	. NightlyNets: [154.88, 125.37]
	. NightlyTaxes: [18.58, 15.04]
```

```
Base Pricing Enrichment:
	. CommissionType: GROSS_BEFORE_TAX
	. ContractGroupId: 0
	. Currency: USD
	. FlatCommission: 0.0
	. FlatMarkup: 0.0
	. PercentCommission: 0.0
	. PercentMarkup: 0.0
	. MarkupsAndCommissionsContractId: 373
	. PricingEnrichmentKey: default: OUTSIDE BULK INDIVIDUAL ROOM
	. SourceInformation: 
```

```
Contract Group Pricing Enrichment:
	. CommissionType: GROSS_BEFORE_TAX
	. ContractGroupId: 28
	. Currency: USD
	. FlatCommission: 0.0
	. FlatMarkup: 1.32
	. PercentCommission: 0.0
	. PercentMarkup: 0.0
	. MarkupsAndCommissionsContractId: 379
	. PricingEnrichmentKey: tourico: OUTSIDE BULK INDIVIDUAL ROOM
	. SourceInformation: ContractGroup id = 28; 
```

```
Other Price Calculation Data:
	. ContractGroupId: 28
	. AgencyMarkupMultiplier: 1
	. MaxPriceAfterMarkup: 0.0</code>
```

### How Many Log Files Are There and How Are They Named?

Our logging strategy differs slightly from what I have seen discussed elsewhere. We create lots of log files, one or more for each time a customer accesses one of our web pages. The names of these files tell a lot about what is happening. For example, this log file:

`/var/ezrez/logs/plg01/2013-03-20/session/get_payment_ajax.cfm/0602/2013-03-20-06-02-43.966__get_payment_ajax.cfm__bookSuperPnr__app31.log.gz`

Log files are in path designed to control the size of any particular directory:

* plg01: the name of the database being used when this log was creatred
* 2013-03-20: the date
* session: this log was created as part of a session interaction, as opposed to an API access to our system
* get_payment_ajax.cfm: this names the page that was hit to initiate processing
* 0602: the time, to the minute level
* 2013-03-20-06-02-43.966__get_payment_ajax.cfm__bookSuperPnr__app31.log.gz: The is the actual name of the log file, which repeats a number of the elements in the path and adds a few elements:
    * bookSuperPnr: the name of the business entry point called by the web page. Note that there could be multiple log files created from a single web page hit.
    * app31: the specific machine in the server farm processing this request.

### When Are Log Files Created?

Logs are created based on several factors. Some actions (such as creating bookings) are so important that they are always logged. Some actions are never logged. And some actions are logged conditionally based on what we call “debug mode”. Our sessions have a mode that can be set by Switchfly support to enable debug mode and turn on this conditional logging.

We invented “debug mode” because log files become, at some point, overwhelming. The log files for hotel searches, for example, can end up being megabytes for a single search. Our system goes to multiple hotel sources and each source can return hundreds of hotels. The number of log files in a high volume booking site is giant. However, availability logs are extremely useful when diagnosing problems with our suppliers. Our solution is to conditionally log availability requests only when explicitly instructed to do so.

There are two major gatekeepers in our system that can decide to create a log file. 

* We have a tomcat servlet filter that examines all web page requests and determines if a log file should be created based on the page that is being accessed.
* We use AOP to process calls from our web tier into our business tier and create logs based on the entry point being called.

The combination of these sets of logs gives a good overview of the actions during a session on the way to creating a booking.

### How Do We FInd Pertinent Log Files?

A key for using log files for debugging is finding the one you need. We have done this by tying log files to either a session or a booking. 

When you do a search on our system and create log files, they are saved in your session so you, as a developer or support person, can review them and save the names of logs that demonstrate some issue you are researching. Here is what our site looks like when debug mode is turned on:

![Admin](/images/admin-img.png)

Everything above the black bar is additional information available only to support people authorized to turn on debug mode. Note the “session logs” link, which if clicked, gives you something like this:

[Session logs](/images/session-logs-img.png)

The session logs indicate that we have hit two web pages in this session (arc_waiting.cfm, which hosted a search form, and arc_process.cfm, which processed the search). arc_process.cfm generated a number of business service calls, some of which were in parallel threads, to process the search and ended up 6 discrete log files.

These log files are persistent. However the links to them are transient in the context of the session. If we stopped now, the session would expire and access to these log files would be harder.

### Booking Log Files

However, if you continue and make a booking, it becomes important to have access to the log files related to the booking as long as the booking exists in the system.

When a booking a created, it is linked to all the log files used to create that booking and any log files created as the booking evolves are added to the list. When a support person pulls up a booking, there can see the logs of all the steps of that booking. Often just the names of the log files can be enough for him to get a picture of the life cycle of the booking and any particular log holds a wealth of information when researching issues.

When pulling up a complete booking here is a small subset of the logs that are available for research:

![Log list 1](/images/log-list-img1.png)
![Log list 2](/images/log-list-img2.png)
![Log list 3](/images/log-list-img3.png)

h2. Summary

In this and the previous article I have explained how we, at Switchfly, have designed our system so that we can have the information we need to diagnose and correct bugs in a production system. Our strategies include:

* A framework for handling exceptions that makes sure that all exceptions are logged in a database and tools for mining that database.
* Tools that make it easy for a developer to replicate the key portions of a development system in their local environment for debugging.
* Extensive logging of configurations, intermediate states, decision points, external communications and exceptions into individual log files associated with a phase of a particular booking coupled with tools to make it easy to access the log file related to the issue at hand.

We have found these practices important in our business and I hope there are some ideas in here that you will find new or interesting.

