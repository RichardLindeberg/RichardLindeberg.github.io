---
layout: post
title:  "Eventsourcing part III"
date:   2016-09-21 21:50:00 +0200
categories: Eventsourcing
---
Want to read from the [Beginning]({% post_url 2016-09-18-EventSourcing %}).

So now we have a very nice Person which we can Create, only some nice classes/interfaces involved.
Do the same with Entity Framework, probably much easier. So why do we do it this way?
In the following posts I will try to show why, but first we start by adding some more features.

Now our Product Owner wants the ability to change the name and the email. When the end user changes the email it can be done either because they misspelled it, or because the changed the email.
There will be to different buttons or work flows in the UI for this. For some odd reason they are only allowed to correct a miss spelled email if the account is no more then 10 days old. O well, PO knows best so just implement it.

So we will need 3 new Commands and 3 new Events. But for the email I will create a common Interface that they inherit. That way will not need to handle the two ways of changing email any different when we are rebuilding our state, since it's only affecting the way we we change it.

I won't show all code here but at the end of the post there will be a link to a git hub repository that with full source code for this post. I also added some folders since we are getting more then just a few events and commands.

Next up we add functions in the Person to handle the email commands.
They are pretty similar, we first check that the command really is meant for this object.
In the CorrectPersonEmail command we make sure that the person was created not more then 10 days ago.
(Ohh and yes, I did a change from the last code and added CreatedAt in the PersonCreatedEvent, the poor bastards that created their Person before this will not be able to correct their emails.)
```cs
public void CorrectEmail(CorrectPersonEmail correctPersonEmail)
{
    if (correctPersonEmail.PersonId != Id)
    {
        throw new InvalidOperationException("Command route error");
    }

    if (CreatedAt.AddDays(10) < DateTime.Now)
    {
        throw new InvalidOperationException("Not allowed to correct email after more then 10 days.");
    }

    AssertEmailIsValid(correctPersonEmail.Email);

    var evt = new PersonEmailCorrected(Id, correctPersonEmail.Email);
    ApplyChange(evt);
}

public void ChangeEmail(ChangePersonEmail changePersonEmail)
{
    if (changePersonEmail.PersonId != Id)
    {
        throw new InvalidOperationException("Command route error");
    }

    AssertEmailIsValid(changePersonEmail.Email);

    var evt = new PersonEmailChanged(Id, changePersonEmail.Email);
    ApplyChange(evt);
}

```
The code for changing name is so similar that I won't waste space to show it here.

The last thing we need to do is to add functionality to handle the commands in the commandhandler.

```cs
public void Execute(ChangePersonName command)
{
   _repository.ActOn(command.PersonId, command.CommandId, person => person.ChangePersonName(command));
}

public void Execute(CorrectPersonEmail command)
{
   _repository.ActOn(command.PersonId, command.CommandId, person => person.CorrectEmail(command));
}

public void Execute(ChangePersonEmail command)
{
   _repository.ActOn(command.PersonId, command.CommandId, person => person.ChangeEmail(command));
}
```

So now we have added functionality to our Person, without to much new code. Only the events are the extra stuff we need to use eventsourcing. Because you already use CQRS, don't you?

Since you are a professional developer you surely uses unit testing. And here is where Eventsourcing starts to shine. It's quite easy to get the you test objects into the preferred state. (Of course it would not be super hard to do in regular CRUD style either this far.)
So lets look at a test:

```cs
public class WhenCreated11DaysAgo
    {
        private Person _sut;

        [SetUp]
        public void Setup()
        {
            var events = new List<IPersonEvent>()
                             {
                                 new PersonCreated(
                                     Guid.NewGuid(),
                                     "Richard Lindeberg",
                                     "Richard.liiindeberg@gmailcom",
                                     DateTime.Now.AddDays(-11))
                             };
            _sut = new Person(events);
        }

        [Test]
        public void ShouldThrowErrorWhenCorrectingEmail()
        {
            var correctEmail = "richard.lindeberg@gmail.com";
            Assert.Throws<InvalidOperationException>(() => _sut.CorrectEmail(new CorrectPersonEmail(Guid.NewGuid(), _sut.Id, correctEmail)));
        }

        [Test]
        public void ShouldHaveNoUncommitedEventsWhenCrrecting()
        {
            var correctEmail = "richard.lindeberg@gmail.com";
            try
            {
                _sut.CorrectEmail(new CorrectPersonEmail(Guid.NewGuid(), _sut.Id, correctEmail));
            }
            catch (Exception)
            {
            }
            Assert.IsEmpty(_sut.UncommitEvents);
        }

        [Test]
        public void ShouldChangeEmail()
        {
            var correctEmail = "richard.lindeberg@gmail.com";
            _sut.ChangeEmail(new ChangePersonEmail(Guid.NewGuid(), _sut.Id, correctEmail));
            Assert.AreEqual(correctEmail, _sut.Email);
        }

        [Test]
        public void ShouldHaveHaveOneUncommitedEventsWhenChangingEmail()
        {
            var correctEmail = "richard.lindeberg@gmail.com";
            _sut.ChangeEmail(new ChangePersonEmail(Guid.NewGuid(), _sut.Id, correctEmail));
            Assert.AreEqual(1,_sut.UncommitEvents.Count());
        }
    }
```

It's just standard nUnit, first we create the system under test by creating a new Person in the state we want.
And that is that we should have created it 11 days ago. Then we check that we can change the email but not correct it. And we also check so that we have the correct amounts of uncommitted events. Later on we might extent the test to check that there is actually the correct type of events in the UncommentEvents enumerable.

The sourcecode can be found at [https://github.com/RichardLindeberg/Blog.EventSourcing/tree/Part3](https://github.com/RichardLindeberg/Blog.EventSourcing/tree/Part3)
