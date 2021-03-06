---
layout: post
title: Sending high volume emails without making your users wait
---
One of the applications I am working on is a social networking site for martial artists. We have a feature similar to the follow feature on Twitter. When you see a page you are interested in you can follow it and get updates when anything changes.

![_config.yml]({{ site.baseurl }}/images/2016-11-14-My-First-Post/follow_gym.jpg){: .center-image }

In our database each page has a unique id and each user has a unique id. When you follow a page we write a row to the followers table that has the page id and the user id. When a change is made to a page we query for all of the followers of that page and send them an email saying the page has been updated with a link to the page.

[show list of steps we take right now, later show new list of steps in diagram]

For a small number of users this won't cause issues. If I own a page I can go make my update, save the changes, and everyone will be notified without the user having to wait too long.

But what happens as the number of followers starts to increase? Sending 100 emails is relatively fast, but what about 1,000, 10,000 or 50,000? Imagine Kim Kardashian uploading something and having to notify 80 million people. When you make this update you would be waiting a long time while all of these emails get sent.

The core issue here is that we are making our update and sending the emails synchonously. That is to say, a user makes a change and we must send all the emails before returning the result to the user. The real question is, does the user really need to wait for all of the emails to be sent before continuing on with what they are doing? In this case the answer is no. I have updated my page and I want my followers to be notified but that process doesn't need to prevent me from continuing to use the site. We should let them continue doing their work without waiting for the emails to be sent(without blocking the operation).

### The Solution

What we need is the ability to define some work to be done, and allow it to be done in the background. You can imagine that we have some code that says please make this update, then schedule all of these emails to be sent, then return. Leaving the email sending to be done by someone else. For this specific case we don't really care exactly who does it or when. We would like it to be done in a reasonable amount of time to have good UX for the people receiving the update. But we also want it for the user making the update. We don't want users being notified in three days of an update that happened today.

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

Specifically, we are going to need the following:

- A way to write the data of what needs to be done
- A way to store the data in a queue
- A way to read the data of what needs to be done
- A way to send emails

The way we are going to fill these needs is by using C#, SQS, and SES. We will write simple console apps to both read from and write to the queue. The queue itself will be maintained by SQS on AWS. For emails we will use SES on AWS. When solving this yourself you can use whatever suits you best. You may decide to use a different programming language, to run your listeners in a console app on EC2, to use a different message queue, etc. The goal here is to illustrate one set of components. Regardless of the underlying technology the pattern and pieces are still the same.

### The Code

Prerequisites:
In order to do this exact setup you will need Visual Studio installed on your computer and an AWS account setup.

We are going to perform the following steps:

1. Create a message queue in SQS
2. Write a program that will wrap SQS and allow us to write to the queue
3. Write a program that will wrap SQS and allow us to read from the queue
- a. This program will also have a wrapper for SES that sends emails

#### Step 1. Create a message queue in SQS

![_config.yml]({{ site.baseurl }}/images/2016-11-14-My-First-Post/aws_services_list.jpg){: .center-image }
![_config.yml]({{ site.baseurl }}/images/2016-11-14-My-First-Post/sqs_get_started.jpg){: .center-image }
![_config.yml]({{ site.baseurl }}/images/2016-11-14-My-First-Post/create_queue.jpg){: .center-image }
![_config.yml]({{ site.baseurl }}/images/2016-11-14-My-First-Post/queue_details.jpg){: .center-image }

TODO: Setup permissions for access to the queue.

#### Step 2. Write a wrapper for SQS and use it to write to the queue

Before we can write the wrapper for SQS we have to add a few NuGet packages. If you aren't familiar with NuGet you can check it out HERE.

To add a new package we are going to click Tools -> NuGet Package Manager -> Manage NuGet Packages for Solution...

![_config.yml]({{ site.baseurl }}/images/2016-11-14-My-First-Post/add_nuget_package.jpg){: .center-image }

After we do that we are going to add the following:

![_config.yml]({{ site.baseurl }}/images/2016-11-14-My-First-Post/sqs_nuget_package.jpg){: .center-image }

![_config.yml]({{ site.baseurl }}/images/2016-11-14-My-First-Post/ses_nuget_package.jpg){: .center-image }

![_config.yml]({{ site.baseurl }}/images/2016-11-14-My-First-Post/newtonsoft_nuget_package.jpg){: .center-image }

Now that we have our packages added, we can start coding!

A great resource on how to do some of this can be found here. I highly recommend the AWS documentation whenever you get stuck with anything. They keep it up to date and they have solutions for pretty much any issue you may have.
http://docs.aws.amazon.com/sdk-for-net/v2/developer-guide/how-to/sqs/SendMessage.html

First we need an interface for our message queue:

```cs
public interface IMessageQueue
{
  void QueueMessage(String message);
}
```

We are also going to need a class to represent the message in the queue that is agnostic of the underlying implementation. This lets us swap out other message queues later:

```cs
public class QueueMessage
{
  public QueueMessage(String body, String receiptHandle)
  {
    this.Body = body;
    this.ReceiptHandle = receiptHandle;
  }

  public String Body { get; private set; }
  public String ReceiptHandle { get; private set; }
}
```

Then we need to create a class that implements the interface and uses the AWS SQS library:

