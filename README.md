# NOTICE Not production ready 

# NOTICE This document is under development as well. The purpose right now is to
get my rough framework on paper for me to develop from. Expect this to be
modified and contain errors.

# Python Simple scalable secrets store solution for snakes 

Until the notice at the top goes away, you should place absolutely no trust in
this software. Not only is it "use at your own risk" but I'm basically saying
"this may eat your company if you sneeze wrong". This is a toy until otherwise
noted.

### Background 

There are a number of secrets management solutions - some managed SaaS and some
provided as packages for you to run on your own. All of these solutions (even
the ones that claim to be open source) have features that require a contract,
maintenance agreement, NDA, and/or money in order to access. This is licensed as
GPL in the hopes that no one will ever be able to sell features of this (maybe
creating an AWS Beanstalk or similar with some best of breed configuration and 
selling that might be feasible, but never this software itself).

#### What is secrets management?

At it's core, there's some key/value store with encryption. Some software needs
to manage the encryption keys to those values, rotate keys, rotate secrets,
authorize users/machines to deliver plaintext secrets to them, be aware of
authentication mechanisms, have some sort of policy language to allow CRUD of
secrets, plugin to different components to create/delete temporary credentials
for them, create verifiable logs for any event related to secrets.

There is an initialization issue whereias you don't want to store the secret to
decrypt your secrets on disk. There are a number of ways to handle this: let an
admin user enter the initial secret manually, keep the initial secret in another
secrets manager (such as KMS), or let a hardware device (such as an HSM) keep 
your initial secret. This is a large part of where the other "free" solutions
start to fail (more on this later).

Lastly, there needs to be some way for clients (servers) to inject secrets into
configuration files and for client software to get updates of secret rotation
(such as using a custom JDBC in DBeever).

#### Why python?

Primarily because all of the components I think should be required for this
project are very popular modules. But also because no one has written a secrets
manager in python, and I figure that it's nice to give people choice - if you
want to write code for a secrets manager in go you'll look to Hashicorp/IBM
Vault or OpenBAO, if you want to write in .Net you'll use BitWarden, and
TypeScript is Infisical. Lastly because everyone I know who works with computers
can write python and I'm hoping this will speed up development.

Python allows for asyncronous code (via asyncio and Threads) but is limited to a
single core (which is similar to golang - go uses ProtoBuf/grpc to handle this).
There are a few job management solutions for python. The most popular is Celery
using Redis. That's what I intend to use in order to allow multiple crypto and
authn operations to take place without slowing other things down. I also hope to
allow separation of trust by splitting up components to run on separate
computers - this should allow for both setups.

The Cryptography library for python is also very good. AWS famously has Boto2
for python, but Google has gcloud and Azure has the azure library for python -
so dealing with IAM is already well documented. Using Celery with Flask is
fairly popular and well documented, but there's also Robyn or Sanic web servers
if Flask proves too slow. It shouldn't be too hard to wire up tons of different
authenttication mechanisms (including the major cloud providers, OIDC, SAML,
etc). Creating a client is pretty obvious with jinja2.

#### Why the name?

Mainly because I want to make things easy to search for in the future. Someone
already took up other "s" names (though I could've prepended py* to that and
been fine - that still adds confusion). So, I figure PySSSSSS is good enough.
Maybe I'll have multi color s shaped pythons or a snake with a hissing thought
bubble as a logo too. Maybe it'll be referred to as octs or octal-s. And I fully
expect IBM sales reps to call this "piss" as they're trying to convince people
to hand over >$100k per year (which I'd be thrilled to hear about).

#### Why not use an existing product?

Basically, because no solution offers HSM support for less than the cost of a
new car per year. And really, I would've been willing to pay $100/year for
personal use of their product with an HSM. But lets go down the list for
completeness.

Hashicorp Vault - I may find the email where I requested a quote for a
non-business license of Vault Enterprise specifically so that I could use an HSM
with it (and not sentinel or namespaces or any of their other enterprise
features). They charge 10s of thousands of dollars on a yearly contractual basis
for their enterprise product though.

OpenBOA - HSM support is still an open issue https://github.com/openbao/openbao/issues/16

Infisical - HSM support is still an open issue https://github.com/Infisical/infisical/issues/1254
However they also sell support for simple features like OIDC login, so I'm
skeptical that they'll offer HSM support in their open source offering.

Bitwarden - Offers HSM support as part of their Key Connector https://github.com/bitwarden/key-connector
Which requires a license to setup https://bitwarden.com/help/deploy-key-connector/#activate-key-connector
(I haven't asked how much that costs though - maybe it's cheap).

So, I am unaware of a secrets management solution that offers HSM support 
for a price an individual could afford.

### Design

This section might be a bit more hand-wavy than I'd like (I'll define schemas
and flow charts in the future). But I think it's a good start.

