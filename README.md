# inbox-python

Python bindings for the Nilas REST API. https://www.nilas.com/docs

## Installation

This library is available on pypi. You can install it by running `pip install inbox`.

##Requirements

- requests (>= 2.3.0)

## Examples

There's an example flask app in the `examples` directory. You can run the sample app to see how an authentication flow might be implemented.

*Note: you will need to replace the APP_ID and APP_SECRET with your Inbox App ID and secret to use the sample app.*

## Usage

### App ID and Secret

Before you can interact with the Nilas REST API, you need to register for the Nilas Developer Program at [https://www.nilas.com/](https://www.nilas.com/). After you've created a developer account, you can create a new application to generate an App ID / Secret pair.

Generally, you should store your App ID and Secret into environment variables to avoid adding them to source control. That said, in the example project and code snippets below, the values are hardcoded for convenience.


### Authentication

The Nilas REST API uses server-side (three-legged) OAuth, and this library provides convenience methods to simplify the OAuth process.
Here's how it works:

1. You redirect the user to our login page, along with your App Id and Secret
2. Your user logs in
3. She is redirected to a callback URL of your own, along with an access code
4. You use this access code to get an authorization token to the API

For more information about authenticating with Nilas, visit the [Developer Documentation](https://www.nilas.com/docs/gettingstarted-hosted#authenticating).

In practice, the Nilas REST API client simplifies this down to two steps.

**Step 1: Redirect the user to Inbox:**

```python
from flask import Flask, session, request, redirect, Response
from inbox import APIClient

@app.route('/')
def index():
    redirect_url = "http://0.0.0.0:8888/login_callback"
    client = APIClient(APP_ID, APP_SECRET)
    return redirect(client.authentication_url(redirect_uri))

```

**Step 2: Handle the Authentication Response:**

```python
@app.route('/login_callback')
def login_callback():
    if 'error' in request.args:
        return "Login error: {0}".format(request.args['error'])

    # Exchange the authorization code for an access token
    client = APIClient(APP_ID, APP_SECRET)
    code = request.args.get('code')
    session['access_token'] = client.token_for_code(code)
```

You can take a look at [examples/server.py](examples/server.py) to see a server
implementing the auth flow.

### Fetching Namespaces

```python
client = APIClient(APP_ID, APP_SECRET, token)

# Get the first namespace
namespace = client.namespaces.first()

# Print out the email address and provider (Gmail, Exchange)
print namespace.email_address
print namespace.provider
```


### Fetching Threads

```python
# Fetch the first thread
thread = namespace.threads.first()

# Fetch a specific thread
thread = namespace.threads.find('ac123acd123ef123')

# List all threads tagged `inbox`
# (paginating 50 at a time until no more are returned.)
for thread in namespace.threads.items():
    print thread.subject

# List the 5 most recent unread threads
for thread in namespace.threads.where(tag='unread'):
    print thread.subject

# List all threads with 'ben@nilas.com'
for thread in namespace.threads.where(any_email='ben@nilas.com').items():
    print thread.subject
```


### Working with Threads

```python
# List thread participants
for participant in thread.participants:
    print participant["email"]

# Mark as read
thread.mark_as_read()

# Archive
thread.archive()

# Unarchive
thread.unarchive()

# Add or remove arbitrary tags
tagsToAdd = ['inbox', 'cfa1233ef123acd12']
tagsToRemove = []
thread.update_tags(tagsToAdd, tagsToRemove)

# List messages
for message in thread.messages.items():
    print message.subject
```


### Working with Files

Files can be uploaded via two interfaces. One is providing data directly, another is by providing a stream (e.g. to an open file).

```python
# List files
for file in namespace.files:
    print file.filename

# Create a new file with the stream interface
f = open('test.py', 'r')
myfile = namespace.files.create()
myfile.filename = 'test.py'
myfile.stream = f
myfile.save()
f.close()

# Create a new file with the data interface
myfile2 = ns.files.create()
myfile2.filename = 'test.txt'
myfile2.data = "Hello World."
myfile2.save()
```

Once the files have been created, they can be added to a draft via the `attach()` function.

### Working with Drafts

Drafts can be created, saved and then sent. The following example will create a draft, attach a file to it and then send it.

```python
# Create the attachment
myfile = namespace.files.create()
myfile.filename = 'test.txt'
myfile.data = "hello world"

# Create a new draft
draft = namespace.drafts.create()
draft.to = [{'name': 'My Friend', 'email': 'my.friend@example.com'}]
draft.subject = "Here's an attachment"
draft.body = "Cheers mate!"
draft.attach(myfile)
draft.send()
```

### Working with Events

The following example shows how to create, update and delete an event.

```python
# Create the event
ev = namespace.events.create()
ev.title = "Party at the Ritz"
ev.when = {"start_time": 1416423667, "end_time": 1416448867} # These numbers are UTC timestamps
ev.location = "The Old Ritz"
ev.participants = [{"name": "My Friend', 'email': 'my.friend@example.com'}]
ev.save()

# Update it
ev.location = "The Waldorf-Astoria"
ev.save()

# Delete it
namespace.events.delete(ev.id)
```

### Working with Messages, Contacts, Calendars, etc.

Each of the primary collections (contacts, messages, etc.) behaves the same way as `threads`. For example, finding messages with a filter is similar to finding threads:

```python
messages = namespace.messages.where(to=ben@nilas.com).all()
```

The `where` method accepts a keyword argument for each of the filters documented in the [Inbox Filters Documentation](https://www.nilas.com/docs/api#filters).

## Account Management

### Account status

It's possible to query the status of all the user accounts registered to an app by using `.accounts`:

```python
accounts = client.accounts
print [(acc.sync_status, acc.account_id, acc.trial, acc.trial_expires) for acc in accounts.all()]
```

## Open-Source Sync Engine

The [Nilas Sync Engine](http://github.com/inboxapp/inbox) is open-source, and you can also use the python library with the open-source API. Since the open-source API provides no authentication or security, connecting to it is simple. When you instantiate the Inbox object, provide null for the App ID, App Secret, and API Token, and pass the fully-qualified address of your copy of the sync engine:

```python
from inbox import APIClient
client = APIClient(None, None, None, 'http://localhost:5555/')
```


## Contributing

We'd love your help making Nilas better. Join the Google Group for project updates and feature discussion. We also hang out in `#nilas` on [irc.freenode.net](http://irc.freenode.net), or you can email [support@nilas.com](mailto:support@nilas.com).

Please sign the Contributor License Agreement before submitting pull requests. (It's similar to other projects, like NodeJS or Meteor.)

If you have access to the PyPI repository, you can make a new release as such:

```shell
python setup.py release <major/minor/patch>
git log # to verify
python setup.py publish
```


## Looking for inbox.py?

If you're looking for Kenneth Reitz's SMTP project, please update your `requirements.txt` file to use `inbox.py` or see the [Inbox.py repo on GitHub](https://github.com/kennethreitz/inbox.py).
