---
layout: post
title:  "Eventsourcing show me the code!"
date:   2016-09-20 21:01:55 +0200
categories: Eventsourcing
---
I will start with explaining how I would go about creating a new Eventsourced system.
First of I would make a marker interface for events:

```cs
public interface IEvent
{
}
```

This makes it easier to distinguis events from other types. And if you are to user somthing like nServicebus later it is a good thing to have.
Then lets make a another marker interface so that we can make different events belong to a certain object.

```cs
public interface IPersonEvent : IEvent
{
    Guid Id { get; }
}
```

Every event that is going to operate on our person is needs to be clear about what person it operates on, so make sure it has an Id.
Then lets create the personCreated event.

```cs
public class PersonCreated : IPersonEvent
{
    public PersonCreated(Guid id, string name, string email)
    {
        Id = id;
        Name = name;
        EMail = email;
    }

    public Guid Id { get; }

    public string Name { get; }

    public string EMail { get; }
}
```
You should make them with equality comparer and imutable but in C# it takes up to much space, so it's only imutable here.

Now we have an event that can be handled by our object. But we need and object!

```cs
public class Person
{
    public Person(IEnumerable<IPersonEvent> events)
    {
        foreach (var personEvent in events)
        {
            Apply(personEvent);
        }
    }

    public Guid Id { get; private set; }

    public string Email { get; private set; }

    public string Name { get; private set; }

    private void Apply(IPersonEvent @event)
    {
      Handle((dynamic)@event);
    }
    private void When(PersonCreated @event)
    {
        Name = @event.Name;
        Email = @event.EMail;
        Id = @event.Id;
    }
}
```

The constructor takes an enumerable of the corrent eventtypes.
For each event it builds the state by calling ApplyChange
Apply casts to dynamic so that the correct When function gets called.
I have an When function for each and eventtype I Expect.
The When function is the function that sets all the properties.
Of course you could use something fancier then dynamics to make sure the right when function gets called.
There are plenty of examples out there that uses reflection and so on, you might wanna use that in production.


I tend to follow aggregateroot thinking, or atleast my understanding of it. So my object has the following critierias:

* It can only operate on itself, does only know about it self
* It must protect it self from ending up in the wrong state.

So lets add a function to get it in to a starting mode.

```cs
private readonly List<TEvent> _uncommitEvents = new List<TEvent>();

public void Create(CreatePerson createPerson)
{
    if (Id != Guid.Empty)
    {
        throw new InvalidOperationException("Cannot create on an already existing Person");
    }

    if (string.IsNullOrEmpty(createPerson.Name))
    {
        throw new ArgumentException("Name is empty");
    }

    if (string.IsNullOrEmpty(createPerson.Email))
    {
        throw new ArgumentException("Email is empty");
    }

    var evt = new PersonCreated(createPerson.PersonId, createPerson.Name, createPerson.Email);
    ApplyChange(evt);
}

public void ApplyChange(TEvent paymentEvent)
{
    _uncommitEvents.Add(paymentEvent);
    Apply(paymentEvent);
}
```

But hey, thats two functions and a field! Yes thats correct. But whe only need to add them the first time.
The CreateFunction takes in a command, checks that the person is not already set up (and should do all other check!)
then it applys the change to the object. Since we want the exact same thing to happen when we run the create function and when we reply the eventstream, we set the properties in the When(PaymentCreated evt) function.
But I also want to be able to save the yet uncommit events to a list so that when the object gets saved it save all my events.

So, yet it's a very complicted way to make a object that we can create with an Id, Name and Email. And we cant even save it. To operate on our objects we will need some kind of repository. In this example project we will use commands and commandhandles to operate on the objects. the commandhandler will accept a command, find the object and apply the command to it. It will only ever find the objects by it's id. So lets create a simple repository.

```cs
public class PersonRepository
{
    private readonly IStoreEvents _store;

    protected PersonRepository(IStoreFactory storeFactory)
    {
        _store = storeFactory.GetStore();
    }

    public string StreamName => "Person";

    public void ActOn(Guid id, Guid commitId, Action<Person> act)
    {
        using (var stream = _store.OpenStream(StreamName, id))
        {
            var events = stream.CommittedEvents.Select(t => t.Body).OfType<IPersonEvent>();

            var person = new Person(events);
            act(person);

            foreach (var @event in person.UncommitEvents)
            {
                var evt = new EventMessage { Body = @event };
                stream.Add(evt);
            }

            stream.CommitChanges(commitId);
        }
    }
}
```
This repository uses NEventstore to store the events. To be able to use NEventstore optimistic concurrency we use the same stream. That means that in a multithreaded environment where to commandhandlers execute on the same object,
Only the first one will be able to commit. The second one will get a concurrencyexception.
The CommitId is used to make sure we don't act on the same thing twice. If we in every command give the command a specific id to identify the command it self, we can use rabbit mq or simular that has AtLeast once delivery garantee. I will write a post about that later, but for now, just accept it as a fact.

So let's create a command and a commandhandler.

```cs
public class CreatePerson
{
   public Guid CommandId { get; set; }

   public Guid PersonId { get; set; }

   public string Name { get; set; }

   public string Email { get; set; }
}

public class CreatePersonCommandHandler
{
   private readonly PersonRepository _repository;

   public CreatePersonCommandHandler(PersonRepository repository)
   {
       _repository = repository;
   }

   public void Execute(CreatePerson command)
   {
       _repository.ActOn(command.PersonId, command.CommandId, person => person.);
   }
}
```
