# How to send emails from MailerQ

MailerQ is an MTA that uses an AMQP message queue as its outbox. It loads JSON encoded messages from this message queue, sends them out over an SMTP connection, and then publishes the [JSON encoded result](/documentation/result-queue "MailerQ result queue") back to one or more result queues, where other programs or scripts can pick them up.

Sending email with MailerQ can be done in three different ways, you can either:

*   write a program or script that [directly adds a JSON encoded email](/documentation/send-email#Amqp "AMQP") to the outbox message queue
*   use MailerQ's [built-in SMTP server](/documentation/send-email#Smtp "SMTP")
*   use MailerQ to [read messages from standard input](/documentation/send-email#Cli)

## Post email directly to the AMQP message queue

Because MailerQ fetches all messages from a RabbitMQ message queue, the fastest way to inject emails is by having your application publish the mails directly to RabbitMQ. This is not difficult. After all, JSON is a [widely known data format](http://www.json.org) that can be easily created and processed by almost every programming language there is, and the message queue can be easily accessed (add messages, remove messages) using the AMQP protocol, for which also [many plugins and libraries](http://www.rabbitmq.com/devtools.html) are available. To help you out, we [created a few examples](/documentation/mailerq-examples "MailerQ examples") for commonly used languages.

![MailerQ put it in RabbitMQ](/Resources/Images/mailerq-put-it-in-rabbitmq.png)

On top of that, if you publish the messages directly to a message queue, your script will be faster (publishing to AMQP is fast), and you can use special features that are not available when you use SMTP.

### Sending messages

As we mentioned above, to send out emails with the MailerQ MTA, you can simply add a message to the outbox message queue. This JSON encoded message should contain a minimum of three fields that hold the envelope email address, recipient's email address and full MIME message.

**Minimal requirements for a JSON encoded email**

*   The envelope email address from which the message is sent
*   The destination address
*   The full message MIME.

That's it. In short, the minimal JSON encoded message to put in the queue therefore looks like this:

````
{
    "envelope": "my-sender-address@my-domain.com",
    "recipient": "info@example.org",
    "mime": "From: my-sender-address@my-domain.com\r\n
             To: info@example.org\r\n
             Subject:Example subject\r\n\r\n
             This is the example message text"
}

````

Note that for ease of reading we added some spaces to the message mime in the above example.

If your MailerQ license allows this, you can also assign nested JSON objects to the "mime" field. MailerQ detects whether you set a string or nested object, and will in the latter case process this nested structure, and create a full MIME out of it:

````
{
    "envelope": "my-sender-address@my-domain.com",
    "recipient": "info@example.org",
    "mime": {
        "from": "my-sender-address@my-domain.com",
        "to": "info@example.org",
        "subject": "Example subject",
        "textVersion": "This is the example message text"
    }
}

````

MailerQ uses the same algorithm for turning JSON into full blown MIME messages as the [responsiveemail.com](https://www.responsiveemail.com) web service, but without having to make any calls to this web service. The algorithm is completely embedded in MailerQ. This feature allows you to send HTML mails without having to worry about the layout and responsiveness of your email: you only supply JSON and the HTML mails are automatically generated by MailerQ.

In the example we only used the "from", "to", "subject" and "textVersion" properties to create a very simple MIME message. Besides these properties however, MailerQ recognizes much more properties that allow you to add HTML code, include images, attachments, RSS feeds -- and much more. To read more about this, please read our documentation about [creating responsive emails with MailerQ](/documentation/responsive-email).

As we said, you need a special license to use this "responsive email" feature. If you do not have such a special license, you can of course still inject emails in the message queue, but will have to create the MIME string yourself, and you can only assign strings to the JSON "mime" property.

## Using the built-in SMTP server

If you do not want to inject mails directly into the RabbitMQ message queue, you can also send emails to the SMTP port that is opened by MailerQ. All messages sent to this port are received by MailerQ and are automatically published to a message queue, where they will then be picked up and delivered to the recipients.

In MailerQ's [config file](/documentation/configuration "MailerQ configuration") you can set to which queue emails received by MailerQ are published. Most users set this to the same value as the outbox queue, so that all received emails are automatically published to the outbox queue, from which they are then directly picked up again and scheduled for immediate forwarding.

![MailerQ shared inbox outbox queue](/Resources/Images/mailerq-shared-inbox-outbox-queue.png)

MailerQ stores all incoming messages first in a message queue. You can add intermediate scripts that process these messages before they are forwarded to the outbox queue. Would you like to add a script that does additional processing or filtering before a message is forwarded? Configure MailerQ to publish received messages in an inbox queue and let your scripts read these messages, from this inbox queue, to process or filter them. After that, post the message to the outbox queue where MailerQ picks them up to deliver them. You might be interested in [AMQPipe](https://www.amqpipe.com "AMQPipe"), a high performance application that reads messages as fast as it can from a RabbitMQ message queue, runs your scripts to process these messages, and publishes the results back to RabbitMQ.

![MailerQ seperate inbox outbox queues](/Resources/Images/mailerq-seperate-inbox-outbox-queues.png)

### SMTP and multiple IP addresses

If you run MailerQ on a server with multiple IP addresses, the SMTP server is available on all those addresses too. You can thus choose to which IP address to send your mail. MailerQ recognizes the IP address to which the mail was originally submitted, stores that information in the JSON message, and when the mail is finally forwarded to the internet, it will be sent out from _exactly the same_ IP address. You should be aware that this can cause problems if you deliver your email to an IP address from which no external connections can be made (like [127.0.0.1](http://en.wikipedia.org/wiki/Localhost)).

If you want MailerQ to send out the mail from a different IP address than that you originally sent it to, you can include [an extra mime header field](/documentation/send-email#localips) that instructs MailerQ to use a different IP instead.

## MailerQ as command line utility

The MailerQ program can be started in two different modes: as a daemon process that runs all the time and that sends out the messages, or as a command line utility that reads a message from standard input and publishes it to the inbox queue. As a command line utility, MailerQ does not attempt to send out the mail - it only puts the message in the RabbitMQ message queue. You need a seperate running MailerQ daemon process that takes care of the delivery.

![MailerQ mime output](/Resources/Images/mailerq-mime-output-stdout.png)

When you normally start up MailerQ, it operates as a daemon process. To start it as a command line utility for reading a message from standard input, you need to pass extra arguments to the program; either the email addresses of the recipients, or the option '--extract-recipients' to tell MailerQ that it should read in a mime message from standard input and filter out the destination addresses from it.

````
$ mailerq recipient1@example.com recipient2@example.com

````

````
$ mailerq --extract-recipients

````

The message read in by MailerQ is converted into a JSON encoded email message and published to the inbox queue. An other MailerQ daemon process can then pick it up from there and take care of the delivery.

### End of message

MailerQ reads in all data from standard input until it encounters a line with a single dot on it. It treats this line as the end-of-message marker and will publish all data up to this dot (but not including the dot) to the inbox message queue.

This could become problematic if your email also contains lines that start with a dot. Such lines should of course not be treated as end-of-message markers. There are two ways to prevent this; you can either stuff those lines with an extra dot in front. MailerQ will automatically recognize this and remove the extra dot, or you can add the command line option '--ignore-dot' to instruct MailerQ that dots do not have a special meaning, and that the message is ended with end-of-file instead.

````
$ mailerq --ignore-dot recipient1@example.com recipient2@example.com

````

````
$ mailerq --ignore-dot --extract-recipients

````

### Envelope address

The envelope address that MailerQ uses for sending out an email is automatically filtered from the message MIME (it can be set in the custom x-mq-envelope header, or in the return-path header). However, you can also set the envelope address manually with the '--envelope' command line option.

````
$ mailerq --envelope myaddress@mydomain.com --extract-recipients

````

### Message format

Messages that are read from standard input should be in MIME format. The headers should be seperated from the message data by a newline, and lines should end with either a newline character, or a linefeed+newline character.

````
From: sender@example.com
To: recipient@example.com
Subject: Example message
x-mq-maxdelivertime: 2013-05-01 00:00:00

Example email data.
Email data ends when a single dot followed by new line character is send.

````

### Sending emails from file

Because MailerQ reads in messages from standard input, you can use regular unix shell pipes and I/O redirection operators for sending mail.

````
$ cat email.txt | mailerq --extract-recipients --ignore-dot

````

````
$ mailerq --extract-recipients --ignore-dot < email.txt 

````

### Using MailerQ in PHP mail() function

The PHP mail() function internally uses a command line utility (for example: "sendmail") to send out the mail. By modifying the configuration of PHP you can change this into "mailerq", so that all mails sent from PHP are automatically delivered by MailerQ.

To use MailerQ for sending emails in PHP using mail() function, set property "sendmail_path" in php.ini to:

````
sendmail_path = mailerq --envelope my-sender-address@my-domain.com --extract-recipients --ignore-dot

````

## Setting additional properties for better delivery control

In the examples that we have so far given, we only demonstrated the elementary properties "envelope", "recipient" and "mime". But to have better control of delivery, you can set all sort of additional properties too. These are properties that for example specify the number of send attempts, the max delivery time, the IP addresses to use, et cetera. These properties can either be set in the JSON object if you publish mails directly to the outbox message queue, or as extra x-mq-* header fields in the MIME message.

Note that the x-mq-* header fields are only processed when you send the mail over SMTP to MailerQ or when you use MailerQ as a command line utility. If you publish mails directly to the AMQP message queue, the x-mq-* headers will not be processed. The headers are removed from the message when the mail is converted into a JSON object, and will thus no longer be included when the message reaches the final recipient.

### Envelope address

The envelope address is the address that is used in the 'MAIL FROM' communication. It is a required property in the JSON object. If you submit a message as MIME (via SMTP or command line), you can set it with the x-mq-envelope header.

#### MIME header

````
x-mq-envelope: my-sender-address@my-domain.com

````

#### JSON property

````
{
    "envelope": "my-sender-address@my-domain.com",
    "recipient": "info@example.org",
    "mime": "..."
}

````

### Max delivery time

When a message cannot be delivered immediately because of unresponsive receivers, greylisting or throttling, MailerQ publishes back the email to the outbox queue for later delivery. This can result in emails that are sent much later than the time that you first added them to the message queue.

If you do not want an email to be delivered after a certain time, just add a maxdelivertime property.

#### MIME header

````
x-mq-maxdelivertime: 2013-02-01 00:00:00

````

#### JSON property

````
{
    "envelope": "my-sender-address@my-domain.com",
    "recipient": "info@example.org",
    "mime": "...",
    "maxdelivertime": "2013-02-01 00:00:00"
}

````

Timestamps in MailerQ always are in UTC. And please note that MailerQ is not very tolerant in parsing timestamps, so better make sure that you use the right formatting (YYYY-MM-DD HH:MM:SS).

If you do not specify an explicit max delivery time, MailerQ will attempt to deliver the mail within 24 hours after the mail was first picked up from the outbox.

### Max number of attempts

Just like a max delivery time, you can also control the max number of attempts that MailerQ uses to send out an email. If a first attempt fails because a remote server is unreachable or does not immediately accept the message, MailerQ will make a new attempt a little later.

By default, MailerQ tries to send out the mail six times. If you want more or less attempts, you can specify maxattempts propery:

#### MIME header

````
x-mq-maxattempts: 3

````

#### JSON property

````
{
    "envelope": "my-sender-address@my-domain.com",
    "recipient": "info@example.org",
    "mime": "...",
    "maxattempts": 3
}

````

### Inlinize CSS

When you send out HTML emails, you face the problem that not all email clients support stylesheets that are set in the header. Some email clients (especially web based clients) strip out the CSS code from the HTML header. This often messes up the layout and look of your messages. To overcome this, it is better not to set CSS settings in the header in the first place, but use 'style="..."' attributes in the HTML code instead.

MailerQ can do this automatically. If you set the "inlinecss" property to true, MailerQ parses the HTML email, and converts the CSS code from the HTML header into inline 'style="..."' attributes in the HTML body.

#### MIME header

````
x-mq-inlinecss: 1

````

#### JSON property

````
{
    "envelope": "my-sender-address@my-domain.com",
    "recipient": "info@example.org",
    "mime": "...",
    "inlinecss": 1
}           

````

### Normalize line endings

All lines of the email must be terminated with a linefeed/newline combination: \r\n. If you use other line endings (for example only newlines and no linefeeds) the email might not get delivered, and DKIM signing might fail. However, if you set the property "normalize" to 1, MailerQ will automatically convert all line endings to valid linefeed/newline endings.

#### MIME header

````
x-normalize: 1

````

#### JSON property

````
{
    "envelope": "my-sender-address@my-domain.com",
    "recipient": "info@example.org",
    "mime": "...",
    "normalize": 1
}

````

### Local IP addresses

Some mail servers keep track of the number of parallel connections that are made to it - and put limits on the number of connections or deliveries. If your server has multiple IPs, MailerQ can send out mail from all those IP addresses at the same time so that the receiver does not notice that the connections all come from the same source. This can be useful to increase the delivery rate for sending out mail.

By default, MailerQ makes all connections to remote mail servers from the default (first) available IP of the host server. If your server has multiple IPs assigned to it, you can instruct MailerQ to use a different local IP for sending out the mail.

To try different IP address for sending out the mail, you can add a list of available IP addresses:

#### MIME header

````
x-mq-ip: 231.34.13.156
x-mq-ip: 231.34.13.158

````

#### JSON property

````
{
    "envelope": "my-sender-address@my-domain.com",
    "recipient": "info@example.org",
    "mime": "...",
    "ips": ["231.34.13.156", "231.34.13.158"]
}

````

MailerQ will randomly pick one of the IPs to send out the mail. Be aware that you can of course only use addresses that are actually bound to the host that MailerQ runs on. Other IP addresses will result in failed deliveries.

### Storing messages in message store

MIME bodies can become very large, especially if your emails contain attachments or embedded content. To prevent that such big messages have to be processed by RabbitMQ, MailerQ can be configured to use MongoDB, MySQL, PostgreSQL or SQLite for the message bodies instead. You can then publish a much smaller JSON message to RabbitMQ, and keep the full MIME messages in an external message store.

MailerQ waits with loading the message from message store until the SMTP connection has been set up, so that no time and resources are wasted on fetching information that is not needed.

In all the given examples we have included the full message data in the JSON object. This is probably the best thing to do, as most messages are delivered right away, and there would be more overhead in storing messages in an external message store. However, if an email has to be rescheduled, and is going to be stored in RabbitMQ for a longer period of time, it is better to store the MIME message in an external message store. In the JSON message you will then not need a "mime" property, but a "key" property instead that refers to the key where the message can be found.

#### JSON property

````
{
    "envelope": "my-sender-address@my-domain.com",
    "recipient": "info@example.org",
    "key": "message-store-key-where-data-can-be-found"
}

````

If you use the external message store, you must make sure that you store either valid MIME messages in this message store, or valid JSON messages according to the specification of the [responsive email specification](/documentation/responsive-email).

Even when you have configured MailerQ to use an external message store, you may still decide to publish full MIME messages to RabbitMQ, and not use the message store. Messages published by MailerQ will however always use the message store.

### Keep messages after delivery

When a message is completely processed – either because it was successfully delivered, or the delivery failed – MailerQ publishes it to the results queue, where you can pick it up for further processing.

By default, MailerQ throws away the mime data to make room in the JSON object and in the message store. If you do not want the message data to be removed, you can tell so by adding the "keepmime" option:

#### MIME header

````
x-mq-keepmime: 1

````

#### JSON property

````
{
    "envelope": "my-sender-address@my-domain.com",
    "recipient": "info@example.org",
    "keepmime": 1
}

````

### Custom result queues

After MailerQ has processed an email, it publishes the mail to one or more result queues. The default queues to use for this are configured in the global configuration file, but you can set other queue names in properties to tell MailerQ to use other result queues.

#### MIME header

````
x-mq-results-queue: name-of-results-queue
x-mq-failure-queue: name-of-failure-queue
x-mq-success-queue: name-of-success-queue
x-mq-retry-queue: name-of-retry-queue

````

#### JSON property

````
{
    "envelope": "my-sender-address@my-domain.com",
    "recipient": "info@example.org",
    "mime": "...",
    "queues": {
        "results": "name-of-results-queue",
        "failure": "name-of-failure-queue",
        "success": "name-of-success-queue",
        "retry": "name-of-retry-queue"
    }
}

````

All properties of the "queues" object are optional. If you leave an option out, MailerQ will not use that queue. Thus, if you for example want to process only the errors for a certain e-mail, you can only set the "failure" queue. When the delivery succeeds, MailerQ will silently discard the mail, without adding it to any result queue.

When the "queues" property is used, the queues mentioned in the global configuration file are completely ignored. This is even so if you have not even specified all possible queues in the MIME header or JSON object.

### Data

It is possible to personalize your mime object input before sending it using the data property. In case this property is omitted the mime is sent as is. This data property will only work in JSON as it can be completely nested. This personalization is done using SMART-TPL.

### Smarthosts

Normally, MailerQ delivers all mail to the actual recipient. However, in some cases it is desired that another server actually delivers the emails, whereas MailerQ only forwards them to that other server. For these cases, the smarthost settings are available. These settings can be set for all messages in the configuration file. If all messages are to be routed via a smarthost, this is the best option.

However, for fine-grained control over smarthosts, these settings can also be added in the JSON for each message individually. If a smarthost is also set globally, these JSON settings will override that setting as a more precise setting.

#### MIME header

````
x-mq-smarthost-name: hostname-smarthost
x-mq-smarthost-port: 25
x-mq-smarthost-username: example-user
x-mq-smarthost-password: example-password

````

#### JSON property

````
{
    "envelope": "my-sender-address@my-domain.com",
    "recipient": "info@example.org",
    "mime": "...",
    "smarthost": {
        "name": "hostname-smarthost",
        "port": "25",
        "username": "example-user",
        "password": "example-password"
    }
}

````

If the smarthost option is added, only the name option is mandatory. Port will default to 25, and the absence of a username and password simply indicates MailerQ does not need to authenticate to the smarthost.

## Setting custom message properties

To have better control over your message queue, you can set additional properties that are included into RabbitMQ message, but will not be send to recipient.

Those properties don't affect sent email at all, they are just for monitoring and debuging purpose.

#### MIME header

````
x-mq-custom-property-name: some debug data that will be visible only in RabbitMQ message

````

#### JSON property

````
{
    "envelope": "my-sender-address@my-domain.com",
    "recipient": "info@example.org",
    "custom-property-name": "some debug data that will be visible only in RabbitMQ message"
    "mime": "..."
}

````

There are some properties that are reserved for internal use and should not be used as custom headers. Complete list is below:

* recipient
* domain
* forcedip
* delayed
* seen
* ips
* key
* mime
* results
* queues