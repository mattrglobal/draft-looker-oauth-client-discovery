---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
title: "OAuth 2.0 Client Discovery"
category: info

docname: draft-looker-oauth-client-discovery-latest
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: AREA
workgroup: OAuth Working Group
keyword:
 - oauth

author:
 -
    fullname: Tobias Looker
    organization: MATTR
    email: tobias.looker@mattr.global

 -
    fullname: Karthik Sivasamy
    organization: MATTR
    email: karthik.sivasamy@mattr.global

normative:

informative:

--- abstract

This specification defines a mechanism for an authorization server to obtain the metadata of a client, including its endpoint locations and capabilities without the need for a registration process.

--- middle

# Introduction

In the traditional OAuth 2.0 model {{!RFC6749}}, the authorization server (AS) registers and assigns an identifier to a client through a registration process, during which the authorization server records certain characteristics about the client, commonly known as its metadata.

This requirement for registration greatly reduces how dynamic the relationship between a client and authorization server can be. For instance, a client that is updating the capabilities it supports must update its registration(s) with affected authorization servers for this change to be recognized. This requirement also affects deployments that feature many clients and authorization servers whereby requiring the client to be registered with and maintain this registration with an authorization server is costly.

To enable a more dynamic relationship between a client and an authorization server, dynamic client registration via {{!RFC7591}} was introduced. This model allows a client to register dynamically with a supporting authorization server by sending a registration request. Although this mechanism does provide some benefits it also introduces new operational challenges for both the client and AS. For instance clients that interface with many authorization servers are burdened with having to manage a client identifier per authorization server and in some cases forced to re-register the same client instance multiple times due to local storage limitations. Furthermore, protecting the authorization servers registration endpoint forces other design tradeoffs, typically either the authorization server enforces some form of authentication (e.g a "registration_token") for registration requests, which is often problematic for public clients to manage/obtain. Or the authorization server permits any registration request and has to mitigate potential spam/malicious registration requests via some other mechanism.

Instead of requiring a registration process, this specification describes a model where a client identifies itself to the authorization server with its client_uri, which can be resolved to its metadata in a similar way to how an authorization server makes its metadata available to a client via {{!RFC8414}}.

