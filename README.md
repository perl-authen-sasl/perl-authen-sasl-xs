# NAME

Authen::SASL::XS	- XS code to glue Perl SASL to Cyrus SASL

# SYNOPSIS

```perl
use Authen::SASL;

my $sasl = Authen::SASL->new(
       mechanism => 'NAME',
       callback => { NAME => VALUE, NAME => VALUE, ... },
);

my $conn = $sasl->client_new(<service>, <server>, <iplocalport>, <ipremoteport>);

my $conn = $sasl->server_new(<service>, <host>, <iplocalport>, <ipremoteport>);
```

# DESCRIPTION

SASL is a generic mechanism for authentication used by several
network protocols. **Authen::SASL::XS** provides an implementation
framework that all protocols should be able to share.

The XS framework makes calls into the existing libsasl.so resp. libsasl2
shared library to perform SASL client connection functionality, including
loading existing shared library mechanisms.

# CONSTRUCTOR

The constructor may be called with or without arguments. Passing arguments is
just a short cut to calling the `mechanism` and `callback` methods.

You have to use the `Authen::SASL` new-constructor to create a SASL object.
The `Authen::SASL` object then holds all necessary variables and callbacks, which
you gave when creating the object.
`client_new` and `server_new` will retrieve needed information from this
object.

# CALLBACKS

Callbacks are very important. It depends on the mechanism which callbacks
have to be set. It is not a failure to set callbacks even they aren't used.
(e.g. password-callback when using GSSAPI or KERBEROS\_V4)

The Cyrus-SASL library uses callbacks when the application
needs some information. Common reasons are getting
usernames and passwords.

Authen::SASL::XS allows Cyrus-SASL to use perl-variables and perl-subs
as callback-targets.

Currently Authen::SASL::XS supports the following Callback types:
(for a more detailed description on what the callback type is used for
see the respective man pages)

**Remark**: All callbacks, which have to return some values (e.g.: \*\*result in
`sasl_getsimple_t`) do this by returning the value(s). See example below.

- user (client)
- auth (client)
- language (client)

    This callbacks represent the `sasl_getsimple_t` from the library.

    Input: none

    Output: `username`, `authname` or `language`

- password (client)
- pass (client)

    This callbacks represent the `sasl_getsecret_t` from the library.

    Input: none

    Output: `password`

- realm &lt;client>

    This callback represents the `sasl_getrealm_t` from the library.

    Input: a list of available realms

    Output: the chosen realm

    (This has nothing to do with GSSAPI or KERBEROS\_V4 realm).

- checkpass (server, SASL v2 only)

    This callback represents the `sasl_server_userdb_checkpass_t` from the
    library.

    Input: `username`, `password`

    Output: true or false

- getsecret (server, SASL v1 only)

    This callback represents the `sasl_server_getsecret_t` from the library. Sasl
    will check if the passwords are matching.

    Input: `mechanism`, `username`, `default_realm`

    Output: `secret_phrase (password)`

    **Remark**: Programmers that are using should specify both callbacks (getsecret and checkpass).
    Then, depending on you Cyrus SASL library either the one or the other is called. Cyrus SASL v1
    ignores checkpass and Cyrus SASL v2 ignores getsecret.

