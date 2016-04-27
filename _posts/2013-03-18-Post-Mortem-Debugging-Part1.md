---
layout: post
title: Designing to Support Post-Mortem Debugging, Part 1
author: E. Harold Williams
author-image: hwilliams.jpg
summary: Programs have bugs. There is extensive literature about how to avoid shipping bugs with a system and we, at Switchfly, do our best to follow modern practices. To improve our code quality we use specification reviews, pair programming, code reviews, unit testing, integration testing, and robot driven, web based end-to-end testing in addition to a highly qualified QA team doing manual testing.

---

## Programs have bugs

There is extensive literature about how to avoid shipping bugs with a system and we, at Switchfly, do our best to follow modern practices. To improve our code quality we use specification reviews, pair programming, code reviews, unit testing, integration testing, and robot driven, web based end-to-end testing in addition to a highly qualified QA team doing manual testing.

Sadly, we still release bugs.

Knowing that there will be bugs in production, we designed and built our system to proactively alert us of certain classes of bugs and to record as much information as possible to facilitate quick diagnosis.

There are two aspects of our system that complicate debugging production issues:

* Our system is highly configurable. With a large number of configurations, it becomes impossible to test the system with every combination of possible configurations. Our clients can configure the system in ways we had not anticipated, leading to unusual paths through our code.
* We interact with a large number of external systems. Indeed, one of our core competencies at Switchfly is our ability to connect and interact with a wide range of external systems. However, being out of our control, these systems frequently return results that we had not planned on or seen before, either because of edge cases we had not coded for, or because they have changed their system behavior without notifying us.

Coupled with the complex and evolving business rules of the travel industry, this makes it vital for us to have as much information as possible when researching issues we or our clients discover.

## Dealing With Bugs That Manifest as Exceptions

One category of bugs we see is runtime exceptions, such as,

* Exceptions thrown by the language (null pointers, indexes out of range, etc.)
* Exceptions thrown by libraries (network connection lost, database constraint errors, etc.)
* Exceptional conditions detected by our code

There are three major “best practices” we have developed at Switchfly for dealing with exceptions in a way that enables us to be alerted to production issues and have the information we need to deal with them.

## Record Exceptions When They Are Caught

We record an exception in a database table when we catch an exception. There were two contending strategies we considered when designing our exception framework. One was to record exceptions at the highest level. For example, if web page creates a thread and you could have a try/catch block around the whole thread that caught and recorded exceptions. Then the rule would be that you must either rethrow the exception or record it. 

We chose the opposite approach, where we record an exception as soon as it is caught. This seemed a simpler approach with which it was easier to enforce correctness. Every catch block in our system should have code that records the exception. If we re-throw an exception, we know it has been recorded, but also carry along information so that callers can add information to the database if desired.

Since the goal of recording these exceptions is port-mortem debugging, we record extensive data (nearly 30 attributes) with each exception. Some examples of the data we record are:

* The timestamp
* The stack
* The name of a log file, if it exists. I will talk more about log files in the next blog.
* The machine name (an important clue if an exception is happening on just one or a few machines in a large server farm)
* if an agent was logged in (and who he was) or if this was triggered by a customer
* The name of the external system with which we were communicating and it’s error message
* An error code and error category along with error parameters
* If this error was displayed to the customer using our site and, if so, the error message that was presented.
* Details about browser hitting the site, the referrer, and the history of pages in the current session that led to this error.

This provides us with a repository of exceptions that have occurred in a running system. Some errors do not even manifest themselves to customers because we can recover from the condition, but this strategy guarantees that it is recorded so that condition will be investigated and fixed..

h2. Code To Detect Exceptional Conditions

We encourage our programmers to code defensively and proactively look for unexpected conditions and throw exceptions rather than continuing. We would suggest that the following code, for example:

```java
switch (status) {
  “success”:
    doSomething();
    break;

  “failure”:
    doSomethingElse();
    break:
}
```

should be written as follows:

```java
switch (status) {
  “success”:
    doSomething();
    break;

  “failure”:
    doSomethingElse();
    break:

  default:
    throw createException(“This is impossible, but it happened”);
}
```

This is a very simple example, but programs are full of instances where there are implied pre-conditions. Making these assumptions explicit makes the program more robust and makes violations more visible. In the example above, if we wanted to simply log the error, but continue processing, we would remove the “throw”. The “createException()” call logs the exception in our database, so it will show up in reports.

## Provide Tools To Mine Exception History

The screen below shows our tool that allows a report to be generated for errors that have been recorded in our database. The top ¾ of the form allows parameters to limit the errors that are displayed in the report.

![Admin Errors - Search 1](/images/admin-errors-search1.png)

The “Group By” and “Results per page” control how the report is displayed. 

![Admin Errors - Search 2](/images/admin-errors-search2.png)

The “Comparison” drop down allows the report to compare the error counts for the specified date range to be compared to previous weeks. This is very valuable when monitoring errors after a release, for example.

![Admin Errors - Search 3](/images/admin-errors-search3.png)

Finally, you can drill in to a specific error to see its details and stack.

!![Admin Errors - Search 4](/images/admin-errors-search4.png)

## Summary and Looking Ahead

The practices we follow to enable post-mortem debugging with exception errors are:

* Record exceptions when they are caught.
* Code to detect exceptional conditions
* Provide tools to mine exception history

In my next blog post I will describe how we record enough information to investigate errors that don’t throw exceptions. Read on to "Part 2":http://blog.switchfly.com///article/Post-Mortem-Debugging-Part2