```cs
public class SqsMessageQueue : IMessageQueue
{
  private readonly AmazonSQSClient _sqsClient;

  /* NOTE: It's not a good practice to store your credentials like this, this is demo code for illustrative purposes only. */
  private readonly String _accessKey = "FILL IN";
  private readonly String _secretKey = "FILL IN";

  public SqsMessageQueue()
  {
    _sqsClient = new AmazonSQSClient(new BasicAWSCredentials(_accessKey, _secretKey), new AmazonSQSConfig { ServiceURL = "http://sqs.us-east-1.amazonaws.com" });
  }

  public void QueueMessage(String message)
  {
    SendMessageRequest sendMessageRequest = new SendMessageRequest
    {
        QueueUrl = "https://sqs.us-east-1.amazonaws.com/xxxxxxxxxxxx/EmailsToBeSent",
        MessageBody = message
    };

    SendMessageResponse sendMessageResponse = _sqsClient.SendMessage(sendMessageRequest);
  }
}
```

Then we need a class to represent an email message, this is what we will store in our queue:

```cs
public class EmailMessage
{
  public EmailMessage(String from, String to, String subject, String body)
  {
    this.From = from;
    this.To = to;
    this.Subject = subject;
    this.Body = body;
  }

  public String From { get; private set; }
  public String To { get; private set; }
  public String Subject { get; private set; }
  public String Body { get; private set; }
}
```

Now that all the components are in place let's combine everything and send a message to our queue!

```cs
private static void SendMessage()
{
  /* Create the message queue object.  */
  IMessageQueue queue = new SqsMessageQueue();

  /* Create the email we want to send. */
  var emailMessage = new EmailMessage(from: "stephenmichaelf@gmail.com", to: "stephenmichaelf@gmail.com", subject: "This is my subject!", body: "This is the body of my email.");

  /* Send the email. We need to serialize the message first. */
  queue.QueueMessage(JsonConvert.SerializeObject(emailMessage));
}
```
If we debug the code we can see the result of sending the message:

![_config.yml]({{ site.baseurl }}/images/2016-11-14-My-First-Post/queue_message_result.jpg){: .center-image }

And if we go back to AWS we can see there is a message in the queue:

![_config.yml]({{ site.baseurl }}/images/2016-11-14-My-First-Post/sqs_queue_item.jpg){: .center-image }

#### Step 3. Reuse the wrapper for SQS and use it to read from the queue. Include a wrapper for SES that sends emails

Talk about the code that receives the message.

Now we need to write some code to wrap the email sending capability of SES. It may look something like this:

```cs
public interface IEmailSender
{
  void Send(EmailMessage emailMessage);
}

public class SesEmailSender : IEmailSender
{
  private readonly AmazonSimpleEmailServiceClient _emailServiceClient;

  public SesEmailSender()
  {
    _emailServiceClient = new AmazonSimpleEmailServiceClient(
      new BasicAWSCredentials(accessKey: "FILL IN", secretKey: "FILL IN"),  
      RegionEndpoint.USEast1);
  }

  public void Send(EmailMessage emailMessage)
  {
    var sendRequest = new SendEmailRequest
    {
      Source = emailMessage.From,
      Destination = new Destination { ToAddresses = new List<string> { emailMessage.To } },
      Message = new Message
      {
          Subject = new Content(emailMessage.Subject),
          Body = new Body { Html = new Content(emailMessage.Body) }
      }
    };

    try
    {
      var response = _emailServiceClient.SendEmail(sendRequest);
    }
    catch {  }
  }
}
```
Now we can finish the code to receive the message:

```cs
private static void ReceiveMessage()
{
    /* Create the message queue object.  */
    IMessageQueue queue = new SqsMessageQueue();

    /* Create the email sender. */
    IEmailSender emailSender = new SesEmailSender();

    /* Listen for messages and send an email when we receive one. */
    QueueMessage message = queue.ReceiveMessage();

    if (message != null)
    {
        /* Serialize the message back into an EmailMessage. */
        EmailMessage emailMessage = JsonConvert.DeserializeObject<EmailMessage>(message.Body);

        /* Send an email based on the contents of the message. */
        emailSender.Send(emailMessage);

        /* Delete the message from the queue once we have successfully processed it. */
        queue.DeleteMessage(message.ReceiptHandle);
    }
}
```

Now if we run this code the message will be processed! We can check SQS to verify there are no mesages left in the queue:

![_config.yml]({{ site.baseurl }}/images/2016-11-14-My-First-Post/zero_messages_available.jpg){: .center-image }

And if I check my email I should see my new email:

![_config.yml]({{ site.baseurl }}/images/2016-11-14-My-First-Post/email_from_message_queue.jpg){: .center-image }

### Conclusion

Congratulations on making it through all of that! I think we have done a solid job of making our solution scale a lot better. There will definitely still be changes we need to make as things grow. You can imagine storing >1 email in the queue message for faster processing and less back and forth with the queue. Also if we get a ton of emails that need to be send we may need more than one machine reading off the queue and sending messages. The nice thing about AWS is that it will handle the scaling of SES and SQS for us!


Tasks
* Add some diagrams to make it more clear, including sample data values
* Add a section describing how this pattern can be generalized to other purposes and how, what to watch out for, what the requirements are
* Add link and info to get the code themselves
