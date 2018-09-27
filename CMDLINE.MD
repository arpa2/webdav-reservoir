# Commandline Control of ARPA2 Reservoir

> *This commandline description explains how you can
> manage the mixture of LDAP and Riak KV that forms the
> backend for ARPA2 Reservoir.*

The following commands were designed to allow gradual
development of the mechanisms of ARPA2 Reservoir.  It
can be built into a WebDAV or other frontend, placed
behind ARPA2-styled access control and it should work.
Note that there is no access control in these
commands, other than the LDAP access control as it is
setup.


## Environment variables

It is assumed below that the following
environment variables have been set up:

    ARPA2_DOMAIN=orvelte.nep
    ARPA2_LDAPURI=ldap://localhost:389
    ARPA2_RIAKURI=TODO
    ARPA2_AUTHNID=bakker@orvelte.nep
    ARPA2_AUTHZID=producent+bakker@orvelte.nep

These will be referenced as needed by
the commandline tool `a2reservoir`.


## Create a Domain

Run the following command:

    shell$ a2reservoir domain add
    Domain: orvelte.nep

The opposite would be:

    shell$ a2reservoir domain del

Between the last two operations it could be listed with:

    shell$ a2reservoir domain list
    Domain: orvelte.nep

This `domain add` command introduces the
domain `orvelte.nep`, found in environment
variable `ARPA2_DOMAIN`, as a
[bucket type](http://docs.basho.com/riak/kv/2.2.3/using/reference/bucket-types/)
in Riak KV.


## Create a Resource Collection

Run the following command:

    shell$ a2reservoir collection add Photos
    Collection: 80f23a20-0f0c-4ca6-85ba-37f7f543e07b

The opposite would be:

    shell$ a2reservoir collection del 80f23a20-0f0c-4ca6-85ba-37f7f543e07b

Between these two calls it could be listed with:

    shell$ a2reservoir collection list

While not yet deleted, one might query with:

    shell$ a2reservoir collection get Photos
    AccessControl: @V@ 80f23a20-0f0c-4ca6-85ba-37f7f543e07b

This `collection add` command creates an empty Resource Collection
in LDAP, named

    cn=Photos,ou=Reservoir,dc=orvelte,dc=nep

This Resource Collection is now available under the
domain `orvelte.nep` and it will be identified by
the UUID `80f23a20-0f0c-4ca6-85ba-37f7f543e07b` that
was printed by the command.

In Riak KV, the Resource Collection is mirrorred as a
[bucket](http://docs.basho.com/riak/kv/2.2.3/using/reference/bucket-types/#buckets-as-namespaces),
named `80f23a20-0f0c-4ca6-85ba-37f7f543e07b` (in lowercase) to be found underneath the
[bucket type](http://docs.basho.com/riak/kv/2.2.3/using/reference/bucket-types/)
for this domain, `orvelte.nep`.


## Simulated Nesting with Links

Run the following command:

    shell$ a2reservoir link add Photos/Holiday 80f23a20-0f0c-4ca6-85ba-37f7f543e07b
    Collection: 80f23a20-0f0c-4ca6-85ba-37f7f543e07b

The opposite would be:

    shell$ a2reservoir link del Photos/Holiday

While not yet unlinked, one might query with:

    shell$ a2reservoir collection get Photos/Holiday
    AccessControl: @V@ 80f23a20-0f0c-4ca6-85ba-37f7f543e07b

This `link add` command creates a reference from the named UUID
of a Resource Collection to the name `Holiday` in the `Photos`
Resource Collection.  In this particular example that creates
a cyclic reference, which is not problematic in itself because
the UUIDs help to quickly detect endless loops.  In real life,
one would use the UUID of something more sensible.


## Modifying Access Control Lists

Run the following command:

    shell$ a2reservoir acl add @RPKOV@ bakker@orvelte.nep 80f23a20-0f0c-4ca6-85ba-37f7f543e07b

The opposite would be:

    shell$ a2reservoir acl del @RPKOV@ bakker@orvelte.nep 80f23a20-0f0c-4ca6-85ba-37f7f543e07b

Between those commands, one might list the ACL and find this line:

    shell$ a2reservoir acl list 80f23a20-0f0c-4ca6-85ba-37f7f543e07b
    ...
    AccessControl: @RPKOV@ bakker@orvelte.nep
    ...

    shell$ a2reservoir acl get bakker@orvelte.nep 80f23a20-0f0c-4ca6-85ba-37f7f543e07b
    AccessControl: @RPKOV@

This command `acl add` creates a new ACL Rule on the ACL of the resource identified by
the UUID.  We repeat that this is not evaluated at this point, but it is part of the
[ARPA2 Service Library](https://github.com/arpa2/libarpa2service)
that is being developed to orthogonally handle access to resources like these when
they are part of a public service.

The
[Bring Your Own IDentity](TODO)
concept implies that users in another realm can also be setup with rights to this
Resource Collection, as long as they can authenticate themselves to the hosting
domain.


## Adding and Removing Objects

Run the following command:

    shell$ a2reservoir object add 80f23a20-0f0c-4ca6-85ba-37f7f543e07b Content-Type=image/jpeg < ~/photos/monkey.jpg
    Resource: 2c23fb12-f87c-4dc3-b773-0a630f6223e7

The opposite would be:

    shell$ a2reservoir object del 80f23a20-0f0c-4ca6-85ba-37f7f543e07b 2c23fb12-f87c-4dc3-b773-0a630f6223e7

One might retrieve the object with:

    shell$ a2reservoir object get 80f23a20-0f0c-4ca6-85ba-37f7f543e07b 2c23fb12-f87c-4dc3-b773-0a630f6223e7 > ~/photos/me.jpg

This command `object add` imports an object into ARPA2 Reservoir, as part of the Resource Collection
80f23a20-0f0c-4ca6-85ba-37f7f543e07b.  The binary object is loaded from standard input, and the only
required bit of metadata is supplied, namely the `Content-Type`, which may in some cases be inferred
from the contents of the supplied data.

The output of the command, here `2c23fb12-f87c-4dc3-b773-0a630f6223e7`, is the UUID that serves as
the key for the object.

This command creates an object in LDAP at

    resource=2c23fb12-f87c-4dc3-b773-0a630f6223e7,ou=80f23a20-0f0c-4ca6-85ba-37f7f543e07b,ou=Reservoir,dc=orvelte,dc=nep

The attributes of this setting include the `Content-Type` setting to `image/jpeg` and the
randomly generated `Resource`.  When the ACL is enforced, only users setup with `@C@`
rights would be able to do this, or when the `Resource` already existed, only those
with `@W@` rights could overwrite the previous value.

In addition to this LDAP entry with metadata, the object itself gets stored in Riak KV
at the following location:

    /types/orvelte.nep/buckets/80f23a20-0f0c-4ca6-85ba-37f7f543e07b/keys/2c23fb12-f87c-4dc3-b773-0a630f6223e7


## Adding and Removing Pipelines

TODO:

    shell$ a2reservoir pipeline add ...
    shell$ a2reservoir pipeline del ...

This command `pipeline add` introduces a resource that is not stored, but pipelined to a service
over AMQP 1.0.

TODO:

  * Is a pipeline a Resource or a Resource Collection?
  * Can a pipeline be read as well as written?
  * Who can create new pipelines?  Probably ACL-bound.
  * Can pipelines connect Reservoirs somehow?


## Sharing data with Remote Reservoirs

This command `remote add` introduces a link to a remote site's Reservoir.  This can
be used to pull a remote resource for which sufficiently permissive credentials have
been setup, into the local Reservoir as though it were a Resource Collection.

TODO:

  * What connection protocol?  WebDAV would be possible, and so is SFTP.  General URIs?

## Editing MetaData for an Object

Run the following command:

    shell$ a2reservoir meta add 2c23fb12-f87c-4dc3-b773-0a630f6223e7 Description='An ugly guy'

The opposite would be:

    shell$ a2reservoir meta del 2c23fb12-f87c-4dc3-b773-0a630f6223e7 Description='An ugly guy'

To retrieve metadata, one might try:

    shell$ a2reservoir meta get 2c23fb12-f87c-4dc3-b773-0a630f6223e7
    Collection: 80f23a20-0f0c-4ca6-85ba-37f7f543e07b
    Resource: 2c23fb12-f87c-4dc3-b773-0a630f6223e7
    Content-Type: image/jpeg
    Description: An ugly guy

To search based on metadata, one might try to look for images of ugly things:

    shell$ a2reservoir meta search Content-Type='image/*' Description='*ugly*'
    Collection: 80f23a20-0f0c-4ca6-85ba-37f7f543e07b
    Resource: 2c23fb12-f87c-4dc3-b773-0a630f6223e7
    Content-Type: image/jpeg
    Description: An ugly guy

This command 'meta add' introduces a new attribute to describe the resource.
If any attribute by the same name already exists, it will not be replaced.

In LDAP, metadata is an attribute on the metadata object that was created
for the Resource underneath its Resource Collection.  It is used for
LDAP searches, which is optimised for this type of query.


## Locking and Unlocking an Object

Run the following command:

    shell$ a2reservoir lock add 2c23fb12-f87c-4dc3-b773-0a630f6223e7

The opposite would be:

    shell$ a2reservoir lock del 2c23fb12-f87c-4dc3-b773-0a630f6223e7

This command `lock add` can be used on Resources that have been setup with the
`lockableObject` object class.  Locking is possible when the value of the
`lockedBy` attribute is `@` by setting the identity of the claimant in `ARPA2_AUTHZID`.
This is based on an atomic update of the object in LDAP, so a clear decision is
made whether locking is supported.