#### Architectures concerns

Python runs on a single core. Cryptographic operations are time consuming by
nature. It would also be nice to delegate trust to modules or microservices. It
is for this reason that I am using celery.

Celery uses redis which can use a username and password (which we will use).
However, the application will check message validity. Messages will contain a
signature - messages will also contain a unique key name. In order to derrive
the signature, the message (including the key name but excluding the signature)
is base64'd and signed. Applications can determine whether it trusts a message
type based on the unique key name and then verify the signature works.

When starting a worker for the first time, a cert will be generated (TODO HSM).
The cert path and cert fingerprint will always be logged. You then need to enter
this information into components that are supposed to trust the data the worker
generates. A rogue worker (with the redis credentials) may still pull messages
off the queue and cause a DoS like event, but we can still ensure that
compromising a single node doesn't compromise data. The web server front end may
still read any secret passed through it. To get around this, we may allow
backend components to negotiate tls with clients and proxy that message through
(unsure).

Celery allows us to define multiple queues and the cert name is also the name of
the celery queue. We then associate this path/name with the key path that this
worker/group is responsible for. So a worker "foo" might be responsible for
"/keys/foobar/*, /keys/baz/*" and "authn" might be associated with "/login/*",
and "authz" with "/policy/*". I'll need to determine whether I want to provide a
mechanism to delegate decryption to a sub-worker so that the worker asks the
sub-worker to do the actual work (maybe over a different protocol). I probably
want to put hints in the worker names so that "authz.oidc" just authorizes
people coming from an oidc login, etc.

The secrets database can be any key/value store. The values are always
encrypted. It is up to the user if or what keys are encrypted. Assuming keys are
encrypted, there is always one unencrypted key which is the name of the worker
responsible for those keys and the value contains the uuids and names/paths of
the keys that worker is responsible for. It would probably be good to name the
worker responsible for a set of keys a subset of that path - such as a worker
called "keys/foo/" to manage keys in "keys/foo/bar/*" and "keys/foo/baz/*". I'm
unsure that I want to go around and check workers for who has the record for a
key (and should be able to manage it).

#### Node type 

There needs to be the front end web server/management interface. Besides that,
these should be celery workers. We need authn (authentication) which will be
called when the "/login" path is accessed and will authenticate a user to
whatever backkend (it's possible we'll want to allow aliasing things like
/login?type=oidc to /oidclogin or something like that - not sure - I think not
being to disable auth types in the HC vault webui was pretty dumb, so I intend
to make that customizable). The authn node(s) populate (and sign) a temporary
user cookie (based on the policy and what's asked for) and returns the user the
token. That data is then passed to the authz (authorization) worker when the
user tries to do something with that token. The authz worker then takes the crud
action and path that is being requested and looks for the policy for that
location and verify that the action is acceptable (we need more work to
determine exactly how this should work - possibly authz:/foo/bar to have
separate authz workers for separate paths). Assuming that's successful, a call
will be made to the worker for the key that can crud the data. 

Most workers are for key/value data. There may be time based backend workers
(such as database temporary access credentials) where another worker updates
that key/value on some frequency. That update needs to be encrypted with a key
that the worker for that path can decrypt for the user, but otherwise there's
nothing novel here. This is similar for temporary webhooks or ldap credentials
etc. However, there are actions like asking to sign a CSR or create temporary
IAM credentials that proxy through to a library and/or service call at the time
of the request and aren't actually data we need to store. We'd want to pass IAM
or CSR and other proxied calls through our secrets server so that all policies
are looked at with the same rigor and access is controlled with the same
mechanisms and logs are signed the same way etc.

#### Policies

All actions are CRUD. These actions use standard http requests. Create is POST,
Read is GET, Update is POST too (users shouldn't be able to actually delete
data), and Delete is DELETE (which should just mark a key as deleted so that it
doesn't show up when we GET a path to list the contents). 

As mentioned above, authn mechanisms populate a table of user attributes along
with the token returned. The authz worker looks up the policy for the path. It
then runs this policy against the attributes and action being attempted and then
returns an ok or failure. While I was hoping there was a pythonic policy
mechanism to use here, it appears there's not and using OPA's Rego might be the
way to go. 

To that end, there should be a policy worker that (assuming you have POST access
to the "/policy/keys/foo" or whatever path) signs and tests and stores new
policies a user might want to create. If we do end up using opa policies
compiled into wasm, this worker should also be able to uncompile and return a
plain text version.

