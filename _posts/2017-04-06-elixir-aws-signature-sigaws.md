---
layout: default
title: Sigaws/PlugSigaws - Elixir Libraries to enable signature authentication for REST APIs
permalink: /blog/elixir/aws-signature-sigaws
image: /assets/images/signature-authentication.jpg
categories: elixir
description: Enable AWS Signature authentication for your Phoenix web resources and API endpoints.
---
{% if page.index_page == true %}
[![Signature Authentication](/assets/images/signature-authentication.jpg)](/blog/elixir/aws-signature-sigaws)
{% else %}
![Signature Authentication](/assets/images/signature-authentication.jpg)
{% endif %}<sup><sup>
  Photo by [Suganth](https://unsplash.com/@suganth)
  on [Unsplash](https://unsplash.com)
</sup></sup>
<br/>
`Sigaws` is an Elixir library that allows you to sign and verify HTTP requests.
This library does not dictate how you compose and send HTTP requests.
The signing functions in this library works with the HTTP request information
provided and return an Elixir map containing signature related parameters/headers.
This siganture information can then be merged with the original request data
and sent to the server using any HTTP client (such as HTTPoison).

A plug called `PlugSigaws` is built using `sigaws` library to enable easier
protection of Phoenix built REST API endpoints and web resources.

We are going to explore how you can enable signature authentication for your
Phoenix REST API endpoints in this post.

Let us get started with the needed setup:

> *    Add `plug_sigaws` and `sigaws_quickstart_provider` to `mix.exs`
> *    Configure the regions, services and credentials to be used by the
>      verification process.
> *    Add `PlugSigaws` to the Phoenix API pipeline in `router.ex`
> *    Update content parsers in `endpoint.ex`

### Project Dependencies

Update your `mix.exs` file with the following along with your other dependencies:

```elixir
  defp deps do
    [
      {:plug_sigaws, "~> 0.1"},
      {:sigaws_quickstart_provider, "~> 0.1"}
    ]
  end
```

### Configure Regions, Services and Credentials

Edit `config/config.exs` (or the appropriate environment specific config file).

```elixir
config :plug_sigaws,
  provider: SigawsQuickstartProvider

config :sigaws_quickstart_provider,
  regions: "us-east-1,alpha-quad,beta-quad,gamma-quad,delta-quad",
  services: "my-service,img-service",
  creds_file: "sigaws_quickstart.creds"
```

Create a text file `sigaws_quickstart.creds` at the project root directory.
If you place that file somewhere else, update the `creds_file` setting
appropriately. This is a simple text file with one credential per line.
The access key ID and secret in each line should be separated by a colon.
For example:

```
access_key_1:secret1
access_key_2:secret2
```

The quick start provider is also a GenServer that loads the credentials from
the specified file when it starts up. For this to work, we need to add this
GenServer to your supervision tree. Locate where the supervision tree is setup
and update it as shown here:

```elixir
use Application
def start(_type, _args) do
  import Supervisor.Spec

  children = [
    worker(SigawsQuickstartProvider, [[name: :sigaws_provider]]),
    # .....
  ]

  # ......
  Supervisor.start_link(children, opts)
end
```

### Update Phoenix API Pipeline

Edit your `router.ex` file and add the signature verification plug to your
API pipeline.

```elixir
pipeline :api do
  plug :accepts, ["json"]
  plug PlugSigaws
end
```

If the signature authentication is needed only for certain APIs, create a
separate pipeline for those APIs and add `PlugSigaws` plug to that pipeline.

### Update Content Parsers

If you had used the Phoenix generator to create your project, the content
parsers are setup in `endpoint.ex` file. Edit this file and update the
JSON and URLENCODED content parsers to what is shown here.

```elixir
plug Plug.Parsers,
  parsers: [PlugSigaws.Parsers.JSON, PlugSigaws.Parsers.URLENCODED, :multipart],
  pass: ["*/*"],
  json_decoder: Poison
```

These parsers are updated versions of what is included in the Plug package.
They add the raw content needed for hash computation during signature verification
as an "assign" in the connection.

`PlugSigaws` checks the signature information in the request parameters/headers
against what it computes. If this verification fails, the pipeline is
halted with an error. If it succeeds, the Phoenix pipeline flow will continue
to your API implementation.

### Take this for a spin

With these updates in place, update your dependencies and run Phoenix under `iex`.

```sh
mix deps.update --all
iex -S mix phoenix.server
```

Assuming you have an API endpoint called `http://localhost:4000/something?a=10`,
try the following in `iex`:

```elixir
# substitute with your API URL
url = "http://localhost:4000/something?a=10"

{:ok, sig_data, _} = Sigaws.sign_req(url, method: "GET",
  region: "us-east-1", service: "my-service",
  access_key: "access_key_1", secret: "secret1")

{:ok, resp} = HTTPoison.get(url, sig_data)
```

If you tried with wrong credentials (one not present in the credential file
you had created earlier) the verification should fail. The request should also
fail if it tampered with (for example, if you pass
`http://localhost:4000/something?a=20` to HTTPoison).

### Using other AWS Signature clients

You don't have to use `Sigaws` to sign the requests. Here is an example of
using `aws4` Node npm library to send the signed request to your Phoenix
REST API endpoint.

```javascript
var http = require('http'),
    aws4 = require('aws4')

var opts = {
  // signQuery: true,
  host: '127.0.0.1',
  port: 4000,
  path: '/something?a=10',
  region: 'us-east-1',
  service: 'my-service'
}

aws4.sign(opts, {accessKeyId: 'access_key_1', secretAccessKey: 'secret1'})

var get_req = http.request(opts, function(res) {
  res.pipe(process.stdout)
})
get_req.end()
```

### Exploring further

What you have seen so far should give a fairly good idea on how signature
authentication would work when enabled in your Phoenix apps.

#### Verification Provider

The verification process requires that the signature is recomputed from
the request data on the server-side. This means the server needs the signing key
that corresponds to the access key ID that was used to sign the request.
This is in a way similar to how authentication works where the authenticator
relies on a credential provider. `Sigaws` follows a similar approach.

> Use `SigawsQuickstartProvider` as your
> [starting point](https://github.com/handnot2/sigaws_quickstart_provider) and
> create your own provider to work with credentials in a database or some other
> external system.

#### Temporary Security Credentials

This is an area that could be developed further by the community if there is
enough interest. There is really no difference between secrets or temporary
tokens when it comes to the actual signing/verification processes.
The verification provider is relying on the `signing_key` callback to get
the key instead of the secret to enable capabilities like this down the line.

#### Policy based decisions

The verification plug `PlugSigaws` makes the verification context available in
the Plug connection as an "assign" (`conn.assigns[:sigaws_ctxt]`) upon success.
Any policy enforcements (additional plugs) can make use of the data in this
exposed context.

#### Usage beyond Phoenix/Plugs

The core signing and verification capability is part of `sigaws` library. While
this library relies on a passed in verification provider, it does not have any
direct dependencies on Plug, Phoenix or HTTPoison. It is possible to use them
in other elixir middlewares.

One such possibility might be its use in the **Nerves** project. For example,
the Nerves [firmware update by HTTP request](https://github.com/nerves-project/nerves_firmware_http) could potentially use this to enable signed firmware upload.

This library in its present form does not support streaming. If there is interest,
hopefully someone can pitch in to enable it.

### References

Use [Elixir Forum](https://elixirforum.com) for any discussions regarding what
you see in this post. That is a better avenue with assistance from a broader helpful
community.

Sigaws Github | <https://github.com/handnot2/sigaws>
Sigaws Doc | <https://hexdocs.pm/sigaws>
PlugSigaws Github | <https://github.com/handnot2/plug_sigaws>
PlugSigaws Doc | <https://hexdocs.pm/plug_sigaws>
Quickstart Provider | <https://github.com/handnot2/sigaws_quickstart_provider>
AWS V4 Req Signing | <http://docs.aws.amazon.com/AmazonS3/latest/API/sig-v4-header-based-auth.html>
AWS V4 URL Signing | <http://docs.aws.amazon.com/AmazonS3/latest/API/sigv4-query-string-auth.html>
Aws4 (Javascript) | <https://github.com/mhart/aws4>
AWS STS | <http://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp.html>