- putsecret (SASL v1) and setpass (SASL v2)

    are currently not supported (and won't be, unless someone needs it).

- canonuser (server/client in SASL v2, server only in SASL v1)

    This callback name represents the `sasl_canon_user_t` from the library.

    Input: `Type of principal`, `principal`, `userrealm` and maximal allowed length of the output.

    Output: canonicalised `principal`

    `Type of principal` is "AUTHID" for Authentication ID or "AUTHZID"
    for Authorisation ID.

    **Remark**: This callback is ideal to get the username of the user using your service.
    If `Authen::SASL::XS` is linked to Cyrus SASL v1, which doesn't have a canonuser callback,
    it will simulate this callback by using the authorize callback internally. Don't worry, the
    authorize callback is available anyway.

- authorize (server)

    This callback represents the `sasl_authorize_t` from the library.

    Input: `authenticated_username`, `requested_username`, (`default_realm` SASL v2 only)

    Output: `canonicalised_username` SASL v1 resp. true or false when using SASL v2 lib
    There is something TODO, I think.

- setpass (server, SASL v2 only)

    This callback represents the `sasl_server_userdb_setpass_t` from the library.

    Input: `username`, `new_password`, `flags` (0x01 CREATE, 0x02 DISABLE,
    0x04 NOPLAIN)

    Out: true or false

## Ways to pass a callback

Authen::SASL::XS supports three different ways to pass a callback

- CODEREF

    If the value passed is a code reference then, when needed, it will be called.

- ARRAYREF

    If the value passed is an array reference, the first element in the array
    must be a code reference. When the callback is called the code reference
    will be called with the value from the array passed after.

- SCALAR
All other values passed will be returned directly to the SASL library
as the answer to the callback.

## Example of setting callbacks

$sasl = new Authen::SASL (
  mechanism => "PLAIN",
    callback => {
      # Scalar
      user => "mannfred",
      pass => $password,
      language => 1,

```perl
   # Coderef
   auth => sub { return "klaus", }
   realm => \&getrealm,

   # Arrayref
   canonuser => [ \&canon, $self ],
}
 );
```

The last example is ideal for using object methods as callback functions.
Then you can do something like this:

sub canon
{
  my ($this,$type,$realm,$maxlen,$user) = @\_;
  $this->{\_username} = $user if ($type eq "AUTHID");
  return $user;
}

# Authen::SASL::XS METHODS

- server\_new ( SERVICE , HOST = "" , IPLOCALPORT , IPREMOTEPORT )

    Constructor for creating server-side sasl contexts.

    Creates and returns a new connection object blessed into Authen::SASL::XS.
    It is on that returned reference that the following methods are available.
    The SERVICE is the name of the service being implemented, which may be used
    by the underlying mechanism. An example service therefore is "ldap".

- client\_new ( SERVICE , HOST , IPLOCALPORT , IPREMOTEPORT )

    Constructor for creating server-side sasl contexts.

    Creates and returns a new connection object blessed into Authen::SASL::XS.
    It is on that returned reference that the following methods are available.
    The SERVICE is the name of the service being implemented, which may be used
    by the underlying mechanism. An example service is "ldap". The HOST is the
    name of the server being contacted, which may also be used
    by the underlying mechanism.

**Remark**:
This and the `server_new` function are called by [Authen::SASL](https://metacpan.org/pod/Authen%3A%3ASASL) when using
its `*_new` function. Since the user has to use Authen::SASL anyway, normally
it is not necessary to call this function directly.

IPLOCALPORT and IPREMOTEPORT arguments are only available, when ASC is
linked against Cyrus SASL 2.x. This arguments are needed for KERBEROS\_V4
and CS 2.x on the server side. Don't know if it necessary for the client
side. Format of this arguments in an IPv4 environment should be: a.b.c.d;port.
See sasl\_server\_new(3) for details.

See SYNOPSIS for an example.

- server\_start ( CHALLENGE )

    `server_start` begins the authentication using the chosen mechanism.
    If the mechanism is not supported by the installed Cyrus-SASL it fails.
    Because for some mechanisms the client has to start the negotiation,
    you can give the client challenge as a parameter.

- client\_start ( )

    The initial step to be performed. Returns the initial value to pass to the server.
    Client has to start the negotiation always.

- server\_step ( CHALLENGE )

    `server_step` performs the next step in the negotiation process. The
    first parameter you give is the clients challenge/response.

- client\_step ( CHALLENGE )

**Remark**:
`client_start`, `client_step`, `server_start` and `server_step`
will return the respective sasl response or undef. The returned value
says nothing about the current negotiation status. It is absolutely possible
that one of these functions return undef and everything is fine for SASL,
there is only another step needed.

Therefore you have to check `need_step` and `code` during negotiation.

See example below.

- listmech( START , SEPARATOR , END )

    `listmech` returns a string containing all mechanisms allowed for the user
    set by `user`. START is the token which will be put at the beginning of the
    string, SEPARATOR is the token which will be used to separate the mechanisms
    and END is the token which will be put at the end of returned string.

- setpass(user, newpassword, oldpassword, flags)
- checkpass(user, password)

    `setpass` and `checkpass` is only available when using Cyrus-SASL 2.x library.

    `setpass` sets a new password (depends on the mechanism if the setpass callback
    is called). `checkpass` checks a password for the user (calls the checkpass
    callback).

    For both function see the man pages of the Cyrus SASL for a detailed description.

    Both functions return true on success, false otherwise.

- global\_listmech ( )

    `global_listmech` is only available when using Cyrus-SASL 2.x library.

    It returns an array with all mechanisms loaded by the library.

- encode ( STRING )
- decode ( STRING )

    Cyrus-SASL developers suggest using the `encode` and `decode` functions
    for every traffic which will run over the network after a successful authentication

    `encode` returns the encrypted string generated from STRING.
    `decode` returns the decrypted string generated from STRING.

    It depends on the used mechanism how secure the encryption will be.

- error ( )

    `error` returns an array with all known error messages.
    Basicly the sasl\_errstring function is called with the current error\_code.
    When using Cyrus-SASL 2.x library also the string returned by sasl\_errdetail
    is given back. Additionally the special Authen::SASL::XS advise is
    returned if set.
    After calling the `error` function, the error code and the special advice
    are thrown away.

- code ( )

    `code` returns the current Cyrus-SASL error code.

- mechanism ( )

    `mechanism` returns the current used authentication mechanism.

- need\_step ( )

    `need_step` returns true if another step is need by the SASL library. Otherwise
    false is returned. You can also use `code == 1` but it looks smarter I think.
    That's why we all using perl, eh?

# EXAMPLE

## Server-side

```perl
# The example uses Cyrus-SASL v2
# Set the SASL_PATH to the location of the SASL-Plugins
# default is /usr/lib/sasl2
$ENV{'SASL_PATH'} = "/opt/products/sasl/2.1.15/lib/sasl2";

#
my $sasl = Authen::SASL->new (
   mechanism => "PLAIN",
   callback => {
     checkpass => \&checkpass,
     canonuser => \&canonuser,
   }
);

# Creating the Authen::SASL::XS object
my $conn = $sasl->server_new("service","","ip;port local","ip;port remote");

# Clients first string (maybe "", depends on mechanism)
# Client has to start always
sendreply( $conn->server_start( &getreply() ) );

while ($conn->need_step) {
   sendreply( $conn->server_step( &getreply() ) );
}

if ($conn->code == 0) {
   print "Negotiation succeeded.\n";
} else {
   print "Negotiation failed.\n";
}
```

## Client-side

```perl
# The example uses Cyrus-SASL v2
# Set the SASL_PATH to the location of the SASL-Plugins
# default is /usr/lib/sasl2
$ENV{'SASL_PATH'} = "/opt/products/sasl/2.1.15/lib/sasl2";

#
my $sasl = Authen::SASL->new (
   mechanism => "PLAIN",
   callback => {
     user => \&getusername,
     pass => \&getpassword,
   }
);

# Creating the Authen::SASL::XS object
my $conn = $sasl->client_new("service", "hostname.domain.tld");

# Client begins always
sendreply($conn->client_start());

while ($conn->need_step) {
   sendreply($conn->client_step( &getreply() ) );
}

if ($conn->code == 0) {
   print STDERR "Negotiation succeeded.\n";
} else {
   print STDERR "Negotiation failed.\n";
}
```

See t/plain.t for working script.

# TESTING

I tested ASC (server and client) with the following mechanisms:

- GSSAPI

    Don't forget to create keytab. Non-root keytabs can be specify through $ENV{'KRB5\_KTNAME'} (Heimdal >= 0.6, MIT).

- KERBEROS\_V4

    Available since 0.10, you have to add IPLOCALPORT and IPREMOTEPORT to \*\_new functions.

- PLAIN

# SEE ALSO

[Authen::SASL](https://metacpan.org/pod/Authen%3A%3ASASL)

man pages for sasl\_\* library functions.

# AUTHOR

Originally written by Mark Adamson <mark@nb.net>

Cyrus-SASL 2.x support by Leif Johansson

Glue for server\_\* and many other structural improvements by Patrick Boettcher <patrick.boettcher@desy.de>

Please report any bugs, or post any suggestions, to the authors.

# THANKS

```
- Guillaume Filion for testing the server part and for giving hints about
  some bugs (documentation).
- Wolfgang Friebel for bother around with rpm building of test releases.
```

# COPYRIGHT

Copyright (c) 2003-5 Patrick Boettcher, DESY Zeuthen. All rights reserved.
Copyright (c) 2003 Carnegie Mellon University. All rights reserved.

This program is free software; you can redistribute it and/or modify it under
the same terms as Perl itself.