The metadata for a client is retrieved from a .well-known location as a JSON {{!RFC8259}} document, which declares its endpoint locations and client capabilities, this process is described in [Obtaining Client Metadata](#obtaining-client-metadata). Once the client metadata is retrieved and processed by the authorization server, the client can interact with the authorization server like any other client.

This specification defines a new request parameter 'client_discovery' to indicate that the interacting client has no prior registration with the authorization server and instead has resolvable metadata that describes its endpoint locations and capabilities.

This specification uses the metadata elements defined in the client registration specification {{!RFC7591}} and no additional metadata fields or formats are defined in this specification.

## Conventions and Terminology

{::boilerplate bcp14-tagged}

This specification uses the terms "access token", "refresh token", "authorization server", "resource server", "authorization endpoint", "authorization request", "authorization response", "token endpoint", "grant type", "access token request", "access token response", "client", "public client", and "confidential client" defined by The OAuth 2.0 Authorization Framework {{!RFC6749}}.

The terms "request", "response", "header field", and "target URI" are imported from {{!RFC9110}}.

# Client Metadata

Clients can have metadata described in their configuration. Examples of existing registered metadata fields that a client can make use of can be found at the <eref target= "https://www.rfc-editor.org/rfc/rfc7591.html#section-4.1"> OAuth 2.0 dynamic client registration metadata IANA registry [RFC 7591]</eref>.

The client's published metadata MUST include the client_uri field as defined in section 2 of RFC7591 {{!RFC7591}}. The value of this field MUST be a URI as defined in RFC3986 {{!RFC3986}} with a scheme component that MUST be https, a host component, and optionally, port and path components and no query or fragment components. Additionally, host names MUST be domain names and MUST NOT be IPv4 or IPv6 addresses.

# Obtaining Client Metadata

A Client supporting metadata discovery MUST make a JSON document containing metadata as specified in RFC7591 {{!RFC7591}} available at a path formed by inserting a well-known URI string into the client_uri between the host component and the path component, if any. By default, the well-known URI string used is "/.well-known/oauth-client". This path MUST use the "https" scheme. The syntax and semantics of ".well-known" are defined in RFC 5785 {{!RFC5785}}. The well-known URI suffix used MUST be registered in the IANA <eref target= "https://www.iana.org/assignments/well-known-uris">"Well-Known URIs"</eref> registry.

Different clients utilizing OAuth 2.0 in application-specific ways may define and register different well-known URI suffixes used to publish client metadata as used by those applications, for example using a well-known URI string such as "/.well-known/example-configuration". Alternatively, many such clients will use the default well-known URI string "/.well-known/oauth-client", which is the right choice for general-purpose OAuth 2.0 applications.

An OAuth 2.0 client using this specification MUST specify what well-known URI suffix it will use for this purpose. The same client MAY choose to publish its metadata at multiple well-known locations derived from its client_uri, for example, publishing metadata at both "/.well-known/example-configuration" and
"/.well-known/oauth-client".

Some OAuth 2.0 applications will choose to use the well-known URI suffix "openid-federation", as described in [Compatibility Notes](#compatibility-notes).

## Client Metadata Request

A client metadata document MUST be queried using an HTTP "GET" request at the previously specified path. The OAuth 2.0 authorization server would make the following request when the client_uri is "https://client.example.com" and the well-known URI suffix is "oauth-client" to obtain the metadata, since the client_uri contains no path component:

~~~ http
GET /.well-known/oauth-client HTTP/1.1
Host: client.example.com
~~~

If the client_uri value contains a path component, any terminating "/" MUST be removed before inserting "/.well-known/" and the well-known URI suffix between the host component and the path component. The OAuth 2.0 authorization server would make the following request when the client_uri is "https://client.example.com/client1" and the well-known URI suffix is "oauth-client" to obtain the metadata, since the client_uri contains a path component:

~~~ http
GET /.well-known/oauth-client/client1 HTTP/1.1
Host: client.example.com
~~~

Using path components enables supporting multiple clients per host. This is required in some complex client configurations. This use of ".well-known" is for supporting multiple clients per host; unlike its use in RFC 5785 {{!RFC5785}}, it does not provide general information about the host.

## Client Metadata Response

The response is a set of metadata values describing client's configuration, including all valid redirection URIs and features supported by the client. A successful response MUST use the 200 OK HTTP status code and return a JSON object using the "application/json" content type that contains a set of metadata fields and values as defined in [Client Metadata](#client-metadata). Other metadata fields MAY also be returned.

Metadata fields that return multiple values are represented as JSON arrays. Metadata fields with no values MUST be omitted from the response.

An error response uses the applicable HTTP status code value.

The following is a non-normative example response:

~~~ http
HTTP/1.1 200 OK
Content-Type: application/json

{
    "client_uri": "https://client.example.com",
    "client_name": "My Example Client",
    "redirect_uris": [
        "https://client.example.com/cb",
        "https://client.example.com/cb2"
    ],
    "logo_uri": "https://client.example.com/logo.png",
    "jwks_uri": "https://client.example.com/my_public_keys.jwks",
    "example_extension_parameter": "example_value"
}
~~~

## Client Metadata Validation

The client_uri value returned in the client metadata response MUST be identical to the client_uri value into which the well-known URI string was inserted to create the URL used to retrieve the metadata. If these values are not identical, the data contained in the response MUST NOT be used.

The following sections describe the mechanism through which a client communicates its metadata discovery url.

# Authorization Request Using Client Discovery

A client can indicate to an authorization server that it has discoverable metadata in an authorization request via the "client_discovery" request parameter. Presence of this parameter in an authorization request with a value of "true" indicates to the authorization server that the "client_id" value of the authorization request is a URL encoded value corresponding to the "client_uri" for the client and if the authorization server does not already have the metadata for the identified client it can retrieve the metadata by following the procedure outlined in [Client Metadata Section](#client-metadata).

The following is a non-normative example request of a client making an authorization request to an authorization server with the "client_discovery" parameter:

~~~ http
GET /authorize?response_type=code
    &client_id=https%3A%2F%2Fclient.example.com
    &client_discovery=true
    &state=af0ifjsldkj
    &redirect_uri=https%3A%2F%2Fclient.example.com%2Fcb
HOST: server.example.com
~~~

The value of the "client_id" parameter in the authorization request MUST represent the URL encoded form of the "client_uri" value for the corresponding client. The "client_id" value MUST be URL decoded by the authorization server to obtain the "client_uri" value which can be used to resolve the client metadata as described in the [Obtaining Client Metadata](#obtaining-client-metadata) section.

**TODO stipulate new error responses**

# Token Request Using Client Discovery

A client can indicate to an authorization server that it has discoverable metadata in an token request via the "client_discovery" request parameter. Presence of this parameter in a token request with a value of "true" indicates to the authorization server that the "client_id" value of the token request represents then "client_uri" for the client and if the authorization server does not already have the metadata for the identified client it can retrieve the metadata by following the procedure outlined in [Client Metadata Section](#client-metadata).

The following is a non-normative example request of a client making an token request using "client_discovery" parameter:

~~~ http
POST /token
Host: server.example.com
Content-type: application/x-www-form-urlencoded
Accept: application/json

grant_type=authorization_code
&code=xxxxxxxx
&client_id=https://client.example.com/
&redirect_uri=https://client.example.com/redirect
&code_verifier=a6128783714cfda1d388e2e98b6ae8221ac31aca31959e59512c59f5
&client_discovery=true
~~~

The "client_id" parameter is passed to the token request during client authentication (<eref target="https://www.rfc-editor.org/rfc/rfc6749#section-3.2.1">as described in the Section 3.2.1 of [RFC6749]</eref>). Clients in possession of a client password MAY use the HTTP Basic authentication scheme as defined in RFC 2617 {{!RFC2617}} or MAY include the client credentials in the request-body to authenticate with the authorization server.

In case of any errors, error response is returned (<eref target="https://www.rfc-editor.org/rfc/rfc6749#section-5.2">as described in the Section 5.2 of [RFC6749]</eref>). TODO expand

# String Operations

Processing some OAuth 2.0 messages requires comparing values in the messages to known values.  For example, the member names in the metadata response might be compared to specific member names such as "client_uri". Comparing Unicode [UNICODE] strings, however, has significant security implications.

Therefore, comparisons between JSON strings and other Unicode strings MUST be performed as specified below:

1.  Remove any JSON-applied escaping to produce an array of Unicode
    code points.

2.  Unicode Normalization [USA15] MUST NOT be applied at any point to
    either the JSON string or the string it is to be compared
    against.

3.  Comparisons between the two strings MUST be performed as a
    Unicode code-point-to-code-point equality comparison.

Note that this is the same equality comparison procedure described in (<eref target= "https://www.rfc-editor.org/rfc/rfc8259#section-8.3"> Section 8.3 of [RFC8259]</eref>).

# Security Considerations

## TLS Requirements

Implementations MUST support TLS.  Which version(s) ought to be implemented will vary over time and depend on the widespread
deployment and known security vulnerabilities at the time of implementation.  The client MUST support TLS version 1.2 {{!RFC5246}} and MAY support additional TLS mechanisms meeting its security requirements.  When using TLS, the authorization server MUST perform a TLS/SSL server certificate check, per RFC 6125 {{!RFC6125}}.
Implementation security considerations can be found in "Recommendations for Secure Use of Transport Layer Security (TLS) and Datagram Transport Layer Security (DTLS)" {{!BCP195}}.

To protect against information disclosure and tampering, confidentiality protection MUST be applied using TLS with a ciphersuite that provides confidentiality and integrity protection.

## Impersonation Attacks

TLS certificate checking MUST be performed by the authorization server, as described in [](#tls-requirements), when making a client metadata request. Checking that the server certificate is valid for the "client_uri" URL prevents man-in-middle and DNS-based attacks. These attacks could cause a authorization server to be tricked into using an attacker's keys and endpoints, which would enable impersonation of the legitimate client.  If an attacker can accomplish this, they can access the resources that the affected client has access to by impersonating their profile.

An attacker may also attempt to impersonate a client by publishing a metadata document that contains a "client_uri" claim using the "client_uri" URL of the client being impersonated, but with its own endpoints and signing keys. This would enable it to impersonate that client, if accepted
by the authorization server.  To prevent this, the authorization server MUST ensure that the "client_uri" URL it is using as the prefix for the metadata request exactly matches the value of the "client_uri" metadata value in the client's metadata document received by the authorization server.


# Compatibility Notes

**TODO**

# IANA Considerations

The following IANA registration requests are made by this document.

## OAuth Parameters Registry

This specification registers the following parameters in the IANA "OAuth Parameters" registry defined in OAuth 2.0 RFC 6749 {{!RFC6749}}

client_discovery - Authorization request

- Parameter name: client_discovery
- Parameter usage location: authorization request
- Change controller: IESG
- Specification document(s): RFC XXXX (this document)

client_discovery - Token request

- Parameter name: client_discovery
- Parameter usage location: token request
- Change controller: IESG
- Specification document(s): RFC XXXX (this document)

## Well-Known URI Registry

This specification registers the well-known URI defined in [Obtaining Client Metadata](#obtaining-client-metadata) in the (<eref target="https://www.iana.org/assignments/well-known-uris/well-known-uris.xhtml">IANA "Well-Known URIs" registry</eref>) established by RFC 5785 {{!RFC5785}}

### Registry Contents

* URI suffix: oauth-client
* Change controller: IESG
* Specification document: [](#obtaining-client-metadata) of RFC 8414
* Related information: (none)

<< TODO - IANA registration - https://www.iana.org/assignments/well-known-uris/well-known-uris.xhtml >>

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
