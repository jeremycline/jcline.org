---
layout: post
title:  "Publishing messages in Fedora's infrastructure"
date:   2024-09-05 14:16:00 -0400
categories: blog fedora amqp
---

While at Flock to Fedora 2024, someone noted, to paraphrase a conversation from
a month ago, that working with Fedora's messaging infrastructure seemed more
intimidating now that it's transitioned from ZeroMQ to AMQP. I promised to
write up a blog post walking through the process of adding support to an
application, so here it is (better late than never?).

There are two ways to interact with Fedora's messaging system as a developer.
Your application can send (publish) messages, receive (consume) them, or both.

This blog post will focus on adding support to your application for publishing
in Fedora's infrastructure, and how to effectively unit test them. For anything
not covered in this post, I recommend the [official
documentation](https://fedora-messaging.readthedocs.io/en/stable/index.html).


## Getting Started

This post assumes you'll be using Python and the
[fedora-messaging](https://github.com/fedora-infra/fedora-messaging/) library
which uses [Pika](https://pypi.org/project/pika/) to communicate with the
message broker. If you're working in a different language this post isn't for
you, but you can use any AMQP-0.9 client to send messages in the [documented
message
format](https://fedora-messaging.readthedocs.io/en/stable/api/wire-format.html).

Use your Python package management tool of choice to [install
fedora-messaging](https://fedora-messaging.readthedocs.io/en/stable/user-guide/installation.html).


## Publishing

A minimal implementation looks like this:

```Python3
from fedora_messaging import api

msg = api.Message(topic="my.topic", body={"some_key": ["some", "value"]})
api.publish(msg)
```

This example, while pleasingly simple, glosses over some important details which
you will need to address.

### Errors

The first adjustment to make is to add error handling. See the [API
documentation](https://fedora-messaging.readthedocs.io/en/stable/api/api.html#publish)
for details on each exception, but here are the ones you generally want to
handle:


```Python3
from fedora_messaging import api, exceptions

msg = api.Message(topic="my.topic", body={"some_key": ["some", "value"]})
try:
    api.publish(msg)
except exceptions.ConnectionException as err:
    print(f"Connection to the message broker was lost: {err}")
except (exceptions.PublishTimeout, exceptions.PublishReturned) as err:
    print(f"Failed to publish the message: {err}")
```

What you do when an error occurs is up to you. You could retry in a loop until
it succeeds, or to skip publishing a message if it's not critical that a
message be sent for each event. Finally, you could do a mixture of the two by
adding it to a queue of messages to publish next time.

### Schema

The next adjustment is to define a schema for your message. While this isn't a
strict requirement, it ensures you don't accidentally change the format of the
message and break users, and it is required if you want notifications generated
from your messages via the Fedora Notification service to look good. There's a
[tutorial on providing
schema](https://fedora-messaging.readthedocs.io/en/stable/tutorial/schemas.html)
which I won't reproduce here, but this is what a message with a schema looks
like:

```Python3
from fedora_messaging import api

class MyMessage(api.Message):
    """A schema and common properties to enable Fedora Notification support"""
    topic = "my.topic"
    body_schema = {
        "id": "https://fedoraproject.org/message-schema/v1/my-app.message",
        "$schema": "https://json-schema.org/draft/2019-09/schema",
        "description": (
            "This message is sent by my-app when something interesting happens"
        ),
        "type": "object",
        "properties": {
            "some_key": {
                "type": "array",
                "description": "This is some key, isn't it?",
                "items": {"type": "string"},
            },
        },
        "required": [
            "some_key",
        ],
    }
```

I highly recommend following the tutorial and using CookieCutter to produce
your Python package containing your message schema.

After you create your Python package containing your message schema, add it as
a dependency to your application and publish it on PyPi for consumers to use.

### Testing

Now that we know how to publish messages in our applications, we probably want
to add tests to ensure messages are published (or not) as we expect.

Here's what a test looks like:

```Python3
from fedora_messaging import api, exceptions, testing

# This is our publish code from above inside a function
def function_that_publishes():
    msg = api.Message(topic="my.topic", body={"some_key": ["some", "value"]})
    try:
        api.publish(msg)
    except exceptions.ConnectionException as err:
        print(f"Connection to the message broker was lost: {err}")
    except (exceptions.PublishTimeout, exceptions.PublishReturned) as err:
        print(f"Failed to publish the message: {err}")


# This will assert a single message is published of type "message.Message"
with testing.mock_sends(api.Message):
    function_that_publishes()

# This will assert a single message is published of type "message.Message"
# and that the message topic and bodies match
with testing.mock_sends(api.Message(topic="my.topic", body={"some_key": ["some", "value"]})):
    function_that_publishes()
```

And that's it!

### Configuration

One last thing will need to be done when deploying your application:
configuring it to connect to Fedora's broker.

The API relies on configuration being provided by a TOML file located at, by
default, `/etc/fedora-messaging/config.toml`, but can be changed by setting the
`FEDORA_MESSAGING_CONF` environment variable to the alternate location.

The default configuration values for `fedora-messaging` work well for RabbitMQ
running on the same host as your application, but we need to adjust some
values to work with Fedora's message broker:

```toml
# This defaults to "false", which means your application will attempt to create
# the AMQP objects which works well for development. In Fedora, your
# application will get a "Permission Denied" on startup since your account will
# only have permission to create a few specific resources.
passive_declares = true

# Both these values rely on variables set in Fedora's Ansible repository, but
# in this example I've expanded them to what they are in production.
#
# <your-app-user> will be provided by Fedora's infrastructure team.
amqp_url = "amqps://<your-app-user>:@rabbitmq.fedoraproject.org/%2Fpubsub"
topic_prefix = "org.fedoraproject.prod"

# The TLS settings include a CA certificate used to verify the identity of the
# Fedora broker, along with a client certificate and key to authenticate your
# application to the broker.
#
# You must file a ticket with Fedora's infrastructure team to have an account
# created for your application along with a client certificate for authentication.
# The key and certificate is kept in Ansible. 
[tls]
ca_cert = "/etc/pki/rabbitmq/ca/rabbitmq.ca"
keyfile = "/etc/pki/rabbitmq/key/user.key"
certfile = "/etc/pki/rabbitmq/cert/user.crt"

# This identifies your application to RabbitMQ to aid in debugging.
[client_properties]
app = "My Application"
```

These are the only settings you should need to adjust for publishing messages.
