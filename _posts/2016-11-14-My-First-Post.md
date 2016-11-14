---
layout: post
title: Sending emails without making your users wait
---
### The Problem 

One of the applications I am working on is a social networking site for martial artists. One of the features we have is similar to the follow feature on Twitter. When you see a page you are interested in you can choose to Follow it and get updates when anything changes.

[insert picture of following here]

For now the structure of this on the back end is straight forward. Each page as a unique id and each user has a unique id. When you want to follow a page we write a row to the followers table that has the page id and the user id. Whenever a change is made to a page we query for all of the followers and send each of them an email saying the page has been updated with a link to the page.

[show list of steps we take right now, later show new list of steps in diagram]

For a small number of users you can see this probably won't cause many issues. If I own a page I can go make my update, save the changes, and be sure everyone will be notified.

What happens though as the number of followers starts to increase? Sending 100 emails should be pretty fast, but what about 1,000, 10,000 or 50,000? Imagine Kim Kardashian uploading something and having to notify 80 million people. When you make this update you would be waiting a pretty long time for all of these emails to get sent.

The core issue here is pretty obvious. We are making our update and sending the emails synchonously. That is to say, a user makes a change and we must send all the emails before returning the result to the user. The real question is, does the user really need to wait for all of the emails to be sent before continuing on with what they are doing? In this case the answer is a clear no. We should let them continue doing their work without waiting for the emails to be sent(without blocking the operation).

### The Solution

At a logical level what we need is the ability to define some work to be done, and allow that work to be done out of band. You can imagine that we have some code that says please make this update, then schedule all of these emails to be sent, then return. Leaving the email sending to be done by someone else. For this specific case we don't really care exactly who does it or when. Though we would like it to be done in a reasonable amount of time to have good UX. We don't want users being notified in three days of an update that happened today.

So the piece we are missing is this ability to define a set of work that needs to be done, store that definition somewhere, and have something to process the set of work when it can. The solution that we are going to use here is message queues. You can imagine a message queue as a queue of work that needs to be done. The three components are:

- Someone creates the item to be done
- The item of work to be done is stored on the queue
- Soneone watches the queue and processes any work that is put on to it

Why is this useful? This process lets us decouple the creation of the work to be done from actually doing it. You can imagine the load on our website going up a little but the need to send emails increasing dramatically and in that case this setup would let us scale the workers who watch the queue independently of scaling the site.

I would like to make a quick note here. If we abstract away the ability to send emails in our application, it would be easy to swap the implementation from one that sends emails synchronously to one that uses a message queue.

If we have the following:

```cs
public interface IEmailSender
{
  void SendEmails(IEnumerable<String> recipients, String body, String message);
}
```

We can then have two implementations(SynchronousEmailSender and MessageQueueEmailSender) of this interface, one that sends them right away and one that writes them to a queue(the latter of which we will see below). Then the change is as simple as writing the new implementation and adjusting our composition root to use the new implementation. We don't have to worry about going through all of our code that sends email and changing it. The implementation is hidden from the user.

### The Technology

Now that we are aware of the process, what pieces do we need to make it happen? We are going to need the ability to write to a queue, the queue itself, and a worker(for now one, can be scaled up as needed) to do the processing. For this specific case that work will also involve sending emails.

Today we are going to use Amazon Web Services for all of the processing and queue management and Java/C# for the code. I assume most of you are familiar with AWS, if not I strongly encourage you to take a deeper look. They have excellent documentation and many managed services covering a broad range of capabilities. Basically almost anything you need to do they probably have a service for it. And you only pay for what you use.

Specifically, we are going to use the following:

- AWS: PAAS/IAAS where all of the services are
- SES: Simple Email Service used to send emails
- SQS: Amazon's queing service to store our messages and pull work from
- C#/Java: What we are going to write the code in
- EC2/Lambda: Used as our workers

### The Code

```cs
int x = 1;
```

Tasks
* Add some diagrams to make it more clear, including sample data values
* Add a section describing how this pattern can be generalized to other purposes and how, what to watch out for, what the requirements are

