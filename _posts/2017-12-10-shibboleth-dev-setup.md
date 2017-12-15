---
layout: default
title: SAML Authentication for Phoenix
permalink: /blog/auth/saml-auth-for-phoenix
image: /assets/images/authentication.jpg
categories: auth saml elixir phoenix shibboleth
description: Enable SAML authentication for Elixir/Phoenix applications.
---
{% if page.index_page %}
[![Authentication](/assets/images/authentication.jpg)](/blog/auth/saml-auth-for-phoenix)
{% else %}
![Authentication](/assets/images/authentication.jpg)
{% endif %}<sup><sup>
  Photo by [William Iven](https://unsplash.com/@firmbee)
  on [Unsplash](https://unsplash.com)
</sup></sup>
<br/>
Want to enable SAML 2.0 SSO authentication in your Elixir/Phoenix application?
It is fairly easy to do using the [`samly`](https://github.com/handnot2/samly)
Elixir library.

`Samly` is known to work with
[`SimpleSAMLphp`](https://simplesamlphp.org/),
[`Sibboleth`](https://www.shibboleth.net/). It has been tried with a few
cloud based commercial SAML providers as well.

Let us get a bit of terminology clarified up front. Your Phoenix/Plug-based
application will be known as the "SAML Service Provider" (SAML SP) or
"Relying Party" (RP) that relies on a SAML Identity Provider (SAML IdP)
for authentication. This IdP in turn uses an authoritative source
such as LDAP for user authentication.

The IdP publishes the HTTP endpoints to be used for communications. The endpoint
information along with the certificate are typically made available in an XML file.
This XML should be registered with or provided as configuration in SP. This is how
SP knows how to communicate with IdP. Similarly, SP publishes the endpoints
and certificate that IdP can use for communication. The "SP/RP registration"
process used by IdPs differs from vendor to vendor.

When the end user successfully authenticates and consents to provide
the user attributes, IdP sends the user attributes as part of SAML Assertion
in its response to SP (sent as a redirect or POST).

With that out of the way, let us move on to the mechanics.

### First things First

You need an IdP to talk to - be it one of the cloud SAML providers or your
corporate SAML provider. Fear not if you don't have one. We will cover
how to setup a Docker based Shibboleth SAML Identity Provider.
This is the first part of this post.

If you already have an IdP to talk to, consider yourself really fortunate
and skip to the second part. The primary purpose of the second part is to
make sure that your get the configuration settings right to talk to your
IdP without writing your own code.

Once you have the right configuration settings, move onto incorporating
`Samly` into your own Phoenix/Plug based application.

### Part 1: Your own Shibboleth SAML Identity Provider

Skip this part if you already have a SAML IdP to work with.

We are going to use a Shibboleth Docker image for our IdP. This IdP will use
an OpenLDAP Docker container for the user repository. Let us get started by
cloning the following Github repo.

```sh
git clone https://github.com/handnot2/samly_shibboleth
cd samly_shibboleth
```

#### Setup OpenLDAP

Let us setup an LDAP instance for use with the IdP. We need to seed this
LDAP instance with an organization and a set of users. The `ldif`
directory contains a file with the seed data. Before proceeding further edit
the `ldif/add_users.ldif` file if you want to change/add user records
(simply copy the existing user records and make the needed changes).
These entries in the ldif file will be automatically seeded
when the OpenLDAP Docker container is started using the following script:

```sh
sudo bin/start_shibb_ldap.sh
```

The OpenLDAP Docker container persists its data in `samly_ldap_dv` directory.
The Docker container this script starts is named `samly_ldap`. Use this name
when working with the Docker CLI.

#### Setup Shibboleth

The Shibboleth setup we are going to perform is built using the Docker Hub Image
[unicon/shibboleth-idp](https://hub.docker.com/r/unicon/shibboleth-idp).
Use the following script to setup a mostly pre-wired Shibboleth configuration.

```sh
sudo bin/1_setup_shibb_idp.sh
```

> This command shows a sequence of prompts expecting input. **Simply accept**
> the defaults recommended within square backets with Enter/Return key.
> When prompted, enter passwords for "backchannel" and "cookie encryption".
> **Make sure to note down the backchannel PKCS12 password.**

Here is what the console output from this command looks like:

```
Please complete the following for your IdP environment:
Hostname: [shibb.idp]

Attribute Scope: [idp]

SAML EntityID: [https://shibb.idp/idp/shibboleth]

Generating Signing Key, CN = shibb.idp URI = https://shibb.idp/idp/shibboleth ...
...done
Creating Encryption Key, CN = shibb.idp URI = https://shibb.idp/idp/shibboleth ...
...done
Backchannel PKCS12 Password:
Re-enter password:
Creating Backchannel keystore, CN = shibb.idp URI = https://shibb.idp/idp/shibboleth ...
...done
Cookie Encryption Key Password:
Re-enter password:
Creating cookie encryption key files...
...done
Rebuilding /opt/shibboleth-idp/war/idp.war ...
...done

BUILD SUCCESSFUL
Total time: 23 seconds
A basic Shibboleth IdP config and UI has been copied to ./customized-shibboleth-idp/ (assuming the default volume mapping was used).
Most files, if not being customized can be removed from what was exported/the local Docker image and baseline files will be used.

Generating Front Channel Certificate ...
Generating a 4096 bit RSA private key
....................................................................................................................................................................................................................................................................................................................++
.........................++
writing new private key to 'customized-shibboleth-idp/credentials/jetty.key'
-----

Exporting Front Channel Key ...
Done

Set Backchannel PKCS12 password in:

  ./customized-shibboleth-idp/ext-conf/idp-secrets.properties

Should match what was provided earlier.
```

> Edit the `idp-secrets.properties` file mentioned above in the console output
> to set the backchannel PKCS12 password. (Use `sudo`)

The Shibboleth Identity Provider is mostly ready. We should now be able to
build the IdP Docker image and start the Identity Provider.

```sh
sudo bin/2_build_image.sh
sudo bin/3_start_shibb_idp.sh
```

> The "start" script above creates a docker container by the name
> `samly_shibb_idp`.
>
> Confirm that IdP is successfully started by checking the logs.
> It takes a little while to start the Shibboleth Identity Provider.

```sh
sudo docker logs samly_shibb_idp

# IdP is up and running when you see the following message in the logs
....
....
... INFO:oejs.Server:main: Started @15880ms
```

#### Fetch IdP Metadata File

We need to get a copy of the IdP Metadata XML file from this setup
that we can use to configure SAML SP (`samly` enabled Phoenix application).

> We need to be able to reach the Shibboleth IdP by the hostname "`shibb.idp`".
>
> :triangular_flag_on_post: Make sure to add `127.0.0.1 shibb.idp` to your
> `/etc/hosts` file. (Check if you are able to "`ping shibb.idp`".)

```sh
wget --no-check-certificate -O idp_metadata.xml https://shibb.idp/idp/shibboleth
```

> :triangular_flag_on_post: The Single Logout endpoint information in this
> `idp_metadata.xml` is commented out by default. Open this file in an editor,
> search for `SingleLogoutService`, uncomment that block and save it back.

The IdP setup is complete except for the SAML SP registration. We need to get
the metadata from SAML SP to complete this step. Instructions for this
are provided in the next part.

### Part 2: Get it right with a Demo Phoenix application

The configuration that is! As mentioned before, the demo Phoenix application
`samly_howto` serves two purposes. It is a showcase on how to use the `samly`
Elixir library. It is also useful to make sure that you get the SAML SP
configuration right when talking to the IdP.

#### Setup SAML SP - SamlyHowto Application

Follow these instructions to clone the demo application repo, generate
the keys and cert needed for `samly` and fetch the `npm` packages needed
for the web UI.

```sh
# clone this at the same level as samly_shibboleth and NOT as s sub-directory
cd ..
git clone https://github.com/handnot2/samly_howto
cd samly_howto
./gencert.sh
mix deps.get
mix compile
cd assets && npm install && cd ..
```

> :triangular_flag_on_post: We are going to use "`samly.howto`" to access
> this demo application. Make sure to add `127.0.0.1 samly.howto` to your
> `/etc/hosts` file.

#### Register Identity Provider with SAML SP

We need to provide the SAML Identity Provider metadata information to `samly_howto`.
Do this by copying the `idp_metadata.xml` file that we obtained earlier
into the `samly_howto` directory. This metadata file has the SAML endpoint URLs
as well as the certificate needed to verify the SAML responses from Idp.

```sh
cp ../samly_shibboleth/idp_metadata.xml .
```

#### Register SAML SP with the SAML IdP

The last step is to register `samly_howto` SAML SP with the SAML IdP.
Start the demo application and use `wget` command to get the SAML SP
metadata XML file with the following commands:

```sh
./runit.sh
```

Now open another terminal and follow the rest of the instructions in this
part using that terminal.

```sh
cd path/to/samly_howto
wget --no-check-certificate -O sp_metadata.xml \
  https://samly.howto:4443/sso/sp/metadata/idp1
```

> The instructions for registering the SAML SP are very specific to IdPs.
> If you are working with an already avaiable SAML IdP installation, follow the
> instructions from that vendor. All information you need for the registration
> are available in `sp_metadata.xml` file.

If you used instructions in Part One to setup your own Shibboleth based SAML IdP,
copy the `sp_metadata.xml` over to the IdP, regenerate the IdP Docker image
and restart the IdP.

```sh
sudo cp sp_metadata.xml ../samly_shibboleth/customized-shibboleth-idp/metadata/
cd ../samly_shibboleth
sudo bin/4_stop_shibb_idp.sh
sudo bin/2_build_image.sh
sudo bin/3_start_shibb_idp.sh
```

Watch the docker container logs to make sure that the IdP startup is complete.
With that the Shibboleth IdP setup is now complete!

#### Test SAML Authentication

Well, the IdP knows about SP and SP knows about IdP by now. Let us try out the
SAML authentication! Open a browser window and enter the following URL:

> [https://samly.howto:4443](https://samly.howto:4443)

You will have to temporarily trust the self-signed certificates that were
generated during the installation of both the IdP and SP.

You should see a page with a **Sign in** button. Clicking on this button
takes you through Shibboleth Identity Provider Login screen. Upon successful
login, you are prompted for your consent to release the authenticated user
attributes to the `samly_howto` Phoenix application. Pick a consent choice
and click on the **Accept** button.

> Login with the uid and password of one of the users in the ldif file.
> For example, you can login as `wflintstone` with `changeme` as the password.
> If you had changed the defaults, use your modified values.

Upon successful authentication and attribute release consent, you will be
redirected back to the demo application that shows the user attributes
available in the authenticated SAML Assertion.

Try out the **Sign out** button as well to complete the logout process.

#### Your friends: SAML Tracer Browser Plugin and Samly documentation

There is a handy SAML Tracer browser plugin that is very helpful to view the
SAML requests and responses between SP and IdP. Google it.

Make sure to check out the `samly` documentation on various configuration choices
it offers. Sometimes, you run into issues talking to SAML providers because those
providers do not sign their requests by default. `Samly` by default signs its
requests and expects signed responses. You can either make sure that signing
is enabled at IdP or turn it off at `samly`.

> Congratulations! You have a working SAML authentication setup now. The `samly`
> related config settings in the `config/dev.exs` file is what you need in your
> own Phoenix application. This takes us to the third part.

### Part 3: Integrate Samly into your application

Having gotten this far, you should not have any major surprises. Most of what is
here is covered in `samly` documentation. Start with dependency related changes
to your `mix.exs` and update the dependencies.

```elixir
# mix.exs

defp deps() do
  [
    # ...
    {:samly, "~> 0.8"}, # Check hex.pm for the latest version number
  ]
end
```

The next step is to add the Samly Provider to your application supervision tree.
This is done in your `application.ex` file:

```elixir
# application.ex

children = [
  # ...
  worker(Samly.Provider, []),
]
```

Routing comes after this. Make the following changes to setup the SAML related
routes in your application. Do this is by **literally** copying the following
to your `router.ex` file:

```elixir
# router.ex

# Add the following scope ahead of other routes
# Keep this as a top-level scope and **do not** add
# any plugs or pipelines explicitly to this scope.
scope "/sso" do
  forward "/", Samly.Router
end
```

`Samly` needs a key and certificate that it can use to sign the requests it sends
to IdP. You can use `openssl` for this or take advantage of the `gencert.sh` from
Part 2.

Now bring in the `samly` related configuration from Part 2 to your application,
making accommodations for any name changes.

> :triangular_flag_on_post: Follow the instructions in Part Two on how to
> fetch the SP Metadata XML file and register with the IdP. Also copy over
> the `idp_metadata.xml` and make it available to your application.

#### Login/Logout URLs

Check out the `samly` documentation. It explains the routing URLs. You can also
checkout the `samly_howto` demo application as your reference.

#### Protecting your application routes

`Samly` provides an API to get the SAML Assertion
[`Samly.get_active_assertion`](https://hexdocs.pm/samly/). This function returns
`%Samly.Assertion{}` if the user is authenticated and `nil` otherwise. Use this
call in an Elixir Plug to check if the user is authenticated or not and redirect
to the SP signing URL appropriately. This Plug should be part of the pipeline
that protects your routes.

### Finally! That's It

> Be sure to checkout the customization section of the `Samly` documentation.
> It covers how you can use a Plug pipeline to transform the attributes in
> the SAML assertion. You can also use this pipeline mechanism to handle
> Just-In-Time user creation in your own application.

### References

Comments and Discussion | <https://elixirforum.com/t/samly-add-saml-sso-to-your-phoenix-application-now-with-multiple-identity-provider-support/8511>
Documentation | <https://hexdocs.pm/samly/readme.html>
Samly Repo | <https://github.com/handnot2/samly>
Samly Demo Application | <https://github.com/handnot2/samly_howto>
Shibboleth 3 | <https://wiki.shibboleth.net/confluence/display/IDP30/Home>
Self-hosted Shibboleth Idp | <https://github.com/handnot2/samly_shibboleth>
SimpleSAMLphp | <https://simplesamlphp.org/>
Self-hosted SimpleSAMLphp Idp | <https://github.com/handnot2/samly_simplesaml>
Dockerized OpenLDAP | <https://hub.docker.com/r/osixia/openldap>
