---
layout: post
title:  "Eventsourcing"
date:   2016-09-18 21:33:55 +0200
categories: Eventsourcing
---
If you read tech-blogs, which you obviously do, you have most certainly heard about eventsouring.

If you don’t really recall what it is:
Instead of updating the state in a database, just store the event that happened. Seems strange? Yes, if you are used to SQL server or Document databases it’s totally different.

Most developers have probably written quite a few CRUD applications. You create a Person, then you update the Person and then eventually you delete the Person. Everything you need to know about the person is stored in a table, maybe many tables since we like normalization. Everytime we need to know the state of the person we just select the data from the database, easy peasy.

In Eventsouring you store events instead of state. So first your system emits PersonCreated, that gets stored. Then the user changes the email so EmailChanged is emited and stored. Then the user gets bored and deleted the person, hence PersonDeleted is emited and stored. If you want to know the current state of the Person you must read every event that has been emited and transform the person accordingly with every event until you reach the end. Maybe not se easy peasy, especially not if you just want to open up your favorite SQL manager and look at the objects you have.

So why do I like Eventsourcing?

- A true audit log to go over when the S**t hits the fan.
- Makes it possible to defer important business decisions until we are ready to take the decision
- Forces concurrency issues to be handled

### What does it really mean?

#### A true audit log, means just that.
If you have a normal normalized table structure in SQL server only the latest state is stored. Of course you could add history tables that stores every change that happens. But that will only tell you what values have changed. With no clue on why they changed. And every time you change your objects you need to update the database schema. Of course you could use a Document store and get rid of that issue. But even with history tables it can be quite hard to find out the exact state since we usually like to normalize and soon we do changes to a lot of tables everytime something happens in the system. With eventsourcing we store every statechange in the system, and we have the possibility to give a explanation to each statechange. We could for example have UserChangedEmailaddress and EmailAddressCorrected. The exact same outcome, the email was changed. But from a business perspective not at all the same. If it was corrected maybe we need to resend some emails.

#### Forces concurrency issues to be handles
When using EventSourcing you will only be able to store events to one stream at the time, unless you abuse the tooling. And when you only can have one transaction to one object that boils down to aggregate root pattern. Only one aggregate root can be manipulated at once, every aggregate root is resposible for its own invariants. This might seem like a quite limiting ability, and that is correct. But it also forces you think and shape you system, in a way that is decoupled and prevents many erroneous states.

For example, we have a system that handles people and their addresses. With SQL you would do two tables, one Person and one Address. The Address has a foriegn key to the Person. Now, how do you make sure that the two tables never go out of sync? What if one user changes the address and another user changes the name. Maybe you are very diligent and use optimstic concurrency, and make sure that the Person must be of the correct version when updating. But how do you handle the adress? You should not be able to change that if the Person is in the wrong version. Do you make the update on both tables always? What if we have 34 tables for the same aggregate (Yes,I was once involved in a project that had 34 tables for one aggregate root/entity).

When using eventsourcing you always save all events for the same aggregate to the same stream, and you always check for optimistic concurrency on that stream.

#### Deferring of business decisions
##### CRUD way
It means we can do this super trendy microservices things. If a product owner comes to us and says: I need a system that controls mobile subscriptions that we buy from this third part, you start the project and build a sleek nice UI and a simple logic for handling it.

Then the product owner comes back and says:
"I need to know how often a subscription is let over to someone else".
So you add some more logic and an extra table.
PO is nice and doesn’t complain that he only gets information about the switches that happened in the past.

Then he comes back and says he needs to know how often subscriptions switch between different data rates. Same thing all over. You now start figuring out a perfect schema for handling all changes and add some extra tables and a lot of logic. The system is not so nice and simple anymore.

##### ES way
We store what has happend (events), not current state. Then we can let other systems read those events. That way, we don’t change any logic in our Subscription Service when PO wants his reports, we build a new Subscription Reporting Service.

This service does only handle the things that is important to it. So the first time we only implement handling of the SubscriptionTransfered event. And the second time only the DataRateChanged event.

If PO then wants to know more, like from whom a subscription was transferred, we let it handle SubscriptionCreated event.

Since this is for reporting you probably you use your favorite SQL server, the tables get modelled just exactly the way PO whants his information, since it doesn’t at all interfere with our Subscription Service.

This is whats called a read model. The nice thing is that we can have as many read models as we need, since the read models doesn’t interfere with our Subscription Service. And we can replay the Events from the first day of our Subscription Service.
If we need to tweak a report, just drop the data and replay the events from the begining of time after you changed the logic.


#### But don't just take my word for it.

If you haven’t already seen: [Greg Young – CQRS and Event Sourcing – Code on the Beach 2014](https://www.youtube.com/watch?v=JHGkaShoyNs), he has way more experience then me and is better at explaining.

[But lets coding]({% post_url 2016-09-20-Eventsourcing-Part-2 %})
