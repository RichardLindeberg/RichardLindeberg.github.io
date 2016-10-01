---
layout: post
title:  "Eventsourcing part 5 - constraints"
date:   2016-10-01 13:33:55 +0200
categories: Eventsourcing
---
This post will probably not make much sense if you did not start from the [beginning]({% post_url 2016-09-18-EventSourcing %})

Now we have our awesome system where you can create and change a person. Remember from part 2 that an aggregate root (our person) can only check it's own invariants. Now our PO wants has a new requirement, there should only be one person per email.
In a regular CRUD/SQL application, you would add a unique constraint on the Email column. And when saving you could check for the error and let the user know they could not use that email, or that they already exists in the system.

But with ES (Eventsourcing), you can not do that. And remember that I have been rambling on about that the aggregate roots only know about them selfs, so they can't really do any check either. But the command handler could. If the command handler would have access to a read model for existing emails, it could easily check before creating a new person.
I would create a new read model that was designed for this purpose, so that all read models have as few dependencies as possible.

```cs
public class PeopleEmailsReadModell : IPeopleEmailsReadModell
  {
      private readonly Dictionary<Guid, string> _namesAndEmailsDictionary = new Dictionary<Guid, string>();

      public ILookup<string, Guid> ExistsingEmails
          => _namesAndEmailsDictionary.ToLookup(t => t.Value, x => x.Key);

      public void Handle(IPersonEvent evt)
      {
          var created = evt as PersonCreated;
          if (created != null)
          {
              Process(created);
          }

          var emailChanged = evt as IPersonEmailChanged;
          if (emailChanged != null)
          {
              Process(emailChanged);
          }
      }

      public bool EmailAlreadyExists(string email)
      {
          return ExistsingEmails.Contains(email);
      }

      private void Process(PersonCreated evt)
      {
          _namesAndEmailsDictionary.Add(evt.Id, evt.Email);
      }

      private void Process(IPersonEmailChanged evt)
      {
          _namesAndEmailsDictionary[evt.Id]= evt.Email;
      }

  }
```

Then I would add that model as a dependency to the commandHandler.
And just before executing the CreatePerson check that the email doesn't already exists.
All done? Not really, if it is absolutely totally necessary that you don't have duplicate emails this is not sufficient. Since the read models are not in the same transaction as the write, there is a slight chance that two people would be created within a very short timespan and then you could end up with to people with the same email.

And you also have the problem of already duplicate email addresses already in the system. You need to handle them somehow, maybe merge all people with the same emails address? Maybe contact them somehow? Maybe mark the oldest as obsolete?  This problem is the same both both models, you need to have a strategy here.

In the CRUD/SQL way you would need to apply that strategy before updating the database, and then apply the behavioral change. Of course you could do some magic with calculated fields and make the constraint on that field, make sure that the calculated field before today are always different and the once after today are not. But that would add a strange behavior to the system, it would be hidden. And if other systems rely on you having unique emails, no good.

In a event sourced system, the unique email is not at all enforced in the database layer and hence you make no changes there. And also it is easy to do subscribe to events, and you could plug in your strategy on a subscription.
Maybe the strategy is to send an email to that address and make the use choose what to do? Have the user to decide which one of the people are the correct ones. If you would do that strategy with a saga, it would be very easy to model and you could have a timeout so if the user don't make a choice within 7 days some default is applied. Or maybe the strategy would just be to mark the newest one as incorrect, delete it and email the person that they already where registered. Maybe close it with a "PersonDeletedDueToDuplicate" and have the other personId in it, so that systems that rely on our events could take actions accordingly.
