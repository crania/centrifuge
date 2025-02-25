v0.8.2
======

* fix for Python 3 in web interface AuthHandler - token could not be serialized into json.

v0.8.1
======

* fix total client count

v0.8.0
======

This is a massive release with lots of changes - the goal was to get rid of structure backends to move all
structure management to configuration file. You don't need SQLite, MongoDB or PostgreSQL anymore.

Changes opened a way for rewriting Centrifuge in Go - [Centrifugo](https://github.com/centrifugal/centrifugo) - it
already has first release. Centrifugo is almost a drop-in replacement for Centrifuge 0.8.0
(with some small differences in command-line argument names for logging). Python version most probably
won't be supported (only bug fixes) - all new changes will be in Centrifugo. I am very sorry
if you don't like all these global changes - but I personally think that moving Centrifuge to Go is
right decision. It will be faster, more robust, will run on several cores, will be much more simple
to deploy.

So the changes are:

* no more structure backends - all project structure must be set in configuration file under "projects" key.
* new web interface. In its [own repo](https://github.com/centrifugal/centrifuge-web). This is ReactJS based single-page application.
* simplified structure settings - some of them renamed, some completely removed - you can use client-side (pure js) converter for structure [on JSFiddle](http://jsfiddle.net/FZambia/17h5p75r/). See documentation for more information about option names and meanings.
* project name will be used as key for API instead of project ID. Project ID not needed anymore.
* use of hostname for node name instead of ip address.
* `md5` support for token and signs generation removed. Now only `sha256` algorithm supported.
* `client_id` and `user_id` renamed to `client` and `user` in `default_info`
* `client` renamed to `info` in published message meta information.
* default info is now `""`(empty string) instead of `"{}"`(empty object)
* SockJS 1.0 instead of 0.3.4

Also there is [new documentation](http://fzambia.gitbooks.io/centrifugal/content/) – it covers
Centrifug**o** as server implementation but can be useful to read as Centrifuge 0.8.0 works mostly
the same way.

# How to migrate:

## migrate project structure

You should migrate your current structure into configuration file under key "projects" - you can
dump all current structure using `/dumps` endpoint in old web interface and then use converter on
[JSFiddle](http://jsfiddle.net/FZambia/17h5p75r/)

Your new configuration file should look something like this:

```javascript
{
  "password": "password",
  "cookie_secret": "cookie_secret",
  "projects": [
    {
      "name": "development",
      "secret": "secret",
      "connection_lifetime": 0,
      "namespaces": [
        {
          "name": "public",
          "publish": true,
          "watch": true,
          "presence": true,
          "anonymous": true,
          "join_leave": true,
          "history_size": 10,
          "history_lifetime": 30
        }
      ]
    }
  ]
}
```

## use new web interface

Centrifuge 0.8.0 have no built-in web interface. You should use [new web interface](https://github.com/centrifugal/centrifuge-web) - it can be served with Nginx or using
Tornado static server when path to web application provided via ``--web`` command-line option:

```
centrifuge --config=config.json --web=/path/to/web/interface/app
```

## update libraries

update to new versions:

* Centrifuge javascript client >= 0.8.0
* Cent >= 0.4.0
* Adjacent >= 0.2.0

If you have any feedback about this release - feel free to write me an email. Maybe you want some features
from Centrifugo backported into Python version.

v0.7.0
======

Some backwards incompatible changes here.

* new Pusher-like private channel auth mechanism - see [documentation](http://centrifuge.readthedocs.org/en/latest/content/private_channels.html).
* new connection check mechanism - see [documentation](http://centrifuge.readthedocs.org/en/latest/content/connection_check.html)
* insecure mode to use Centrifuge as simple solution for real-time demos, personal presentations, testing ideas etc. This mode allows to connect to Centrifuge without token, timestamp and user parameters. This mode turns on ``anonymous`` access option for all channels and turns on ``publish`` option for all channels. See [docs](http://centrifuge.readthedocs.org/en/latest/content/insecure_mode.html) and [example](https://github.com/centrifugal/centrifuge/tree/master/examples/insecure_mode).
* support for `sha256` in signs and tokens HMACs - sha256 is now used by default in new version of Cent - read new [chapter](http://centrifuge.readthedocs.org/en/latest/content/tokens_and_signatures.html) about various signatures in Centrifuge.
* Redis engine now can listen for API commands - Redis must be firewalled and only owner should use this as there is no secret key based sign check. Due to use of queue mechanism for this you can only use commands for which you don't need response body or error - ex. `publish`, `unsubscribe`...
* Tornado updated to version 4.1
* do not publish if channel name is empty string
* `pre_publish_callbacks` and `post_publish_callbacks` now accept 2 arguments - `project_id` and `message`

How to migrate
--------------

* update Cent client to latest version (>=0.3.0). If you don't use Cent then you should update your code.
* update javascript client to latest version (>=0.7.0).
* update Adjacent (if you use it) to latest version (>=0.1.0)
* if you are using old private channel subscription mechanism via POST request from Centrifuge then you should migrate using new one.
* if you are using connection check then you should adapt your code to use new mechanism
* note that `user_info` kwarg renamed to `info` in Cent `generate_token` function
* note that `user_info` kwarg renamed to `info` in Adjacent `get_connection_parameters` function
* ``path`` option in File structure renamed into ``file`` to not overlap with ``path`` in Sqlite structure backend.

How to start Centrifuge in insecure mode
----------------------------------------

```
CENTRIFUGE_INSECURE=1 centrifuge --logging=debug --debug --config=config.json
```

How to publish via Redis engine API listener
--------------------------------------------

Start Centrifuge with Redis Engine and `--redis_api` option:

```
CENTRIFUGE_ENGINE=redis centrifuge --logging=debug --config=config.json --redis_api
```

Then use Redis client for your favorite language, ex. for Python:

```python
import redis
import json

client = redis.Redis()

to_send = {
    "project": "1d88332ec09e4ed3805fc1999379bcfd",
    "data": [
        {
            "method": "publish",
            "params": {"channel": "$public:chat", "data": {"input": "hello"}}
        },
        {
            "method": "publish",
            "params": {"channel": "events", "data": {"event": "message"}}
        },
    ]
}

client.rpush("centrifuge.api", json.dumps(to_send))
```

So you send JSON object with project ID as a value for `project` key and list of commands as a value for `data` key.

Note again - you don't have response here. If you need to check response - you should use HTTP API.
For example, it's absolutely useless to call `namespace_list` using this.

`publish` is the most usable command in Centrifuge (btw at our work in Mail.Ru we only
use `publish` method) so Redis API listener was invented with primary goal to reduce HTTP overhead
when publishing quickly. This can also help using Centrifuge with other languages for which we don't
have HTTP API client yet.


v0.6.3
======

* new command-line option `address` to specify the address to bind to ([pull request](https://github.com/centrifugal/centrifuge/pull/57))
* it's now possible to provide `tornado_settings` in configuration file to pass initialization parameters to Tornado's HTTPServer instance ([pull request](https://github.com/centrifugal/centrifuge/pull/60)) :

```
{
    "password": "admin",
    "cookie_secret": "secret",
    "api_secret": "secret",
    "tornado_settings": {
        "xheaders": true
    }
}
```
* Toro dependency updated to version 0.8


v0.6.2
======

* refactor authorization logic (POST request to `auth_url_address`) when using `is_private` namespace option. 403 response code from web application now results in immediate return without permission to subscribe on channel.


v0.6.1
======

* possibility to override security related options via environment variables (see example below)
* Toro dependency updated
* Use new SockJS cdn if no explicitly provided (cdn.sockjs.org will be retired soon)

```bash
CENTRIFUGE_PASSWORD=x45d CENTRIFUGE_COOKIE_SECRET=longlongrandomstring CENTRIFUGE_API_SECRET=54sfg centrifuge --logging=DEBUG
```

v0.6.0
======

* use Tornado 4 (redis client, sockjs-tornado and other requirements updated too)
* Bootstrap 3.2.0
* show alert messages in web interface after form submissions
* set new logo as favicon

v0.5.9
======

* centrifuge logs now look at tornado `logging` option and behave accordingly
* Dockerfile (thanks [Brett Ausmeier](https://github.com/bausmeier))
* remove redis dependency on Windows - as it's not supported
* show Centrifuge version in web interface

v0.5.8
======

* refactor history expire heap checking

v0.5.7
======

* history is now disabled by default - this will prevent extra memory usage in simple setups - where history is not used
* `history_expire` is now 1 hour by default instead of 24 hours
* send connection transport name counters into graphite
* use `ujson` if available for JSON encoding/decoding 

v0.5.6
======

* tornado version updated
* six requirement updated

v0.5.5
======

* `/loads` handler improvements - structure now uses default values for optional project/namespace parameters
* show HTTP API endpoint in web interface


v0.5.4
======

* fix anonymous field updating via API


v0.5.3
======

* register anonymous connections, do not check anonymous connection expiration
* add `anonymous access` option to project/namespace settings


v0.5.2
======

* Centrifuge now collects various metrics and has an option to log them or to export them into Graphite
* New optional `--name` launch option to give your node human readable unique name (will be used in web interface and Graphite data path)
* History for inactive channels now expires to prevent permanent memory grows (expiration time is configurable via project/namespace settings).
* Tox for testing

At moment Centrifuge collects following metrics:

* broadcast - time in milliseconds spent to broadcast messages (average, min, max, count of broadcasts)
* connect - amount and rate of connect attempts to Centrifuge
* messages - amount and rate of messages published
* channels - amount of active channels
* clients - amount of connected clients
* unique_clients - amount of unique clients connected
* api - count and rate of admin API calls


v0.5.1
======

* Redis engine now supports redis_url command line option

v0.5.0
======

As usual I've broken backwards compatibility again! I'm so sorry for this, but this is all for the great good.

Here is a list of changes:

* MIT license instead of BSD.
* ZeroMQ is not supported by main repository anymore. You can write your own engine though.
* Engine backends which now combine state and PUB/SUB - there are two of them: Memory engine and Redis engine.
* Engine and structure storage backend are now set up via environment variables when starting Centrifuge.
* Connection parameters must contain `timestamp` - Unix seconds as string.
* Experimental support for expiring connections. Connections can now expire if project option `connection_check` turned on.
* Centrifuge admin api can now work with list of messages instead of single one.
* Javascript client now supports message batching.
* New client API `ping` method to prevent websocket disconnects on some hosting platforms (ex. Heroku)
* New admin API `disconnect` method - disconnect user by user ID. Note, that this prevents official javascript client from reconnecting. But user can theoretically reconnect to Centrifuge immediately and his connection will be accepted. This is where connection check mechanism required.
* No more namespaces in protocol. Now namespaces are virtual - i.e. if channel name starts with `namespace_name:` then Centrifuge backend will search for its settings.
* Tornado updated to version 3.2 - this means that websockets become faster due to Tornado Websocket C module
* MongoDB and PostgreSQL structure backends must be installed from their own packages from Pypi.
* And really sweet - private channels for users without sending POST request to your web app

As you can see there are lots of important changes, so I hope you forgive me for migration inconveniences.

Migration notes:

* read updated documentation
* update Cent client to the latest version
* update javascript client to the latest version
* it's recommended to flush your structure database
* fix your configuration file to fit new changes
* `magic_project_param` configuration setting renamed to `owner_api_project_param`
* `magic_project_id` configuration setting renamed to `owner_api_project_id` - no more magic.

#### What does it mean that there are no namespaces in protocol anymore?

In the earliest versions of Centrifuge to publish message you should send something like this
via admin API:

```javascript
{"namespace": "private", "channel": "secrets", "data": {"message": "42"}}
```

Now you must do the same in this way:

```javascript
{"channel": "private:secrets", "data": {"message": "42"}}
```

I.e. like from browser.

#### Why the hell you dropped ZeroMQ support?


Because of several reasons:

* ZeroMQ is hard to configure, it has nice features like brokerless etc but I think that it is not a big win in case of using with Centrifuge.
* It's relatively slow. Redis is much much faster for real-time staff.
* To have history and presence support you will anyway need Redis.

#### How can I make private channel for user without sending POST request to my web app?

This is very simple - just add user ID as part of channel name to subscribe!

For example you have a user with ID "user42". Then private channel for him will be
`news#user42` - i.e. main channel name plus `#` separator plus user ID.

`#` in this case special symbol which tells Centrifuge that everything after it
must be interpreted as user ID which only can subscribe on this channel.

Moreover you can create a channel like `dialog#user42,user43` to create private channel
for two users.

BUT! Your fantasy here is limited by maximum channel length - 255 by default (can be changed
via configuration file option `max_channel_length`).


### Where can I found structure backends for MongoDB and PostgreSQL

MongoDB backend: https://github.com/centrifugal/centrifuge-mongodb

PostgreSQL backend: https://github.com/centrifugal/centrifuge-postgresql


v0.4.2
======

* it's now possible to specify Redis auth password for state and pubsub backends [pull request by Filip Wasilewski](https://github.com/FZambia/centrifuge/pull/23)
* it's now possible to specify PostgreSQL connection params as database url [pull request by Filip Wasilewski](https://github.com/FZambia/centrifuge/pull/24)
* now Centrifuge can be deployed on Heroku backed with Redis and PostgreSQL

The recipe of deploying Centrifuge on Heroku can be found here: https://github.com/nigma/heroku-centrifuge

The final result is available here: [centrifuge.herokuapp.com](https://centrifuge.herokuapp.com/)

v0.4.1
======

* python 3 fixes (thanks to [Filip Wasilewski](https://github.com/nigma))

v0.4.0
======

Backwards incompatible! But there is a possibility to migrate without losing your current
structure. Before updating Centrifuge go to `/dumps` location in admin interface and copy and save
output. Then update Centrifuge. Create your database from scratch. Then run Centrifuge, go to `/loads`
location and paste saved output into textarea. After clicking on submit button your previous structure
must be loaded.

Also now structure backends are classes, so you should change your configuration file according
to [current documentation](http://centrifuge.readthedocs.org/en/latest/content/configuration.html#configuration-file).

* Structure storage refactoring
* Fix API bugs when editing project ot namespace
* Node information and statistics in web interface

v0.3.8
======

Security fix! Please, upgrade to this version or disable access to `/dumps` location.

* auth now required for structure dump handler

v0.3.7
======

Backwards incompatible! Cent 0.1.3 required.

* no base64 decode for incoming API requests
* it's now possible to override sockjs-tornado settings from Centrifuge config file using `sockjs_settings` dictionary
* attempt to fix some possible races

v0.3.6
======

* handling exceptions when sending messages to client
* fix bug in application connections - which resulted in incorrect unsubscribe command behaviour

v0.3.5
======

* pyzmq 14.0.1

v0.3.4
======

* pyzmq 14.0.0
* added timestamp to message
* info connection parameter support for dom plugin
* some important fixes in documentation

v0.3.3
======

Backwards incompatible! Cent 0.1.2 required.

* extra parameter `info` to provide information about user during connect to Centrifuge.
* history messages now include all client's information
* fix Python 3 TypeError when sending message as dictionary.
* change sequence token generation steps to be more semantically correct.

This release contains important fixes and improvements. Centrifuge client must
be updated to repository version to work correctly.

Now you can provide extra parameter `info` when connecting to Centrifuge:

```javascript
var centrifuge = new Centrifuge({
    url: 'http://centrifuge.example.com',
    token: 'token',
    project: '123',
    user: '321',
    info: '{"first_name": "Alexandr", "last_name": "emelin"}'
});
```

To prevent client sending wrong `info` this JSON string must be used
while generating token:

```python
def get_client_token(secret_key, project_id, user, user_info=None):
    sign = hmac.new(six.b(str(secret_key)))
    sign.update(six.b(project_id))
    sign.update(six.b(user))
    if user_info is not None:
        sign.update(six.b(user_info))
    token = sign.hexdigest()
    return token
```

If you don't want to use `info` - you can omit this parameter. But if you omit
it make sure that it does not affect token generation - in this case you need
to generate token without `sign.update(six.b(user_info))`.


v0.3.2
======
* Base State - run single instance of Centrifuge with in-memory state storage

Now single instance of Centrifuge can work without any extra dependencies
on ZeroMQ or Redis. This can be done using Base PUB/SUB mechanism and
Base State class.

To use Base PUB/SUB mechanism you need to use `--base` command line option
when running Centrifuge's instance:

```bash
centrifuge --config=centrifuge.json --base
```

To use Base State for presence and history you should properly fill `state`
section of configuration JSON file:

```javascript
{
    "password": "admin",
    "cookie_secret": "secret",
    "api_secret": "secret",
    "state": {
        "storage": "centrifuge.state.base.State",
        "settings": {}
    }
}
```

One more time - Base options will work only when you use SINGLE INSTANCE of
Centrifuge. If you want to use several instances you need to use Redis or
ZeroMQ PUB/SUB and Redis State class (`centrifuge.state.redis.State`).

v0.3.1
======
* web interface css improvements
* fullMessage option for centrifuge.dom.js jQuery plugin
* use absolute imports instead of relative
* fix installation when set up without extra dependencies on mongodb

v0.3.0
======
* centrifuge.dom.js - jQuery plugin to add real-time even easier.
* Base Pub/Sub for single node.
* Refactor web interface, make it more mobile and human friendly, add 'actions' section to make channel operations.
* A couple of API methods to get project and namespace by name.
* fix UnicodeDecodeError in web interface.

v0.2.9
======
* fix API bug

v0.2.8
======
* experimental structure API support
* experimental Redis support for PUB/SUB
* setup.py options to build Centrifuge without ZeroMQ, PostgreSQL, MongoDB or Redis support.
* javascript client now lives on top of repo in a folder `javascript`
* rpm improvements

v0.2.7
======
* fix unique constraints for SQLite and PostgreSQL
* add client_id to default user info

v0.2.6
======
* fix handling control messages

v0.2.5
======
* fix AttributeError when Redis not configured
* doc improvements
* fix possible error in SockJS handler

v0.2.4
======
* fix unsubscribe Client method
* decouple ZeroMQ specific code into separate file
* use ":" instead of "/" as namespace and channel separator.

v0.2.3
======
* use only one ZeroMQ SUB socket for process instead of having own socket for every client.
* join/leave events for channels (structure affected).
* fix bug with Centrifuge javascript client when importing with RequireJS.
* possibility to provide structure in configuration file (useful for tests and non-dynamic structure configuration)

v0.2.2
======
* fix project settings caching in Client's instance class.
* fix unsubscribe admin API command behaviour.
* repo clean ups.

v0.2.1
======
* ping fix

v0.2.0
======
* global code refactoring.
* presence support for channels.
* history support for channels.
* Simple javascript client to communicate with Centrifuge.
* Bootstrap 3.0 for web interface.
* SQLite for structure store - now no need in installing PostgreSQL or MongoDB.
* Categories renamed into namespaces.
* Possibility to set default namespace.

v0.1.2
======
* use SockJS for admin connections instead of pure Websockets(lol, see v0.0.7)

v0.1.1
======
* Update Motor version

v0.1.0
======
* As SockJS-Tornado can handle raw websockets - WebsocketConnection class
was removed, examples and Nginx configuration updated.

v0.0.9
======
* State class to keep current application's data in memory instead of
using database queries every time.
* Small code refactoring in rpc.py

v0.0.8
======
* small admin web interface improvements

v0.0.7
======
* use Websockets in admin interface instead of SockJS

v0.0.6
======
* completely remove user management

v0.0.4
======
* changes to run on Python 3
* authentication via Github OAuth2.
* exponential backoff within tornado process.
* allow empty permissions while authentication to give full rights to client.
* PostgreSQL database support.
* different bug fixes.


v0.0.1
=====
Initial non-stable release.

