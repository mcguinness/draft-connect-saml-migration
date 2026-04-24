---
title: "SAML 2.0 to OpenID Connect and OAuth 2.0 Migration Profile"
abbrev: "SAML-OIDC Migration"
category: info

docname: draft-mcguinness-saml-oidc-migration-profile-latest
submissiontype: independent
number:
date:
v: 3
# area: Security
# workgroup: Individual Submission

author:
 -
    fullname: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com

normative:
  RFC2119:
  RFC4648:
  RFC6749:
  RFC7591:
  RFC7662:
  RFC8174:
  RFC8176:
  RFC8414:
  RFC8693:
  RFC8707:
  OIDC-CORE:
    title: "OpenID Connect Core 1.0 incorporating errata set 2"
    target: "https://openid.net/specs/openid-connect-core-1_0-18.html"
    author:
      - ins: N. Sakimura
      - ins: J. Bradley
      - ins: M.B. Jones
      - ins: B. de Medeiros
      - ins: C. Mortimore
    date: false
  OIDC-DISCOVERY:
    title: "OpenID Connect Discovery 1.0 incorporating errata set 2"
    target: "https://openid.net/specs/openid-connect-discovery-1_0.html"
    author:
      - ins: N. Sakimura
      - ins: J. Bradley
      - ins: M.B. Jones
      - ins: E. Jay
    date: false
  OIDC-REGISTRATION:
    title: "OpenID Connect Dynamic Client Registration 1.0 incorporating errata set 2"
    target: "https://openid.net/specs/openid-connect-registration-1_0.html"
    author:
      - ins: N. Sakimura
      - ins: J. Bradley
      - ins: M.B. Jones
    date: false
  SAML2-CORE:
    title: "Assertions and Protocols for the OASIS Security Assertion Markup Language (SAML) V2.0"
    target: "https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf"
    author:
      - org: OASIS
    date: 2005
  SAML2-METADATA:
    title: "Metadata for the OASIS Security Assertion Markup Language (SAML) V2.0"
    target: "https://docs.oasis-open.org/security/saml/v2.0/saml-metadata-2.0-os.pdf"
    author:
      - org: OASIS
    date: 2005
  SAML2-SUBJ-ID:
    title: "SAML V2.0 Subject Identifier Attributes Profile Version 1.0"
    target: "https://docs.oasis-open.org/security/saml-subject-id-attr/v1.0/saml-subject-id-attr-v1.0.html"
    author:
      - org: OASIS
    date: 2019
  OIDC-FC-LOGOUT:
    title: "OpenID Connect Front-Channel Logout 1.0"
    target: "https://openid.net/specs/openid-connect-frontchannel-1_0.html"
    author:
      - ins: M.B. Jones
    date: false
  OIDC-BC-LOGOUT:
    title: "OpenID Connect Back-Channel Logout 1.0"
    target: "https://openid.net/specs/openid-connect-backchannel-1_0.html"
    author:
      - ins: M.B. Jones
      - ins: J. Bradley
    date: false

informative:
  RFC7009:
  RFC7522:
  I-D.ietf-oauth-identity-assertion-authz-grant:
  SAML2-IOP:
    title: "SAML V2.0 Metadata Interoperability Profile Version 1.0"
    target: "https://docs.oasis-open.org/security/saml/Post2.0/sstc-metadata-iop.html"
    author:
      - org: OASIS
    date: 2009
  INCOMMON-ATTRS:
    title: "InCommon Federation Attributes"
    target: "https://incommon.org/federation/attributes/"
    author:
      - org: InCommon
    date: false
  INCOMMON-OID:
    title: "InCommon Object Identifiers"
    target: "https://incommon.org/community/mace-registries/incommon-object-identifiers/"
    author:
      - org: InCommon
    date: false
  EDUPERSON:
    title: "eduPerson Object Class Specification"
    target: "https://refeds.org/specifications/eduperson"
    author:
      - org: REFEDS
    date: false
  REFEDS-RS:
    title: "REFEDS Research and Scholarship Entity Category"
    target: "https://refeds.org/category/research-and-scholarship"
    author:
      - org: REFEDS
    date: false

...

--- abstract

This document defines a migration profile for deployments that transition
existing SAML 2.0 single sign-on integrations to OpenID Connect and OAuth 2.0
while preserving an existing relying-party trust relationship. The profile
defines client metadata that binds an OAuth client to an existing SAML Service
Provider entity identifier, authorization server metadata that binds an OAuth
issuer to an existing SAML Identity Provider entity identifier, and rules for
mapping SAML subject identifiers, authentication statements, authentication
context, and selected SAML attributes to OpenID Connect claims. The profile
also defines use of OAuth 2.0 Token Exchange to obtain a refresh token, access
token, or ID Token from SAML input, and an introspection extension that
validates SAML input and returns normalized claims as JSON. Accepted SAML input
can be either a signed SAML assertion or a signed SAML `Response` wrapper that
conveys exactly one bearer assertion.

The intent is to let relying parties adopt OAuth 2.0 and OpenID Connect without
changing the underlying enterprise SSO trust model or requiring applications to
relink existing user accounts.

--- middle

# Introduction {#introduction}

Many enterprise and SaaS deployments already have a stable SAML 2.0 trust
relationship between an Identity Provider (IdP) and a Service Provider (SP).
When the same relying party later moves to OpenID Connect or OAuth 2.0, the
deployment often has to create a new client registration, a new issuer
relationship, and a new subject identifier model. In practice, this can break
subject continuity, claim release policy, assurance signaling, and operational
trust.

The Identity Assertion JWT Authorization Grant
{{I-D.ietf-oauth-identity-assertion-authz-grant}} defines a SAML-specific
interoperability path for obtaining OAuth artifacts from an existing SAML
deployment. This document extracts the migration-specific aspects of that
approach and defines them as a reusable OAuth and OpenID Connect profile.

This document does not define a new grant type, a new assertion format, or a
replacement for SAML Web Browser SSO. Instead, it defines a profile that:

* binds an OAuth client to an existing SAML SP Entity ID;
* binds an OAuth issuer to an existing SAML IdP Entity ID;
* defines how a SAML assertion can be exchanged for an OAuth 2.0 or OpenID
  Connect token;
* defines how a SAML assertion can be introspected when token issuance is not
  needed;
* preserves subject continuity and claim continuity across the migration.

Where {{saml-input-validation}} permits it, a signed SAML `Response` wrapper carrying exactly
one bearer assertion can be submitted instead of a bare assertion.

# Conventions and Terminology

## Requirements Notation

{::boilerplate bcp14-tagged}

## Terminology

This document uses the terms "authorization server", "client", "client
identifier", "issuer", and "scope" from OAuth 2.0, OAuth 2.0 Authorization
Server Metadata, Token Exchange, and OpenID Connect. It uses the terms
"Identity Provider", "Service Provider", "entityID", "Assertion", "Attribute",
"AttributeStatement", "AuthnStatement", and "AuthnContext" from SAML 2.0.

For the purposes of this document:

SAML IdP Entity ID:
: The SAML entityID of the asserting party that issues SAML assertions for the
  deployment.

SAML SP Entity ID:
: The SAML entityID of the relying party deployment that receives SAML
  assertions. In common SAML deployments, this is also the audience value used
  for that relying party.

Migration Client:
: An OAuth 2.0 or OpenID Connect client that has a `saml_sp_entity_id`
  registered and represents an existing SAML SP relationship. This document
  uses "client" as a shorthand for Migration Client when the context is clear.

Subject Continuity:
: The property that the same end-user is represented to the migrated relying
  party by a stable identifier that does not require account relinking.

Local Account:
: The authorization server's internal account record for the end-user to which
  the SAML assertion is linked for token issuance and claim release.

Stable Local Subject Key:
: A non-reassignable local identifier for the resolved Local Account, used as
  input when this profile requires a derived subject value.

# Applicability and Deployment Model {#applicability}

This profile assumes that an administrative authority operates, or tightly
governs, both:

* a SAML 2.0 IdP identified by a SAML IdP Entity ID; and
* an OAuth 2.0 Authorization Server or OpenID Provider identified by an OAuth
  issuer.

The profile establishes three bindings:

* issuer binding: the OAuth issuer is bound to the SAML IdP Entity ID;
* client binding: the OAuth client is bound to the SAML SP Entity ID;
* subject and claim binding: OpenID Connect subject and claim values are bound
  to the identifiers and statements that were already used for the SAML SP.

When these bindings are present, the authorization server can preserve subject
format, claim release policy, and assurance semantics from the prior SAML
deployment instead of treating the migrated client as a wholly new relying
party.

This profile is intended for migrations in which the SAML IdP relationship and
the OAuth authorization server relationship are known to correspond to the same
relying-party deployment. It is not intended to establish trust between
otherwise unrelated SAML and OAuth parties.

This profile also assumes that the authorization server has a deterministic way
to resolve assertions from that SAML deployment to exactly one active Local
Account before issuing OAuth tokens or returning an active introspection
response, and that the asserted subject already corresponds to an existing
Local Account under the authorization server's control.

This profile is therefore applicable only when the authorization server can
identify exactly one existing Local Account using either a previously persisted
linkage for the trusted SAML deployment or at least one stable, policy-accepted
SAML subject input present in the assertion.

This profile can be applied when the authorization server receives either:

* a signed SAML `Assertion`; or
* a signed SAML `Response` containing exactly one bearer `Assertion`.

When a signed SAML `Response` is used to convey or protect that Assertion,
any deployment-required validation of response-level protocol fields
such as `Destination` or `InResponseTo` remains outside this profile unless the
deployment has separately established how those checks are performed. However,
if a signed SAML `Response` wrapper is used, this profile does require the
authorization server to validate the `Status` element as defined in
{{saml-input-validation}}.

This profile is defined only for confidential clients. Public clients are out
of scope and MUST NOT use the Token Exchange or direct SAML assertion
introspection patterns defined by this document.

This profile requires SAML XML signatures sufficient to validate the effective
assertion under {{saml-input-validation}}. That requirement can be satisfied
either by a signature on the assertion itself or, when permitted by
{{saml-input-validation}}, by a signed SAML `Response` wrapper containing
exactly one effective assertion.

This profile binds exactly one `saml_idp_entity_id` to an OAuth issuer.
Deployments that need to bridge multiple SAML Identity Providers require
separate OAuth issuer instances, each with its own `saml_idp_entity_id`
binding.

## Authorization and Consent Model {#authorization-consent-model}

The authorization server MUST NOT treat a valid SAML assertion as end-user
consent to issue OAuth tokens, release OpenID Connect claims, or return an
active introspection response. The assertion proves the authentication event and
supplies identity inputs; authorization remains an authorization server policy
decision.

Before issuing a token or returning an active introspection response, the
authorization server MUST have an explicit authorization basis. That basis:

* MUST be bound to the authenticated client, the `saml_sp_entity_id`, and the
  resolved Local Account;
* for Token Exchange, MUST be bound to the requested token type, requested
  scope, and, for access tokens, the resolved target service set; and
* for introspection, MUST authorize the authenticated client to validate the
  presented SAML input and receive any returned claims for the bound
  relying-party relationship.

The authorization basis MAY come from static configuration, administrative
consent, a previously recorded authorization grant at the same authorization
server, or another policy mechanism that is established before the SAML input is
processed.

The authorization server MUST NOT create, expand, or infer an authorization
grant solely from the presence, validity, or contents of the SAML assertion.
SAML attributes MAY be used as inputs to an already-established authorization
policy rule, but the policy rule itself MUST exist independently of the
assertion presented in the current request.

For requests involving refresh tokens, ID Tokens, OpenID Connect scopes, target
services, or claim release, the authorization server MUST apply the following
additional rules:

* if the requested scope includes `offline_access`, policy MUST authorize
  refresh token issuance to the authenticated client for the resolved Local
  Account and bound `saml_sp_entity_id`;
* a prior SAML SSO event by itself MUST NOT authorize refresh token issuance;
* if the requested token type is
  `urn:ietf:params:oauth:token-type:id_token`, or if an access token request
  includes `openid`, policy MUST authorize release of the resulting OpenID
  Connect subject and claims to the authenticated client under the bound
  `saml_sp_entity_id`;
* the authorization server MUST NOT release claims solely because corresponding
  SAML attributes are present in the assertion; and
* if the requested OAuth scope, OpenID Connect scope, target service, or claim
  set exceeds what the prior SAML relying-party relationship and current
  authorization server policy permit, the authorization server MUST either
  narrow the granted scope and returned claims to the permitted set or reject the
  request with `invalid_scope`, `invalid_target`, or `unauthorized_client`, as
  appropriate.

If end-user consent is required by local policy and no valid consent or
equivalent policy exists, the authorization server MUST reject the request with
`invalid_scope` or `unauthorized_client` according to local policy.

# Client Registration {#client-registration}

## `saml_sp_entity_id` {#client-registration-saml-sp-entity-id}

This document defines the `saml_sp_entity_id` client metadata parameter.

The `saml_sp_entity_id` value is a string that identifies the SAML 2.0 SP
Entity ID to which the OAuth client is bound.

When `saml_sp_entity_id` is present:

* its value MUST exactly match the SAML SP Entity ID used in the existing SAML
  deployment;
* the authorization server MUST bind the client registration to that SAML SP
  relationship for purposes of subject continuity and claim release policy;
* the authorization server MAY allow multiple OAuth clients to share the same
  `saml_sp_entity_id` when local policy determines that those clients represent
  the same historical relying-party relationship;
* the authorization server MUST reject registration if the value is unknown,
  disallowed, or local policy does not authorize the client to use that
  binding; and
* the authorization server MUST ensure that the client authenticated at runtime
  is one of the clients authorized to use that `saml_sp_entity_id`.

An authorization server MAY support `saml_sp_entity_id` for dynamic client
registration, static client registration, or both. For statically registered
clients, the `saml_sp_entity_id` binding MAY be established through
administrative configuration rather than a client registration request.

Because this profile is defined only for confidential clients, an authorization
server MUST NOT enable this profile for a public client registration.

When multiple OAuth clients share the same `saml_sp_entity_id`, the
authorization server MUST treat them as the same relying-party context for
claim release and subject continuity under this profile.

If multiple OAuth clients sharing the same `saml_sp_entity_id` present
conflicting `subject_type` values, the authorization server MUST either apply a
single effective subject type for that shared binding or reject the
registration or request that would create inconsistent subject semantics.

Clients SHOULD register `subject_type` explicitly when using this profile.
When `subject_type` is not registered but `saml_sp_entity_id` is present, the
authorization server MUST apply `public` as the effective subject type. This
default aligns with OpenID Connect Core behavior when `subject_type` is not
registered; deployments that previously relied on SAML SP-specific (pairwise)
identifiers SHOULD register `subject_type=pairwise` explicitly to preserve that
behavior. When `subject_type` is `pairwise` and `saml_sp_entity_id` is present,
the authorization server MUST use the `saml_sp_entity_id` as the relying-party
input for pairwise subject continuity.

When `saml_sp_entity_id` is present, a registered `sector_identifier_uri`
{{OIDC-REGISTRATION}} MUST NOT alter pairwise subject calculation under this
profile. If a client registration includes both `saml_sp_entity_id` and
`sector_identifier_uri`, the authorization server MUST either ensure that the
`sector_identifier_uri` represents the same historical relying-party context
as the bound `saml_sp_entity_id` or reject the registration or request as
inconsistent.

# Authorization Server Metadata {#as-metadata}

## `saml_idp_entity_id` {#metadata-saml-idp-entity-id}

This document defines the `saml_idp_entity_id` authorization server metadata
parameter.

The `saml_idp_entity_id` value is a string that identifies the SAML 2.0 IdP
Entity ID bound to the OAuth issuer.

When `saml_idp_entity_id` is present:

* it MUST equal the SAML IdP Entity ID used by the corresponding SAML IdP;
* it MUST identify the SAML IdP that is operated by or under the administrative
  control of the same entity that operates the OAuth authorization server
  identified by this OAuth issuer;
* it MUST be the value against which incoming SAML assertion issuers are
  validated when this profile processes SAML assertions directly.

The OpenID Connect `iss` claim and OAuth authorization server `issuer` metadata
value remain OAuth issuer identifiers. They are not required to equal the SAML
IdP Entity ID, and they commonly will not, because OAuth issuer values are
HTTPS URLs with OAuth-specific discovery semantics.

## `saml_metadata_uri` {#metadata-saml-metadata-uri}

This document defines the `saml_metadata_uri` authorization server metadata
parameter.

The `saml_metadata_uri` value is an HTTPS URI that resolves to the SAML
metadata document describing the SAML IdP Entity ID bound to the OAuth issuer.

When `saml_metadata_uri` is present:

* the referenced SAML metadata MUST describe the same `saml_idp_entity_id`;
* any implementation that fetches `saml_metadata_uri` MUST verify that the
  retrieved document is a well-formed SAML metadata document containing an
  `EntityDescriptor` whose `entityID` attribute exactly matches
  `saml_idp_entity_id`; if this validation fails, the retrieved document
  MUST NOT be used;
* the authorization server SHOULD make the metadata available in a form suitable
  for standard SAML metadata processing;
* clients and migration tooling MAY use the metadata to correlate the OAuth
  issuer with the pre-existing SAML trust configuration.

## `token_exchange_requested_token_types_supported` {#metadata-token-exchange-requested-token-types-supported}

This document defines the `token_exchange_requested_token_types_supported`
authorization server metadata parameter.

The `token_exchange_requested_token_types_supported` value is a JSON array of
strings containing the token type URIs that the authorization server accepts as
`requested_token_type` values for SAML token exchange under this profile.

If the authorization server publishes metadata and supports Token Exchange
under this profile, it SHOULD include this metadata value.

When `token_exchange_requested_token_types_supported` is present:

* each value MUST be one of
  `urn:ietf:params:oauth:token-type:refresh_token`,
  `urn:ietf:params:oauth:token-type:id_token`, or
  `urn:ietf:params:oauth:token-type:access_token`;
* if `urn:ietf:params:oauth:token-type:id_token` is included, the
  authorization server MUST be an OpenID Provider; and
* if `urn:ietf:params:oauth:token-type:access_token` is included, the
  authorization server MUST support the target-selection processing defined by
  the Token Exchange access token rules for access token requests.

This profile intentionally limits `requested_token_type` to these three values.
Other token type URIs defined by {{RFC8693}}, such as
`urn:ietf:params:oauth:token-type:jwt`, are outside the scope of this profile.

## `introspection_token_types_supported` {#metadata-introspection-token-types-supported}

This document defines the `introspection_token_types_supported`
authorization server metadata parameter.

The `introspection_token_types_supported` value is a JSON array of strings.
Each string is a token type URI identifying a token format that the
authorization server accepts for direct introspection at its introspection
endpoint according to {{saml-assertion-introspection}}.

If this metadata value is omitted, no support is implied for direct
introspection of additional token types under this profile.

When `introspection_token_types_supported` includes
`urn:ietf:params:oauth:token-type:saml2`:

* the authorization server's `introspection_endpoint` metadata MUST be present;
* the introspection endpoint MUST accept the request pattern defined in
  {{saml-assertion-introspection}};
* the authorization server SHOULD accept
  `token_type_hint=urn:ietf:params:oauth:token-type:saml2`.

## Capability and Metadata Matrix {#metadata-capability-matrix}

The following table summarizes the authorization server metadata used to signal
capabilities defined by this profile.

| Capability | Required metadata | Conditional metadata | Notes |
| --- | --- | --- | --- |
| SAML issuer binding | `saml_idp_entity_id` | `saml_metadata_uri` | `saml_idp_entity_id` identifies the SAML IdP bound to the OAuth issuer. `saml_metadata_uri` is optional but, when present, MUST describe the same SAML IdP Entity ID. |
| Token Exchange with SAML input | `grant_types_supported` containing `urn:ietf:params:oauth:grant-type:token-exchange` when `grant_types_supported` is published | `token_exchange_requested_token_types_supported` | `token_exchange_requested_token_types_supported` SHOULD be published when Token Exchange is supported under this profile. |
| Token Exchange issuing ID Tokens | `token_exchange_requested_token_types_supported` containing `urn:ietf:params:oauth:token-type:id_token` | OpenID Connect Discovery metadata required by {{OIDC-DISCOVERY}} | The authorization server MUST be an OpenID Provider. |
| Token Exchange issuing access tokens | `token_exchange_requested_token_types_supported` containing `urn:ietf:params:oauth:token-type:access_token` | `resource` and `audience` support according to local policy | The authorization server MUST support target-selection processing for access token requests. |
| Direct SAML assertion introspection | `introspection_endpoint`; `introspection_token_types_supported` containing `urn:ietf:params:oauth:token-type:saml2` | None | The introspection endpoint MUST accept the request pattern defined by {{saml-assertion-introspection}}. |
| SAML Response wrapper input | No separate metadata parameter | Capability is implied only for protocol patterns where the authorization server accepts such input | Support for a signed SAML `Response` wrapper is optional. If supported, the rules in {{response-wrapper-processing}} apply. |

Omission of `token_exchange_requested_token_types_supported` or
`introspection_token_types_supported` does not imply support for the
corresponding capability. A client or migration tool MUST NOT assume support for
Token Exchange requested token types or direct SAML assertion introspection
unless the relevant metadata or an equivalent administrative configuration
indicates that support.

## SAML Key Material and ACR Metadata {#saml-key-material}

This document does not define an authorization server metadata parameter for
SAML signing keys or encryption keys. That omission is intentional.

SAML keys are published and processed through SAML metadata and `KeyDescriptor`
elements, while OAuth and OpenID Connect signing keys are published and
processed through `jwks_uri` or `jwks`. Even when the same underlying key
material is reused, it MUST be published and validated using the metadata and
encoding rules of each protocol separately.

An OpenID Provider supporting this profile and publishing `acr_values_supported`
metadata in Discovery {{OIDC-DISCOVERY}} SHOULD include any `acr` values that
it can emit under this profile, including SAML `AuthnContextClassRef` URIs that
it passes through without transformation when that set is enumerable. If the
OpenID Provider cannot publish a reliable list, it SHOULD omit
`acr_values_supported` rather than publish incomplete or misleading metadata.

# SAML Input Validation and Binding {#saml-input-validation}

This section applies whenever the authorization server directly consumes SAML
input under this profile.

## Accepted SAML Inputs {#accepted-saml-inputs}

For this profile, `urn:ietf:params:oauth:token-type:saml2` can identify either
a base64url-encoded SAML 2.0 assertion, as in {{RFC8693}}, or a signed SAML
`Response` wrapper containing exactly one bearer assertion. When a `Response`
wrapper is submitted, the authorization server extracts the enclosed assertion
and treats it as the effective assertion.

This profile is limited to signed SAML input containing exactly one unencrypted
SAML 2.0 bearer assertion. The SAML input is signed either as a signed
`Assertion` or as a signed SAML `Response` containing exactly one bearer
`Assertion`. Encrypted assertions are out of scope. SAML protocol messages other
than `Response`, and assertions that rely on subject confirmation methods other
than bearer, are also out of scope.

This profile uses an assertion-centric processing model: the effective token is
always one SAML `Assertion`. The submitted SAML input MUST therefore be either:

* a signed SAML `Assertion`; or
* a signed SAML `Response` from which the authorization server extracts exactly
  one enclosed `Assertion`.

## Response Wrapper Processing {#response-wrapper-processing}

If a signed SAML `Response` is submitted, the authorization server:

* MAY use the response signature to establish integrity and SAML Response Issuer
  authenticity for the enclosed `Assertion` when that `Assertion` is not
  independently signed;
* MUST NOT treat successful validation of a SAML `Response` signature as proof
  that response-level fields such as `Destination`, `Consent`, or
  `Response/@InResponseTo` have been validated;
* MUST NOT allow response-level fields by themselves to determine subject
  continuity, claim release, or target binding under this profile;
* MUST verify that the top-level `StatusCode` value is
  `urn:oasis:names:tc:SAML:2.0:status:Success`;
* MUST reject the response if a subordinate `StatusCode` element is present; and
* MAY ignore `StatusMessage` and `StatusDetail` except for logging or
  diagnostics.

This profile does not define full response-level protocol processing.
Deployments that require additional response-level checks MUST perform them
outside this profile before or during extraction of the effective `Assertion`.
Such additional checks SHOULD include, at minimum, verification that
`Response/@Destination` matches the intended endpoint, validation that
`Response/@InResponseTo`, if present, corresponds to a previously issued
`<AuthnRequest>` ID, and confirmation that the `Response/@ID` value has not been
replayed.

## Encrypted Content {#encrypted-content}

This profile does not define processing of encrypted subject or attribute
content inside an otherwise accepted assertion, including `EncryptedID` and
`EncryptedAttribute`. An authorization server consuming SAML input under this
profile MUST reject any effective assertion containing such elements.

## SAML and OAuth Issuer Boundary {#issuer-binding}

This profile uses distinct issuer identifiers from different protocol contexts:

* SAML Response Issuer: the value of the SAML `Response/Issuer` element, when a
  SAML `Response` wrapper is submitted;
* SAML Assertion Issuer: the value of the effective SAML `Assertion/Issuer`
  element;
* Trusted SAML IdP Entity ID: the `saml_idp_entity_id` value bound to the OAuth
  issuer identifier; and
* OAuth issuer identifier: the authorization server issuer value used in OAuth
  authorization server metadata and as the OpenID Connect ID Token `iss` claim.

The SAML Assertion Issuer MUST match the Trusted SAML IdP Entity ID. When a SAML
`Response` wrapper is submitted, the SAML Response Issuer MUST also match the
Trusted SAML IdP Entity ID.

The OAuth issuer identifier and the Trusted SAML IdP Entity ID identify related
but distinct protocol issuers. The `iss` claim in an ID Token issued under this
profile MUST be the OAuth issuer identifier, not the SAML Assertion Issuer or
SAML Response Issuer.

The authorization server MUST NOT copy a SAML issuer value into the ID Token
`iss` claim or use a SAML issuer value as a substitute for OAuth issuer
metadata.

## Signature, SAML Issuer, and Audience Binding {#signature-issuer-audience-binding}

The authorization server MUST:

1. determine the effective `Assertion` to be processed:
   * if the submitted SAML input is an `Assertion`, that `Assertion` is the
     effective assertion and it MUST be signed;
   * if the submitted SAML input is a `Response`, that `Response` MUST be
     signed, it MUST contain exactly one enclosed `Assertion`, and that
     enclosed `Assertion` becomes the effective assertion for the remaining
     rules in this section;
2. validate the signed SAML element or elements using SAML metadata and SAML
   key material, not JOSE metadata;
3. if the submitted SAML input is a `Response`, verify that:
   * the SAML `Response/Issuer` value matches the Trusted SAML IdP Entity ID
     defined in {{issuer-binding}};
   * the top-level `Response/Status/StatusCode/@Value` is
     `urn:oasis:names:tc:SAML:2.0:status:Success`; and
   * no nested `Response/Status/StatusCode/StatusCode` element is present;
4. validate the effective `Assertion` according to SAML 2.0 processing rules,
   including `Assertion/Issuer`, time validity (enforcing both
   `Conditions/@NotBefore` and `Conditions/@NotOnOrAfter`), conditions, and
   subject confirmation;
5. verify that the effective SAML `Assertion/Issuer` value matches the Trusted
   SAML IdP Entity ID defined in {{issuer-binding}};
6. verify that the effective `Assertion` contains at least one
   `AudienceRestriction` element;
7. verify that the bound `saml_sp_entity_id` appears as an `Audience` value
   in at least one `AudienceRestriction` element; additional `Audience` values
   beyond the bound `saml_sp_entity_id` are permitted, need not correspond to
   the bound `saml_sp_entity_id`, and MUST NOT by themselves invalidate the
   assertion; and
8. reject the SAML input if client authentication, assertion audience, and the
   bound `saml_sp_entity_id` do not identify the same relying-party migration
   context.

An authorization server consuming SAML input under this profile MUST reject any
encrypted assertion, any submitted SAML input that is neither an `Assertion` nor
a `Response` as defined by this profile, or any assertion using a subject
confirmation method outside this profile. Endpoint-specific error behavior is
defined by {{token-exchange-error-response}} and {{introspection-error-response}}.

## Bearer Subject Confirmation {#bearer-subject-confirmation}

The authorization server MUST identify at least one usable `SubjectConfirmation`
element with the bearer method URI (`urn:oasis:names:tc:SAML:2.0:cm:bearer`).
If multiple bearer `SubjectConfirmation` elements are present, any one valid
bearer confirmation is sufficient. For a usable bearer `SubjectConfirmationData`:

* `NotOnOrAfter`, if present, MUST be in the future at the time of processing;
* `Recipient`, if present, MUST match a recipient URI or ACS endpoint
  associated by local configuration or SAML metadata with the bound
  `saml_sp_entity_id`. This check applies to assertions submitted under both
  the Token Exchange pattern and the introspection pattern;
* `InResponseTo`, if present, MUST either have been validated by the component
  that handled the front-channel SAML exchange or be validated by equivalent
  state known to the authorization server;
* if `InResponseTo` is absent, the assertion MAY still be accepted under this
  profile. Deployments that require SP-initiated request correlation for the
  bound SAML relationship MUST reject such assertions;
* `Address`, if present, MUST either have been validated by the component that
  handled the front-channel SAML exchange or be validated by deployment-
  specific means available to the authorization server; and
* values that cannot be validated under the preceding rules MUST cause the
  assertion to be rejected as unusable under this profile.

## Replay and Freshness {#replay-freshness}

The authorization server MUST enforce any applicable SAML one-time-use,
replay-prevention, and assertion freshness requirements associated with the
trusted SAML deployment. At minimum, it MUST detect reuse of the same trusted
assertion identifier from the same SAML Assertion Issuer until the assertion's
validity window or local freshness window has expired. Whether reuse within that
window is accepted, rejected, or further constrained by the authenticated client and
bound `saml_sp_entity_id` is a deployment policy decision under this profile,
except where a stricter rule is stated below. The authorization server MUST
also enforce any applicable SAML proxying restrictions.

If the SAML assertion's `Conditions` element includes a `OneTimeUse`
condition, the authorization server MUST treat the assertion as exhausted
immediately after its first successful use under this profile and MUST reject
any subsequent submission of the same assertion, even within its validity
window.

An active introspection response or successful Token Exchange response does not,
by itself, require the assertion to be treated as exhausted under this profile
unless local replay policy or SAML `OneTimeUse` semantics require that result.

Authorization servers SHOULD permit a limited clock skew tolerance when
evaluating time-based conditions such as `Conditions/@NotBefore`,
`Conditions/@NotOnOrAfter`, and `SubjectConfirmationData/@NotOnOrAfter`.
Any clock skew allowance MUST be consistent with local security policy and
SHOULD NOT exceed five minutes, in accordance with {{SAML2-CORE}} Section
2.5.1.2.

When an `AuthnStatement` is present, the authorization server SHOULD verify
that the time elapsed since `AuthnStatement/@AuthnInstant` does not exceed a
deployment-configured freshness window. In the absence of a stricter local
policy, a maximum freshness window of 8 hours is RECOMMENDED, consistent with
typical enterprise SSO session durations. Such an assertion-age check is
independent of the `Conditions/@NotOnOrAfter` validity constraint and of
`SessionNotOnOrAfter`.

Client authentication does not replace SAML assertion validation. A valid
client cannot cause an invalid or misbound SAML assertion to become acceptable
under this profile.

## Migration Binding Rationale

This profile uses the SAML SP Entity ID as the primary migration key because
SAML pairwise subject identifiers, attribute release policy, and audience
restrictions are already typically bound to that value.

# Token Exchange Using a SAML Assertion {#token-exchange}

This section defines how a client can use OAuth 2.0 Token Exchange to obtain
OAuth 2.0 or OpenID Connect tokens from a SAML 2.0 assertion associated with an
existing SAML SP relationship.

An authorization server supporting this pattern MUST support Token Exchange as
defined by {{RFC8693}}. If the authorization server publishes metadata, its
`grant_types_supported` metadata SHOULD include
`urn:ietf:params:oauth:grant-type:token-exchange`.

An authorization server that supports
`requested_token_type=urn:ietf:params:oauth:token-type:id_token` under this
profile MUST also be an OpenID Provider and MUST comply with {{OIDC-CORE}} for
the ID Tokens that it issues.

Under this pattern, the migration client authenticates the user through the
existing SAML 2.0 deployment, receives a SAML assertion for the SAML SP
identified by `saml_sp_entity_id`, and presents that assertion, or a signed
SAML `Response` that conveys it, to the authorization server token endpoint
using Token Exchange. The issued token is determined by the
`requested_token_type` parameter and authorization server policy. This profile
defines issuance of refresh tokens, access tokens, and ID Tokens.

## Request {#token-exchange-request}

The client makes a Token Exchange request to the token endpoint using the
following parameters:

The following non-normative example requests an ID Token from a SAML assertion:

~~~ http
POST /token HTTP/1.1
Host: as.example.edu
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Atoken-exchange
&subject_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Asaml2
&subject_token=PHNhbWxwOlJlc3BvbnNlLi4u
&requested_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Aid_token
&scope=openid%20profile%20email
~~~

The request parameters are summarized below:

| Parameter | Requirement | Applies To | Summary |
|---|---:|---|---|
| `grant_type` | REQUIRED | all requests | Identifies the request as OAuth 2.0 Token Exchange. |
| `subject_token` | REQUIRED | all requests | Carries the base64url-encoded SAML `Assertion` or SAML `Response` wrapper. |
| `subject_token_type` | REQUIRED | all requests | Identifies the effective subject token as SAML 2.0. |
| `requested_token_type` | REQUIRED | all requests | Selects refresh token, ID Token, or access token issuance. |
| `scope` | Conditional | all requests | Required for refresh token and ID Token requests; optional for access token requests. |
| `resource` | OPTIONAL | access token requests | Identifies resource URI targets using {{RFC8707}}. |
| `audience` | OPTIONAL | access token requests | Identifies logical target services using {{RFC8693}}. |

`grant_type`:

REQUIRED. The value `urn:ietf:params:oauth:grant-type:token-exchange`.

`subject_token`:

REQUIRED. The base64url-encoded octet sequence, as defined in Section 5 of
{{RFC4648}}, of the XML serialization of either:

* a signed SAML 2.0 `<Assertion>` element obtained for the existing SAML SP
  relationship; or
* a signed SAML 2.0 `<Response>` containing exactly one bearer `<Assertion>`
  for that relationship.

The encoded value MUST NOT be line wrapped and MUST NOT include padding
characters (`=`). This encoding convention follows the operational form used by
{{RFC7522}}. When a SAML `<Response>` is supplied, it is used only as a signed
wrapper from which this profile extracts the effective assertion, according to
{{saml-input-validation}}.

`subject_token_type`:

REQUIRED. The value `urn:ietf:params:oauth:token-type:saml2`.
When a SAML `<Response>` wrapper is supplied, this token type identifies the
effective enclosed SAML 2.0 assertion rather than the wrapper element itself.

`requested_token_type`:

REQUIRED. This profile permits only the values shown in the following table.
{{RFC8693}} marks `requested_token_type` as OPTIONAL at the Token Exchange
protocol level. This profile narrows that rule: an omitted
`requested_token_type` MUST be rejected with `invalid_request`.

| requested_token_type value | Requested token | scope requirement | Target-selection requirement |
|---|---|---|---|
| `urn:ietf:params:oauth:token-type:refresh_token` | Refresh token | `scope` MUST be present and MUST include `offline_access`. An absent `scope` parameter MUST be treated as a missing `offline_access` value and rejected with `invalid_request`. | This profile defines no `resource` or `audience` semantics. Clients SHOULD NOT send them. |
| `urn:ietf:params:oauth:token-type:id_token` | ID Token | `scope` MUST be present and MUST include `openid`. An absent `scope` parameter MUST be rejected with `invalid_request`. | This profile defines no `resource` or `audience` semantics. Clients SHOULD NOT send them. |
| `urn:ietf:params:oauth:token-type:access_token` | Access token | `scope` is OPTIONAL. The request MAY omit `scope` entirely or contain only OAuth scope values without `openid`. | The client MAY send `resource`, `audience`, or both. When both are present, they MUST identify a consistent target service set as understood by the authorization server. |

Target-selection processing for access token requests is defined in
{{token-exchange-common-processing}}.

`scope`:

Conditional. The `scope` requirement depends on `requested_token_type`, as
defined above. This profile permits ordinary OAuth scope values in addition to
OpenID Connect scopes.

This profile does not define a token-endpoint parameter for fine-grained claim
selection. Claims included in a directly issued ID Token, and claims later made
available through UserInfo, are determined by the granted scope, the rules in
{{claim-mapping}}, and the claim release policy for the relying-party relationship
represented by the bound `saml_sp_entity_id`.

`resource`:

OPTIONAL. For access token requests, one or more resource indicators as defined
by {{RFC8707}}. Clients requesting an access token under this profile SHOULD
send a single `resource` value when a resource URI is known.

`audience`:

OPTIONAL. For access token requests, one or more logical target service
identifiers as defined by {{RFC8693}}.

The target-selection rules for `resource` and `audience` depend on
`requested_token_type`, as defined above.

This profile does not define the use of `actor_token`, `actor_token_type`, or
`authorization_details` in this Token Exchange request. Deployments MAY support
them by separate agreement, but they are outside this profile.

Only confidential clients can use this profile. Client authentication at the
token endpoint uses the client's registered OAuth 2.0 client authentication
method. The authenticated client MUST be a confidential client authorized for
the `saml_sp_entity_id` that matches the SAML assertion audience.

## Authorization Server Processing {#token-exchange-as-processing}

### Common Processing Rules {#token-exchange-common-processing}

Upon receiving the request, the authorization server MUST:

1. authenticate the client in accordance with its registered client
   authentication method;
2. validate the SAML input as described in {{saml-input-validation}};
3. verify that the authenticated client is authorized for the
   `saml_sp_entity_id` that matches the SAML assertion audience;
4. verify that the requested token type is defined by this profile and is
   permitted for the authenticated client;
5. resolve the SAML assertion to exactly one active Local Account as described
   in {{local-subject-resolution}};
6. resolve the requested target service set for access token requests using the
   `resource` and `audience` parameters, or the implicit UserInfo target when
   `openid` is requested, or a preconfigured default target service set for the
   authenticated client when neither `resource` nor `audience` is provided and
   local policy permits;
7. evaluate the requested scope according to the policy that applies to the
   same relying-party relationship represented by the existing SAML SP and the
   resolved target service set, if any;
8. derive the end-user subject according to {{subject-identifier-mapping}} and
   any claims needed for the issued token according to {{claim-mapping}}; and
9. issue the requested token type, bound to the authenticated client and the
   end-user where applicable, if the request is approved.

Steps 1 through 3 are mutually dependent: client authentication determines which
`saml_sp_entity_id` bindings are applicable, and assertion audience validation
requires knowing those bindings. Implementations SHOULD evaluate these steps as
an integrated unit.

The authorization server MUST reject the request if the SAML assertion was
issued for a different SAML SP than the one bound to the authenticated client.

The authorization server MUST reject the request if subject resolution yields no
Local Account, more than one candidate Local Account, or a Local Account that
is disabled, suspended, deprovisioned, or otherwise not permitted to
authenticate.

The authorization server MUST enforce the token-type-specific scope and target
selection requirements defined in {{token-exchange-request}}. If the
authorization server cannot determine an acceptable target service set for an
access token request, or if the request asks for multiple target services that
policy does not allow to share a single token, it MUST reject the request with
`invalid_target`.

When `requested_token_type` is
`urn:ietf:params:oauth:token-type:refresh_token` or
`urn:ietf:params:oauth:token-type:id_token`, the authorization server SHOULD
ignore `resource` and `audience` if present.

When `requested_token_type` is
`urn:ietf:params:oauth:token-type:access_token` and the requested scope includes
`openid`, the authorization server:

* MUST be an OpenID Provider;
* MUST publish a `userinfo_endpoint` in accordance with OpenID Connect Discovery
  {{OIDC-DISCOVERY}};
* MUST resolve the UserInfo endpoint as part of the target service set when both
  `resource` and `audience` are omitted;
* MAY include the UserInfo endpoint in the target service set when explicit
  `resource` and/or `audience` values are present and policy allows a single
  token to cover both the UserInfo endpoint and those targets; and
* MUST either omit `openid` from the granted scope or reject the request with
  `invalid_target` or `invalid_scope` if the resolved target service set does
  not include the UserInfo endpoint.

When `requested_token_type` is
`urn:ietf:params:oauth:token-type:access_token`, the requested scope does not
include `openid`, and both `resource` and `audience` are omitted, the
authorization server MAY use a preconfigured default target service set for the
authenticated client and bound `saml_sp_entity_id`, or it MAY reject the request
with `invalid_target`.

For any request that includes `scope`, the authorization server MUST apply the
authorization and consent rules in {{authorization-consent-model}}. If no
applicable grant policy exists for a requested scope, the authorization server
MUST reject the request with `invalid_scope`.

### Successful Response {#token-exchange-successful-response}

For any successful request, the authorization server MUST return a Token Exchange
response as defined by {{RFC8693}}. The response:

* MUST include `issued_token_type` set to the issued token type and equal to one
  of `urn:ietf:params:oauth:token-type:refresh_token`,
  `urn:ietf:params:oauth:token-type:id_token`, or
  `urn:ietf:params:oauth:token-type:access_token`;
* MUST include `access_token` containing the issued token;
* MUST include `token_type` set to `N_A` when the issued token type is
  `urn:ietf:params:oauth:token-type:refresh_token` or
  `urn:ietf:params:oauth:token-type:id_token`, and otherwise set to the
  applicable access token type such as `Bearer`;
* MAY include `expires_in` as defined by {{RFC8693}}; and
* MUST include `scope` when the granted scope differs from the requested scope,
  as required by {{RFC6749}} and {{RFC8693}}.

{{RFC8693}} uses the `access_token` response member to carry all issued token
types. Clients MUST interpret the `access_token` value according to
`issued_token_type`, as specified in {{RFC8693}} Section 2.2.

### Refresh Token {#token-exchange-refresh-token}

When issuing a refresh token, the authorization server MUST ensure that the
issued refresh token preserves the subject continuity established by the SAML
assertion. In particular, later use of the refresh token for the same client
MUST result in the same `sub` value that would have been derived directly from
the SAML assertion under {{local-subject-resolution}},
{{subject-identifier-mapping}}, and {{claim-mapping}}.

If Token Exchange under this profile issues a refresh token, the client can use
the standard OAuth 2.0 `refresh_token` grant at the same authorization server.

When the authorization server issues a replacement refresh token as part of
refresh token rotation, the replacement token MUST be bound to the same Local
Account as the original refresh token and MUST produce the same `sub` value
under {{subject-identifier-mapping}} when used by the same migrated client.

When the granted scope includes `openid`, the authorization server SHOULD
return an ID Token in the first successful refresh token response for that
refresh token. Returning such an ID Token gives the client the migrated subject
and authentication claims in a signed, verifiable format. Any returned ID Token:

* MUST use the `iss` value of the authorization server;
* MUST use the `sub` value defined by {{subject-identifier-mapping}};
* MUST apply the claim mapping rules in {{claim-mapping}}; and
* MUST preserve the original authentication time from the SAML
  `AuthnStatement`, if known, in the `auth_time` claim.

Subsequent refresh token responses are governed by {{OIDC-CORE}}. If an
ID Token is returned in a later refresh response, it MUST continue to preserve
the same `sub` and authentication continuity unless a later authentication
event changes those values under {{OIDC-CORE}}.

The refresh token issued under this profile MAY also be used as an input to
other specifications that accept
`urn:ietf:params:oauth:token-type:refresh_token` as a Token Exchange
`subject_token_type`, but those subsequent exchanges are defined by the
respective specifications, not by this document.

Subsequent refresh token requests for access tokens MAY use `resource` and
`audience` according to {{RFC8707}}, {{RFC8693}}, and authorization server policy.
This profile does not modify their meaning in refresh token requests. Scope
reduction in subsequent refresh token requests is governed by {{RFC6749}} and is
not modified by this profile.

When a refresh token issued under this profile expires or is revoked, the client
MAY obtain a new SAML assertion through the SAML 2.0 deployment and perform a
new Token Exchange request. The client MAY also use any other OAuth 2.0 or
OpenID Connect flow that the authorization server supports.

If the authorization server learns that the resolved Local Account has been
disabled, that the authoritative SAML-authenticated session has terminated, or
that administrative policy has revoked the migrated session, it MUST reject
further use of any derived refresh token for session-preserving operations. It
SHOULD also revoke or expire related access tokens and other session-bound
artifacts according to local capabilities. This document does not define a
logout or revocation signaling protocol between the SAML IdP and the
authorization server. Deployments that support back-channel session
termination signaling MAY use the mechanisms defined by {{OIDC-BC-LOGOUT}} to
propagate SAML session termination events to registered OpenID Connect clients.

### ID Token {#token-exchange-id-token}

When issuing an ID Token directly from Token Exchange, that ID Token:

* MUST use the `iss` value of the authorization server;
* MUST contain an `aud` claim whose value is exactly the authenticated
  client's client identifier, as required by {{OIDC-CORE}} Section 2. Any
  `audience` parameter present in the Token Exchange request MUST be ignored
  for purposes of ID Token audience construction;
* MUST use the `sub` value defined by {{subject-identifier-mapping}};
* MUST apply the claim mapping rules in {{claim-mapping}};
* MUST preserve the original authentication time from the SAML
  `AuthnStatement`, if known, in the `auth_time` claim;
* MUST set `iat` to the time the Token Exchange response is issued, not to
  the SAML `AuthnInstant`;
* MUST set `exp` to a time appropriate for a directly issued ID Token under
  local policy. When the ID Token represents the same SAML-authenticated
  session and `SessionNotOnOrAfter` is present, `exp` MUST NOT be later than
  `SessionNotOnOrAfter`. This cap is a session bound, not a direct translation
  of the SAML assertion lifetime. `exp` MUST NOT be taken directly from the
  SAML assertion's `Conditions/@NotOnOrAfter`;
* MUST NOT include `nonce`, `at_hash`, or `c_hash`, because no preceding
  OpenID Connect authentication request or authorization code exists from
  which these values could be derived; and
* SHOULD NOT include `azp`. If `azp` is included, it MUST equal the
  authenticated client identifier.

### Access Token {#token-exchange-access-token}

This document does not define the syntax of an access token issued by this
exchange. However, when issuing an access token, the authorization server:

* MUST bind the token to the authenticated client, the granted scope, and the
  resolved target service set;
* MUST bind any end-user authorization carried by the token to the resolved
  Local Account and to a subject representation consistent with
  {{subject-identifier-mapping}};
* MUST audience-restrict the token to the resolved target service set; and
* MAY represent that audience restriction in the token value itself or
  associated token metadata.

If the granted scope includes `openid`, the authorization server MUST issue an
access token that is valid at the OpenID Provider's UserInfo endpoint. Under
this profile, a granted `openid` scope makes the UserInfo endpoint part of the
token's resolved target service set. If the original request included
`openid` but the issued access token is not valid at the UserInfo endpoint,
the granted scope in the Token Exchange response MUST omit `openid` or the
request MUST have been rejected.

If an access token issued under this profile is presented to the UserInfo
endpoint, the UserInfo response MUST conform to
{{userinfo-oidc-output-consistency}}. If the access token was issued without `openid`, the OpenID
Provider MUST reject its use at the UserInfo endpoint according to
{{OIDC-CORE}}.

Any representation of the end-user subject produced from an access token issued
under this profile by the same authorization server for the same client MUST be
consistent with {{subject-identifier-mapping}}.

This document does not define additional introspection semantics for access
tokens issued under this profile. If the authorization server supports access
token introspection, that behavior remains governed by {{RFC7662}}, the
authorization server's access token profile, and local policy.

## Error Response {#token-exchange-error-response}

If the request is malformed, the authorization server MUST return an error
response as defined by {{RFC6749}} and {{RFC8693}}.

The authorization server MUST use `invalid_request` for malformed or internally
inconsistent Token Exchange parameters, including SAML input that is invalid,
expired, audience-mismatched, encrypted, supplied in an unsupported wrapper
form, wrapped in a SAML `Response` whose `Status` is not acceptable under this
profile, or otherwise unusable under this profile.

The authorization server SHOULD use:

* `unauthorized_client` when the authenticated client is not permitted to use
  the bound `saml_sp_entity_id` relationship or the requested token type;
* `invalid_target` when the `resource` and `audience` parameters are missing,
  malformed, inconsistent, unknown, or unacceptable for the requested access
  token; and
* `invalid_scope` when the requested scope cannot be granted.

The authorization server SHOULD also use `invalid_request` when the assertion
cannot be resolved to exactly one active Local Account or when the assertion is
rejected as replay or otherwise exhausted under {{saml-input-validation}}.

The authorization server SHOULD also use `invalid_request` when a currently
asserted stable subject identifier conflicts with an existing persisted mapping
and no explicitly authorized administrative remapping exists.

# SAML Assertion Introspection {#saml-assertion-introspection}

This section defines an introspection extension for cases in which the client
does not need OAuth tokens and only needs a SAML assertion to be validated and
translated into normalized JSON claims.

This pattern uses the authorization server's introspection endpoint as defined
by {{RFC7662}}. Under this pattern, a migration client authenticates the user
through the existing SAML 2.0 deployment, receives a SAML assertion for the
SAML SP identified by `saml_sp_entity_id`, and submits that assertion, or a
signed SAML `Response` that conveys it, to the introspection endpoint. If the
assertion is valid for the client and the client is authorized to receive the
mapped subject and claims, the authorization server returns an active
introspection response containing a JSON claims object. No OAuth access token,
refresh token, or ID Token is issued.

## Request {#introspection-request}

The client makes an introspection request to the introspection endpoint using
the following parameters:

The following non-normative example asks the authorization server to validate a
SAML assertion and return normalized JSON claims and SAML protocol metadata:

~~~ http
POST /introspect HTTP/1.1
Host: as.example.edu
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded

token=PHNhbWxwOlJlc3BvbnNlLi4u
&token_type_hint=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Asaml2
~~~

The request parameters are summarized below:

| Parameter | Requirement | Summary |
|---|---:|---|
| `token` | REQUIRED | Carries the base64url-encoded SAML `Assertion` or SAML `Response` wrapper. |
| `token_type_hint` | OPTIONAL | Identifies the effective token as a SAML 2.0 assertion. |

`token`:

REQUIRED. The base64url-encoded octet sequence, as defined in Section 5 of
{{RFC4648}}, of the XML serialization of either:

* a signed SAML 2.0 `<Assertion>` element obtained for the existing SAML SP
  relationship; or
* a signed SAML 2.0 `<Response>` containing exactly one bearer `<Assertion>`
  for that relationship.

The encoded value MUST NOT be line wrapped and MUST NOT include padding
characters (`=`). This encoding convention follows the operational form used by
{{RFC7522}}. When a SAML `<Response>` is supplied, it is used only as a signed
wrapper from which this profile extracts the effective assertion, according to
{{saml-input-validation}}.

`token_type_hint`:

OPTIONAL. If sent, the value MUST be
`urn:ietf:params:oauth:token-type:saml2`.
When a SAML `<Response>` wrapper is supplied, this token type hint still
identifies the effective enclosed SAML 2.0 assertion rather than the wrapper
element itself.

This profile does not define a request parameter for selecting a subset of
normalized claims in the introspection response.

Only confidential clients can use this profile. Client authentication for the
introspection endpoint is performed according to {{RFC7662}} and authorization
server policy. The authenticated client MUST be authorized for the
`saml_sp_entity_id` that matches the SAML assertion audience.

## Authorization Server Processing {#introspection-as-processing}

### Common Processing Rules {#introspection-common-processing}

Upon receiving the request, the authorization server MUST:

1. authenticate the client and authorize it to use the introspection endpoint;
2. validate the SAML input as described in {{saml-input-validation}};
3. verify that the authenticated client is authorized for the
   `saml_sp_entity_id` that matches the SAML assertion audience;
4. resolve the SAML assertion to exactly one active Local Account as described
   in {{local-subject-resolution}};
5. derive the end-user subject according to {{subject-identifier-mapping}} and
   any claims needed for the response according to {{claim-mapping}}; and
6. apply the claim release policy for that same relying-party relationship
   before returning any claims.

### Inactive Response {#introspection-inactive-response}

If the SAML assertion is invalid, expired, audience-mismatched, otherwise not
usable for the client, wrapped in a SAML `Response` whose `Status` is not
acceptable under this profile, cannot be resolved to exactly one active Local
Account, or the authenticated client is not entitled to introspect that specific
assertion or bound relying-party context, the authorization server MUST return a
successful introspection response with `"active": false` and SHOULD NOT include
additional members.

### Successful Response {#introspection-successful-response}

For an active SAML assertion, the authorization server MUST return an
introspection response as defined by {{RFC7662}} with:

* `active` set to `true`;
* `claims` set to a JSON object containing the normalized claims derived from
  the SAML assertion for this client; and
* `saml`, when returned, set to a JSON object containing normalized SAML
  protocol metadata as defined below.

The `claims` member is intentionally nested rather than returned at the top
level of the introspection response. This separates the token-validity metadata
(`active`) from end-user identity claims derived from the SAML assertion, and
avoids ambiguity with existing or future top-level {{RFC7662}} response members.

The `claims` object:

* MUST include the mapped `sub` value defined by {{subject-identifier-mapping}};
* MAY include `auth_time`, `acr`, `amr`, `sid`, and attribute-derived claims
  defined by {{claim-mapping}}; and
* MUST NOT include claims that would not be releasable to the same relying
  party under this profile.

The authorization server SHOULD return only the minimum normalized claims
necessary for the authorized client and applicable relying-party policy.

The `saml` object contains protocol metadata, not end-user identity claims. The
authorization server MUST NOT return those values inside the `claims` object.

When the `saml` object is returned:

* every member value MUST be derived from SAML input that was successfully
  validated under {{saml-input-validation}};
* values from a SAML `Response` wrapper MUST be included only when a signed
  SAML `Response` wrapper was submitted and accepted under
  {{response-wrapper-processing}};
* response-level values that this profile does not validate, such as
  `Destination` or `Response/@InResponseTo`, MAY be returned as extracted
  values, but their presence in the response MUST NOT be interpreted as a
  statement that the authorization server validated them unless a separate
  deployment agreement says so; and
* the authorization server MUST omit any member for which it has no validated or
  extracted value.

The `saml` object is intended for clients that need the authorization server to
perform XML parsing and XML signature validation, but still need protocol values
from the validated SAML input to complete local SAML SP processing.

The `saml` object MAY contain these members:

`input_type`:
: String. Either `assertion` or `response`, indicating whether the submitted
  SAML input was a bare Assertion or a Response wrapper.

`response`:
: JSON object containing values extracted from the accepted SAML `Response`
  wrapper. This member is omitted when the submitted SAML input was a bare
  Assertion.

`assertion`:
: JSON object containing values from the effective SAML `Assertion`.

When present, the `response` object MAY contain these members:

`id`:
: String. The extracted value of `Response/@ID`.

`issuer`:
: String. The extracted value of the `Response/Issuer` element.

`issue_instant`:
: String. The extracted value of `Response/@IssueInstant`.

`destination`:
: String. The extracted value of `Response/@Destination`.

`in_response_to`:
: String. The extracted value of `Response/@InResponseTo`.

`status_code`:
: String. The validated value of the top-level
  `Response/Status/StatusCode/@Value`.

`has_nested_status_code`:
: Boolean. `true` if a nested `Response/Status/StatusCode/StatusCode` element
  was present in the submitted Response, and `false` otherwise. Under this
  profile, an active response that includes a `response` object will normally
  have this value set to `false` because {{response-wrapper-processing}}
  requires the authorization server to reject a Response wrapper that contains a
  nested status code.

When present, the `assertion` object MAY contain these members:

`id`:
: String. The value of `Assertion/@ID`.

`issuer`:
: String. The value of the `Assertion/Issuer` element.

`issue_instant`:
: String. The value of `Assertion/@IssueInstant`.

`audiences`:
: Array of strings. The `Audience` values from all `AudienceRestriction`
  conditions in the effective assertion.

`not_before`:
: String. The value of `Assertion/Conditions/@NotBefore`.

`not_on_or_after`:
: String. The value of `Assertion/Conditions/@NotOnOrAfter`.

`subject_confirmation_method`:
: String. The `Method` value of the bearer `SubjectConfirmation` that the
  authorization server treated as usable under {{bearer-subject-confirmation}}.

`subject_confirmation_recipient`:
: String. The `Recipient` value from the `SubjectConfirmationData` associated
  with the usable bearer `SubjectConfirmation`.

`subject_confirmation_in_response_to`:
: String. The `InResponseTo` value from the `SubjectConfirmationData`
  associated with the usable bearer `SubjectConfirmation`.

`subject_confirmation_not_on_or_after`:
: String. The `NotOnOrAfter` value from the `SubjectConfirmationData`
  associated with the usable bearer `SubjectConfirmation`.

Date and time values in the `saml` object SHOULD be represented as strings
preserving the SAML dateTime semantics. Implementations MAY normalize
equivalent dateTime values to a consistent UTC representation. Clients MUST
compare these values according to SAML dateTime semantics, not by lexical string
comparison unless the representation has first been normalized.

This profile does not require the introspection response to include OAuth token
metadata fields such as `scope`, `token_type`, `exp`, or `iss`, because no
OAuth token is being issued by the introspection operation.

## Error Response {#introspection-error-response}

If the introspection request is malformed — for example, the `token` parameter
is absent or its value cannot be decoded as valid base64url — the authorization
server MUST return an HTTP 400 (Bad Request) error response as defined by
{{RFC7662}}, not an introspection response.

If client authentication fails, or if the client is not authorized to use the
introspection endpoint at all, the authorization server MUST return an HTTP 401
(Unauthorized) or HTTP 403 (Forbidden) response, consistent with {{RFC7662}}
Section 2.2 and authorization server policy.

A syntactically valid request in which the client is authenticated and
authorized, but the presented SAML assertion is invalid, expired, audience-
mismatched, replayed, or otherwise not usable for the client, MUST result in
an introspection response with `"active": false`. The authorization server
SHOULD NOT include additional members in such a response.

# Local Subject Resolution {#local-subject-resolution}

Before deriving `sub` or other end-user claims under this profile, the
authorization server MUST resolve the SAML assertion to exactly one active
Local Account.

This document does not require or define just-in-time provisioning. If a
deployment performs provisioning or account activation based on a validated SAML
assertion, that behavior is outside this profile and MUST complete before the
authorization server applies the subject continuity and claim mapping rules
defined here.

The authorization server MUST perform this resolution using stable SAML subject
identifiers, existing persisted linkages, or equivalent administrative mapping
for the trusted SAML deployment. Releasable attributes such as email address,
display name, telephone number, or generic usernames MUST NOT by themselves be
treated as sufficient evidence to create or select a Local Account under this
profile, although they MAY be used as supplemental hints together with a stable
binding already established by policy.

If the assertion contains a `NameID` whose format is
`urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress`, or whose semantics are
those of an email address under local policy, the authorization server MUST
treat that `NameID` as a mutable account attribute for purposes of this
profile. Such a `NameID` MUST NOT by itself establish a new Local Account
linkage or a new subject continuity mapping. It MAY be used to locate an
existing Local Account only when local policy has already bound that asserted
email identifier for the trusted SAML issuer to exactly one Local Account.

If resolution yields no Local Account, more than one candidate Local Account,
or a Local Account that is disabled, suspended, deprovisioned, or otherwise not
eligible to authenticate, the authorization server MUST fail the profile
operation and MUST NOT issue tokens or return an active introspection response.

Once a continuity mapping has been established for a resolved Local Account,
the authorization server MUST continue to resolve the same subject input to
that same Local Account unless an authorized administrative remapping occurs.

When this document requires a Stable Local Subject Key, the value MUST be taken
from the resolved Local Account and MUST be non-reassignable within the
authorization server.

# Subject Identifier Mapping {#subject-identifier-mapping}

The OpenID Connect `sub` claim is the primary continuity point for migrated
relying parties. The authorization server MUST issue a `sub` value that is
stable for the client and that preserves the semantics of the corresponding
SAML deployment.

## NameID Format Recognition {#nameid-format-recognition}

The following `NameID` formats are recognized for subject mapping purposes
under this profile:

* `urn:oasis:names:tc:SAML:2.0:nameid-format:persistent`: A persistent,
  SP-specific or globally non-reassignable identifier. MAY be used as input
  to the subject mapping rules in this section.
* `urn:oasis:names:tc:SAML:2.0:nameid-format:transient`: A session-scoped,
  non-persistent identifier. MUST NOT be used as input to subject mapping
  under this profile and MUST NOT be used as the OpenID Connect `sub`.
* `urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress`: A mutable account
  attribute. Treated as defined in {{local-subject-resolution}}. MUST NOT be used
  as `sub`.
* `urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified`: Deployment-defined
  semantics. MUST NOT be used as a subject mapping input unless local policy
  has explicitly established the identifier as stable and non-reassignable for
  the trusted SAML issuer.
* `urn:oasis:names:tc:SAML:2.0:nameid-format:entity`: Identifies a system
  entity, not an end-user. MUST NOT be used as a subject mapping input.

Any other `NameID` format MUST NOT be used as a subject mapping input under
this profile unless local policy explicitly establishes its semantics as
persistent and non-reassignable.

## Subject Type Determination {#subject-type-determination}

The authorization server MUST determine whether the migrated client is using a
public subject or a pairwise subject. The registered `subject_type`, if present,
MUST be used for this determination. If no `subject_type` is registered, the
authorization server MUST apply `public` as the effective subject type, in
accordance with {{client-registration-saml-sp-entity-id}}.

If multiple clients sharing the same `saml_sp_entity_id` have registered
conflicting `subject_type` values and no single effective subject type can be
determined per {{client-registration-saml-sp-entity-id}}, the authorization server MUST fail the profile
operation.

When the authorization server uses, or would otherwise use, the
`urn:oasis:names:tc:SAML:attribute:subject-id` or
`urn:oasis:names:tc:SAML:attribute:pairwise-id` attribute as a subject mapping
input, it MUST validate that attribute according to the SAML V2.0 Subject
Identifier Attributes Profile. In particular, the attribute MUST use
`NameFormat` `urn:oasis:names:tc:SAML:2.0:attrname-format:uri` and contain
exactly one non-empty `AttributeValue` that conforms to that profile's syntax
and semantics. If no persisted mapping exists and the attribute corresponding
to the client's effective subject type is present but fails these requirements,
the authorization server MUST reject the profile operation rather than derive a
different `sub` from fallback rules.

If a persisted mapping already exists and the current assertion also contains a
valid subject identifier input matching the client's effective subject type, or
a persistent `NameID` that would otherwise be eligible as a subject mapping
input, the authorization server MUST compare the current asserted identifier
context to the persisted source identifier context. If they differ, the
authorization server MUST NOT silently replace the persisted mapping based only
on the current assertion. The authorization server MAY continue only when an
explicitly authorized administrative remapping has bound the new identifier
context to the same Local Account; otherwise it MUST fail the profile
operation.

When multiple clients share the same `saml_sp_entity_id`, the authorization
server MUST apply the same effective subject mapping rules to all such clients.
In particular, it MUST produce the same `sub` value for the same resolved Local
Account when those clients are operating under the same subject type semantics.

## Pairwise Subject Identifier Mapping {#pairwise-subject-identifier-mapping}

For clients using pairwise subject identifiers, the authorization server MUST
determine `sub` using the first applicable rule:

1. If the authorization server already has a persisted pairwise mapping for
   the same resolved Local Account and the same `saml_sp_entity_id`, it MUST
   reuse that mapping, subject to the mismatch handling rules above.
2. If no persisted mapping exists and the SAML assertion contains a valid
   `urn:oasis:names:tc:SAML:attribute:pairwise-id` attribute (as validated by
   the rules in {{subject-type-determination}}), the authorization server MUST
   use that attribute value as `sub` and MUST persist the resulting mapping.
3. If no persisted mapping exists and the assertion contains a persistent
   `NameID` (`urn:oasis:names:tc:SAML:2.0:nameid-format:persistent`) whose
   semantics are SP-specific, stable, and non-reassignable for that same SAML
   SP relationship, the authorization server MAY use that value as `sub`, and
   it MUST persist the resulting mapping.
4. Otherwise, the authorization server MUST derive a deterministic,
   non-reassignable pairwise identifier by combining the Stable Local Subject
   Key and the bound `saml_sp_entity_id` using a collision-resistant method.
   The pairwise subject computation defined in Section 8 of {{OIDC-CORE}} is a
   suitable derivation method. The authorization server MUST persist the
   resulting mapping.

The `pairwise-id` attribute value has the format `localpart@scope` as
defined by {{SAML2-SUBJ-ID}}. When a
`pairwise-id` value is used as `sub`, the resulting value carries this scoped
format. Clients MUST NOT treat a `sub` value derived from `pairwise-id` as an
email address or interpret the scope portion as an email domain.

## Public Subject Identifier Mapping {#public-subject-identifier-mapping}

For clients using public subject identifiers, the authorization server MUST
determine `sub` using the first applicable rule:

1. If the authorization server already has a persisted public subject mapping
   for the same resolved Local Account under the same OAuth issuer, it MUST
   reuse that mapping, subject to the mismatch handling rules above.
2. If no persisted mapping exists and the SAML assertion contains a valid
   `urn:oasis:names:tc:SAML:attribute:subject-id` attribute (as validated by
   the rules in {{subject-type-determination}}), the authorization server MUST
   use that attribute value as `sub` and MUST persist the resulting mapping.
3. If no persisted mapping exists and the assertion contains a persistent
   `NameID` (`urn:oasis:names:tc:SAML:2.0:nameid-format:persistent`) whose
   semantics are stable, non-reassignable, and not SP-specific, the
   authorization server MAY use that value as `sub`, and it MUST persist the
   resulting mapping.
4. Otherwise, the authorization server MUST derive a deterministic,
   non-reassignable public identifier from the Stable Local Subject Key alone,
   without client-specific input, using a collision-resistant method. The
   authorization server MUST persist the resulting mapping.

## Common Rules {#subject-identifier-common-rules}

For all subject types:

* before issuing any chosen or derived value as `sub`, the authorization
  server MUST ensure that it satisfies the {{OIDC-CORE}} `sub`
  requirements, including that it be a case-sensitive string no longer than
  255 ASCII characters;
* if the chosen source identifier would exceed that limit or contain non-ASCII
  characters, the authorization server MUST derive a deterministic,
  collision-resistant ASCII `sub` value from the chosen source identifier and
  required context, and it MUST persist the resulting mapping;
* if both `subject-id` and `pairwise-id` are available, the authorization
  server MUST select the one whose semantics match the client's effective
  subject type;
* the authorization server MUST NOT expose a `pairwise-id` value to a client
  other than a client bound to the corresponding `saml_sp_entity_id`;
* the authorization server MUST NOT use a transient identifier as the OpenID
  Connect `sub`;
* the authorization server MUST NOT directly use mutable identifiers such as an
  email address, a `NameID` in the SAML `emailAddress` format, or a
  non-persistent `NameID` format as the OpenID Connect `sub`.

When the only usable SAML subject input is a `NameID` whose semantics are those
of an email address, and {{local-subject-resolution}} succeeds under the rules
above, the authorization server MUST derive `sub` from the resolved Stable
Local Subject Key using the applicable fallback rule in this section rather
than using the asserted email value itself.

When a persistent `NameID` is used as an input to subject mapping, the
authorization server MUST evaluate the identifier together with its qualifier
context. In particular:

* `SPNameQualifier`, if present for pairwise processing, MUST correspond to the
  bound `saml_sp_entity_id` or to an equivalent configured identifier for that
  same relying-party relationship;
* `NameQualifier`, if present, MUST correspond to the trusted SAML IdP
  identity or to an equivalent configured identifier for that same issuer;
* `SPProvidedID`, if present, MUST be treated as part of the source identifier
  context rather than ignored; and
* two `NameID` values that differ in value or qualifier context MUST be treated
  as distinct unless local policy can prove them equivalent.

# Claim Mapping {#claim-mapping}

The mappings in this section apply to ID Tokens and, when applicable, UserInfo
responses. Values derived from SAML subject identifiers or authentication
statements take precedence over generic attribute mapping.

## Authentication Event Claims {#authentication-event-claims}

When a SAML assertion contains an `AuthnStatement`, the authorization server
SHOULD map its contents as follows:

* `AuthnStatement/@AuthnInstant`, when mapped, MUST be emitted as the
  OpenID Connect `auth_time` claim as a NumericDate value.
* `AuthnContext/AuthnContextClassRef` maps to the OpenID Connect `acr` claim.
  The value SHOULD be copied without transformation. Clients operating under
  this migration profile MUST be prepared to receive SAML authentication
  context class URIs as `acr` values (for example,
  `urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport`). These
  are not registered OpenID Connect ACR values. If local policy maps SAML
  context class references to a registered Level of Assurance scheme, that
  mapping takes precedence over direct passthrough.
* `AuthnStatement/@SessionIndex` MAY be used as authoritative input to the
  OpenID Connect `sid` claim, as defined by {{OIDC-FC-LOGOUT}} and
  {{OIDC-BC-LOGOUT}}, when the deployment supports session continuity or logout
  semantics.

  When `sid` is emitted, it MUST identify the OpenID Provider session
  corresponding to the SAML-authenticated session and MUST be unique within
  the OAuth issuer. The authorization server MUST NOT assume that the raw SAML
  `SessionIndex` is already a valid `sid`; it MUST either reuse the raw value
  only when local policy guarantees it is issuer-scoped and collision-resistant
  for this purpose, or derive and persist an OpenID Provider-scoped `sid`
  bound to that SAML-authenticated session. If `SessionIndex` is absent from
  the selected `AuthnStatement`, `sid` MUST NOT be emitted unless the
  authorization server has another authoritative session identifier for that
  SAML-authenticated session.

The OpenID Connect `amr` claim identifies authentication methods, not an
authentication context class. Therefore:

* the authorization server MUST NOT copy the SAML
  `AuthnContextClassRef` directly into `amr`;
* the authorization server MAY populate `amr` only when it can derive concrete
  authentication methods from local policy or explicit SAML authentication
  evidence; and
* when `amr` is populated, the authorization server SHOULD use values from the
  Authentication Method Reference Values registry {{RFC8176}}, and it MUST represent
  `amr` as a JSON array of strings.

If multiple `AuthnStatement` elements are present, the authorization server
MUST select the statement corresponding to the authentication event that is
authoritative for the OAuth or OpenID Connect transaction. When no other basis
for selection exists, the authorization server SHOULD prefer the statement with
the most recent `AuthnInstant` value. If the authorization server cannot
determine the authoritative statement unambiguously, it SHOULD omit ambiguous
derived claims rather than misstate them.

If the selected `AuthnStatement` contains `SessionNotOnOrAfter`, the
authorization server MUST treat that instant as an upper bound on operations
that preserve the SAML-authenticated session under this profile. In
particular:

* the authorization server MUST NOT issue a refresh token whose validity
  extends beyond `SessionNotOnOrAfter`;
* if it issues an access token or ID Token directly from the assertion and that
  token is intended to represent the same authenticated session, the token
  expiration time MUST NOT extend beyond `SessionNotOnOrAfter`; and
* once `SessionNotOnOrAfter` has passed, the authorization server MUST reject
  further session-preserving use of the assertion or derived refresh token.

If the authorization server receives an authoritative signal that the
underlying SAML-authenticated session ended earlier than
`SessionNotOnOrAfter`, it MUST treat that earlier event as the operative end of
the migrated session for purposes of this profile.

The following SAML data has no direct standard OpenID Connect claim mapping:

* `AuthnContextDecl`
* `AuthnContextDeclRef`
* `AuthnContext/AuthenticatingAuthority`
* `SubjectLocality`

If the `AuthnContext` contains an `AuthnContextDeclRef` element but no
`AuthnContextClassRef`, the authorization server MUST NOT emit an `acr` claim
unless local policy defines a deterministic mapping from that declaration
reference to a specific ACR value. If no such mapping exists, `acr` MUST be
omitted.

Such values MAY be conveyed in private claims by prior agreement between the
authorization server and client. Deployments using such claims SHOULD use
collision-resistant claim names.

## Attribute Claims {#attribute-claims}

A `NameID` in emailAddress format is not by itself an attribute mapping input.
However, if the assertion contains a `NameID` whose format is
`urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress`, or whose semantics are
those of an email address under local policy, the authorization server MAY use
that value as the OpenID Connect `email` claim when:

* claim release policy for the relying-party relationship permits release of
  `email`;
* the value is bound by local policy to the resolved Local Account; and
* no higher-precedence SAML attribute mapping produces a conflicting `email`
  claim value.

If the emailAddress `NameID` conflicts with an `email` value derived from a
SAML attribute and no deterministic local precedence rule exists, the
authorization server SHOULD omit the `email` claim rather than guess. An
emailAddress `NameID` MUST NOT by itself cause the authorization server to
emit `email_verified=true`.

When a SAML attribute directly corresponds to a registered OpenID Connect claim,
the authorization server SHOULD emit that claim. Common mappings include:

* `givenName` or `given_name` to `given_name`
* `sn`, `surname`, or `family_name` to `family_name`
* `displayName` or `name` to `name`
* `mail` or `email` to `email`
* `uid` or `preferred_username` to `preferred_username`
* `telephoneNumber` or `phone_number` to `phone_number`

When the SAML deployment uses attribute `Name` values based on object
identifiers or URIs, the authorization server MAY apply a deployment-specific
mapping table that resolves those names to the equivalent OpenID Connect claim
names.

Appendix B provides non-normative deployment guidance for common InCommon and
`eduPerson` attribute mappings.

All `AttributeStatement` elements in the assertion MUST be processed as a
single logical attribute set. Attributes having the same `Name` and the same
`NameFormat` are part of the same logical attribute and their values MUST be
combined in document order. Attributes that differ by `Name` or `NameFormat`
MUST be treated as distinct inputs even if their `FriendlyName` values are the
same.

An omitted `NameFormat` MUST be treated as
`urn:oasis:names:tc:SAML:2.0:attrname-format:unspecified` for purposes of
attribute grouping and mapping under this profile.

Mapping decisions MUST be based primarily on the SAML attribute `Name` and
`NameFormat`. `FriendlyName` MAY be used only as a secondary hint when no
deterministic `Name`-based mapping exists. If `Name`, `NameFormat`, and
`FriendlyName` would imply conflicting claim mappings, the authorization server
MUST use the `Name` and `NameFormat` interpretation.

The registered OpenID Connect claims listed above are single-valued string
claims if present. The `auth_time` claim is a single numeric value. The `amr`
claim is an array of strings.

Attribute processing MUST follow these rules:

* a single SAML `AttributeValue` maps to a single JSON value;
* multiple SAML `AttributeValue` elements MAY map to a JSON array only for
  claims whose syntax permits arrays or for private claims defined by prior
  agreement;
* boolean-valued SAML attributes SHOULD map to JSON booleans when the XML type
  is unambiguous;
* if multiple source values are available for a registered single-valued claim,
  the authorization server MUST either apply a deterministic, claim-specific
  precedence rule or omit that claim;
* if multiple distinct SAML attributes could map to the same registered claim,
  the authorization server MUST apply a deterministic precedence rule based on
  SAML attribute identifiers or omit that claim;
* the authorization server MUST NOT emit a JSON array for a registered
  single-valued claim; and
* generic attribute mapping MUST NOT overwrite `sub`, `auth_time`, `acr`,
  `amr`, or `sid` values that were derived by the rules in this document.

The authorization server MUST NOT infer `email_verified` from the mere
presence of an `email` attribute. The authorization server MUST NOT infer
`phone_number_verified` from the mere presence of a `phone_number` attribute.
These verified status claims MUST be populated only when an explicit
verification assertion is available from the SAML deployment or from a local
policy determination.

Attributes that do not correspond to standard OpenID Connect claims MAY be
returned as private claims when both parties have a prior agreement on syntax
and semantics.

## UserInfo Responses and OIDC Output Consistency {#userinfo-oidc-output-consistency}

If an access token issued under this profile is presented to the OpenID
Provider's UserInfo endpoint, the OpenID Provider MUST process that request in
accordance with {{OIDC-CORE}} and the additional rules in this section.

The UserInfo response:

* MUST include the `sub` claim;
* MUST use a `sub` value consistent with {{subject-identifier-mapping}};
* MUST apply the claim mapping rules in {{authentication-event-claims}} and
  {{attribute-claims}} to any returned claims; and
* MUST NOT include claims that would not be releasable to the same relying
  party under this profile.

The OpenID Provider MUST determine which claims to return based on the granted
scope, any other OpenID Connect claim-selection mechanism it supports, and the
claim release policy applicable to the relying-party relationship represented by
the bound `saml_sp_entity_id`.

If an ID Token and a UserInfo response are both issued under the same migrated
authorization context, the `sub` value in the UserInfo response MUST exactly
match the `sub` value in the corresponding ID Token. If the UserInfo response
includes `auth_time`, `acr`, `amr`, or `sid`, those values MUST be derived
according to {{authentication-event-claims}}. When an ID Token has been issued for the same
migrated authorization context, the authorization server MUST cache the values
of `auth_time`, `acr`, `amr`, and `sid` at the time of ID Token issuance and
MUST return those cached values in any UserInfo response for the duration of
the access token's validity, even if the originating SAML assertion has since
expired. The UserInfo values for these claims MUST match the corresponding
claims in the most recently issued ID Token for that migrated authorization
context.

This profile does not require the UserInfo response to repeat every claim
available in an ID Token, nor does it require claims omitted from the ID Token
to be present in UserInfo. However, any claim returned from UserInfo under this
profile MUST reflect the same Local Account resolution, subject continuity, and
claim release policy used for the access token.

The claim release policy applicable at the time of access token issuance
and the claim release policy applicable at the time of the UserInfo request
MUST both permit any claim returned from UserInfo. The authorization server MAY
retain the issuance-time policy snapshot for continuity, but it MUST NOT use
that snapshot to return claims that are disallowed by current policy or by the
current Local Account or session state.

# Security Considerations {#security-considerations}

When a SAML assertion is submitted without an `InResponseTo` element in its
`SubjectConfirmationData`, the authorization server cannot confirm that the
assertion was issued in response to a specific `<AuthnRequest>`. This pattern
is common in IdP-initiated SSO flows. Deployments SHOULD require SP-initiated
flows and SHOULD reject assertions that lack `InResponseTo` unless the
deployment threat model has explicitly assessed and accepted the associated
risks, including the potential for session fixation and assertion injection
attacks.

XML Signature Wrapping (XSW) attacks allow an adversary to manipulate a
SAML document by inserting a malicious element while retaining a valid
signature on a benign element elsewhere in the document structure.
Implementations MUST validate SAML signatures in a manner resistant to XSW
attacks. Specifically:

* when the authorization server relies on an assertion signature, it MUST
  verify that the element referenced by the signature's `Reference/@URI` is
  the same assertion element being processed; and
* when the authorization server relies on a signed SAML `Response` wrapper, it
  MUST verify that the element referenced by the signature's `Reference/@URI`
  is the same `Response` element being processed, and that the effective
  enclosed assertion selected under {{saml-input-validation}} is the unique
  assertion bound to that validated wrapper.

The authorization server MUST reject any document structure where these
bindings cannot be confirmed unambiguously. Implementations SHOULD use
established SAML processing libraries that have been tested against known XSW
attack patterns.

Implementations MUST NOT accept SAML assertions signed using deprecated
cryptographic algorithms. RSA with SHA-1
(`http://www.w3.org/2000/09/xmldsig#rsa-sha1`) and DSA with SHA-1
(`http://www.w3.org/2000/09/xmldsig#dsa-sha1`) MUST be rejected. Authorization
servers SHOULD require RSA with SHA-256 or stronger and SHOULD document their
minimum acceptable algorithm requirements.

When an authorization server or migration tooling fetches the
`saml_metadata_uri`, the fetch target is derived from a configuration value
that may be attacker-influenced in some deployment models. Implementations
MUST restrict fetches to URIs that use the HTTPS scheme and MUST NOT follow
redirects to non-HTTPS URIs. Deployments SHOULD additionally constrain
permitted fetch targets to a pre-approved allowlist of hosts, in order to
prevent Server-Side Request Forgery (SSRF) attacks that could expose internal
services.

Mixing SAML trust inputs and OAuth trust inputs creates a risk of protocol
confusion. Implementations MUST validate SAML assertions using SAML metadata and
MUST validate ID Tokens and other JOSE objects using OAuth or OpenID Connect key
discovery. Implementations MUST NOT assume that a key published in one metadata
format is automatically valid in the other.

Authorization servers MUST apply transport security to the token endpoint,
introspection endpoint, metadata retrieval, and any other endpoint used by this
profile in the same manner as the underlying OAuth, OpenID Connect, and SAML
specifications that they extend.

SAML assertions used with Token Exchange or introspection are bearer artifacts.
Authorization servers MUST detect and prevent replay of the same assertion
according to the security properties of the underlying SAML deployment, any
SAML `OneTimeUse` condition, and the replay rules in {{saml-input-validation}}. Client
authentication alone is not sufficient
protection against replay of a stolen assertion. Authorization servers deployed
across multiple nodes MUST ensure that assertion identifier state used for
replay detection is shared across all nodes or that equivalent distributed
coordination is in place; per-node replay stores are insufficient.

When this profile accepts a signed SAML `Response` as a wrapper for the
effective assertion, response-level signature validation does not by itself
validate response-level protocol fields such as `Destination` or
`Response/@InResponseTo`. This profile does require the wrapped `Response` to
carry a top-level SAML `StatusCode` of `Success` with no subordinate status
code, but other response-level protocol checks remain outside this profile.
Deployments that depend on those additional checks MUST ensure they are
performed outside this profile before the enclosed assertion is used to issue
tokens or return claims.

The bindings defined by `saml_sp_entity_id` and `saml_idp_entity_id` are
security-critical. If a client can register an arbitrary `saml_sp_entity_id`, it
may be able to inherit claim release policy or pairwise identifiers intended for
another relying party. Authorization servers therefore MUST protect registration
and administrative mapping of these values.

When multiple OAuth clients share the same `saml_sp_entity_id`, the
authorization server is intentionally treating them as the same relying-party
context. Such sharing MUST therefore be an explicit IdP or authorization server
policy decision. Accidental sharing can expose the same pairwise `sub` values
and claims to clients that should have been isolated from one another.

Incorrect `amr` translation can overstate the assurance of an authentication.
Implementations SHOULD pass through the SAML `AuthnContextClassRef` as the
`acr` value rather than attempting to derive `amr` from it, and SHOULD emit
`amr` values only when the actual authentication methods are known.

Implementations MUST NOT use mutable identifiers, transient identifiers, or
reassignable `NameID` formats as OpenID Connect `sub` values. Doing so would
break the `sub` stability requirements and can cause account takeover or
misbinding after identifier reuse.

The introspection alternative can expose normalized claims without minting OAuth
tokens. Authorization servers therefore MUST strongly authenticate clients to
the introspection endpoint and MUST return
`"active": false` when the client is not entitled to introspect the presented
SAML assertion.

When issuing an ID Token directly from Token Exchange, the authorization server
MUST ensure that the resulting token meets the same signing, audience, issuer,
and claim integrity requirements that would apply if the token had been issued
through an ordinary OpenID Connect flow.

# Privacy Considerations {#privacy-considerations}

This profile exists in part to preserve privacy properties during migration.
Deployments that previously relied on SAML pairwise subject identifiers SHOULD
continue to use pairwise identifiers after migration rather than silently
switching to public identifiers.

The `saml_sp_entity_id` parameter is important for privacy because it provides a
stable relying-party key for pairwise continuity. If a deployment instead
derives pairwise subjects from unrelated OAuth client metadata, the migrated
application can observe a different subject than it observed as a SAML SP,
forcing account relinking or introducing correlation errors.

When multiple OAuth clients share the same `saml_sp_entity_id`, they will
observe the same relying-party-specific subject continuity and claim release
under this profile. Deployments SHOULD do this only when those clients are
intended to represent the same historical relying-party context.

Authorization servers SHOULD release no more claims to the migrated client than
would have been available to the same relying party in the SAML deployment,
unless separate policy authorizes broader disclosure.

Private claim mappings can introduce new correlation vectors across migrated
clients. Deployments SHOULD therefore minimize use of private claims and SHOULD
avoid emitting values whose semantics were not already established for the same
relying-party relationship.

When the introspection pattern is used, the authorization
server SHOULD return only the minimum claims necessary for the authorized
client, because introspection can reveal identity data without the normal
indirection of token issuance.

# IANA Considerations {#iana-considerations}

## OAuth Dynamic Client Registration Metadata {#iana-client-registration-metadata}

This document requests registration of the following value in the OAuth Dynamic
Client Registration Metadata registry established by {{RFC7591}}:

* Client Metadata Name: `saml_sp_entity_id`
* Client Metadata Description: Identifier of the SAML 2.0 Service Provider
  entity bound to the client for migration and subject continuity
* Change Controller: IESG
* Specification Document(s): This document, {{client-registration-saml-sp-entity-id}}

## OAuth Authorization Server Metadata {#iana-as-metadata}

This document requests registration of the following values in the OAuth
Authorization Server Metadata registry established by {{RFC8414}}:

* Metadata Name: `saml_idp_entity_id`
* Metadata Description: Identifier of the SAML 2.0 Identity Provider entity
  bound to the authorization server issuer
* Change Controller: IESG
* Specification Document(s): This document, {{metadata-saml-idp-entity-id}}

* Metadata Name: `saml_metadata_uri`
* Metadata Description: HTTPS URI for the SAML metadata describing the SAML
  Identity Provider entity bound to the authorization server issuer
* Change Controller: IESG
* Specification Document(s): This document, {{metadata-saml-metadata-uri}}

* Metadata Name: `token_exchange_requested_token_types_supported`
* Metadata Description: JSON array of token type URIs accepted as
  `requested_token_type` values for SAML token exchange under this profile
* Change Controller: IESG
* Specification Document(s): This document, {{metadata-token-exchange-requested-token-types-supported}}

* Metadata Name: `introspection_token_types_supported`
* Metadata Description: JSON array of token type URIs identifying token types
  accepted for direct introspection at the authorization server's
  introspection endpoint according to this profile
* Change Controller: IESG
* Specification Document(s): This document, {{metadata-introspection-token-types-supported}}

## OAuth Token Type Hints {#iana-token-type-hints}

This document requests registration of the following value in the OAuth Token
Type Hints registry established by {{RFC7009}}. The token type URI below is
defined by {{RFC8693}} as a token type identifier; this registration makes it
available as a `token_type_hint` value at introspection and revocation
endpoints, which use a separate registry.

* Hint Value: `urn:ietf:params:oauth:token-type:saml2`
* Change Controller: IESG
* Specification Document(s): This document, {{introspection-request}}

## OAuth Token Introspection Response Registry {#iana-introspection-response-registry}

This document requests registration of the following values in the OAuth Token
Introspection Response registry established by {{RFC7662}}:

* Response Name: `claims`
* Response Description: JSON object containing normalized subject and claim
  values derived from the introspected SAML assertion for the authorized client
* Change Controller: IESG
* Specification Document(s): This document, {{introspection-successful-response}}

* Response Name: `saml`
* Response Description: JSON object containing normalized SAML protocol metadata
  extracted from the validated SAML input
* Change Controller: IESG
* Specification Document(s): This document, {{introspection-successful-response}}

--- back

# Examples

This appendix is non-normative.

Unless otherwise noted, these examples use HTTP Basic client authentication for
brevity. Encoded SAML values are shortened placeholders.

The examples focus on the three primary deployment scenarios for this profile:
converting SAML input into an ID Token, delegating SAML validation to the
introspection endpoint, and converting SAML input into OAuth access tokens or
refresh tokens.

All examples assume a confidential client with the following registration:

~~~ json
{
  "client_name": "Calendar Example",
  "redirect_uris": [
    "https://calendar.example.com/callback"
  ],
  "token_endpoint_auth_method": "client_secret_basic",
  "subject_type": "pairwise",
  "saml_sp_entity_id": "https://calendar.example.com/saml/sp"
}
~~~

and an authorization server publishing the following relevant metadata:

~~~ json
{
  "issuer": "https://login.example.com",
  "authorization_endpoint": "https://login.example.com/authorize",
  "token_endpoint": "https://login.example.com/token",
  "userinfo_endpoint": "https://login.example.com/userinfo",
  "jwks_uri": "https://login.example.com/jwks.json",
  "introspection_endpoint": "https://login.example.com/introspect",
  "registration_endpoint": "https://login.example.com/register",
  "grant_types_supported": [
    "authorization_code",
    "refresh_token",
    "urn:ietf:params:oauth:grant-type:token-exchange"
  ],
  "saml_idp_entity_id": "https://login.example.com/idp",
  "saml_metadata_uri": "https://login.example.com/idp/metadata",
  "token_exchange_requested_token_types_supported": [
    "urn:ietf:params:oauth:token-type:refresh_token",
    "urn:ietf:params:oauth:token-type:id_token",
    "urn:ietf:params:oauth:token-type:access_token"
  ],
  "introspection_token_types_supported": [
    "urn:ietf:params:oauth:token-type:saml2"
  ]
}
~~~

## Convert SAML to an ID Token

~~~ http
POST /token HTTP/1.1
Host: login.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&requested_token_type=urn:ietf:params:oauth:token-type:id_token
&scope=openid+profile+email
&subject_token=PHNhbWwyOkFzc2VydGlvbiB4bWxuczpzYW1sMj0iLi4uIj4uLi48L3NhbWwyOkFzc2VydGlvbj4
&subject_token_type=urn:ietf:params:oauth:token-type:saml2
~~~

~~~ json
{
  "issued_token_type": "urn:ietf:params:oauth:token-type:id_token",
  "access_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjIyIn0...",
  "token_type": "N_A",
  "scope": "openid profile email",
  "expires_in": 3600
}
~~~

The ID Token carried in the `access_token` response member might contain claims
like:

~~~ json
{
  "iss": "https://login.example.com",
  "sub": "p7b4cf5d-9c2f-4f22-a6b9-6e3d8df5a1b0",
  "aud": "s6BhdRkqt3",
  "exp": 1776805200,
  "iat": 1776804900,
  "auth_time": 1776794400,
  "acr": "urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport",
  "sid": "op-sid-61b7d66f-4a6f-4f04-b0e5-9b8176d92ad0",
  "email": "alice@example.com",
  "given_name": "Alice",
  "family_name": "Ng"
}
~~~

## Delegate SAML Validation with Introspection

~~~ http
POST /introspect HTTP/1.1
Host: login.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW

token=PHNhbWwycDpSZXNwb25zZSB4bWxuczpzYW1sMj0iLi4uIiB4bWxuczpzYW1sMnA9Ii4uLiI-Li4uPC9zYW1sMnA6UmVzcG9uc2U-
&token_type_hint=urn:ietf:params:oauth:token-type:saml2
~~~

~~~ json
{
  "active": true,
  "saml": {
    "input_type": "response",
    "response": {
      "id": "_d71b9f4f8b5b4a4b8f2f",
      "issuer": "https://login.example.com/idp",
      "issue_instant": "2026-04-21T18:00:00Z",
      "destination": "https://calendar.example.com/saml/acs",
      "in_response_to": "_sp-authnrequest-8f3a",
      "status_code": "urn:oasis:names:tc:SAML:2.0:status:Success",
      "has_nested_status_code": false
    },
    "assertion": {
      "id": "_a75adf55d9a24d6f8c2b",
      "issuer": "https://login.example.com/idp",
      "issue_instant": "2026-04-21T18:00:00Z",
      "audiences": [
        "https://calendar.example.com/saml/sp"
      ],
      "not_before": "2026-04-21T17:55:00Z",
      "not_on_or_after": "2026-04-21T18:05:00Z",
      "subject_confirmation_method": "urn:oasis:names:tc:SAML:2.0:cm:bearer",
      "subject_confirmation_recipient": "https://calendar.example.com/saml/acs",
      "subject_confirmation_in_response_to": "_sp-authnrequest-8f3a",
      "subject_confirmation_not_on_or_after": "2026-04-21T18:05:00Z"
    }
  },
  "claims": {
    "sub": "p7b4cf5d-9c2f-4f22-a6b9-6e3d8df5a1b0",
    "auth_time": 1776794400,
    "acr": "urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport",
    "sid": "op-sid-61b7d66f-4a6f-4f04-b0e5-9b8176d92ad0",
    "email": "alice@example.com",
    "given_name": "Alice",
    "family_name": "Ng"
  }
}
~~~

Before redirecting the user to the SAML IdP, the SP creates an AuthnRequest and
stores local request state such as:

~~~ json
{
  "request_id": "_sp-authnrequest-8f3a",
  "acs_url": "https://calendar.example.com/saml/acs",
  "sp_entity_id": "https://calendar.example.com/saml/sp",
  "idp_entity_id": "https://login.example.com/idp"
}
~~~

After the authorization server validates XML signatures and SAML assertion
processing rules, the SP performs local correlation and location checks over the
returned JSON values, for example:

~~~ text
saml.input_type == "response"
saml.response.issuer == stored.idp_entity_id
saml.assertion.issuer == stored.idp_entity_id
saml.response.destination == stored.acs_url
saml.response.in_response_to == stored.request_id
saml.assertion.subject_confirmation_recipient == stored.acs_url
saml.assertion.subject_confirmation_in_response_to == stored.request_id
stored.sp_entity_id is in saml.assertion.audiences
saml.response.status_code == "urn:oasis:names:tc:SAML:2.0:status:Success"
saml.response.has_nested_status_code == false
current_time < saml.assertion.subject_confirmation_not_on_or_after
current_time < saml.assertion.not_on_or_after
~~~

## Convert SAML to an Access Token or Refresh Token

The client can request an access token for an explicit API target:

~~~ http
POST /token HTTP/1.1
Host: login.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&requested_token_type=urn:ietf:params:oauth:token-type:access_token
&scope=payments.read+payments.write
&resource=https%3A%2F%2Fapi.example.com%2Fpayments
&audience=payments-api
&subject_token=PHNhbWwyOkFzc2VydGlvbiB4bWxuczpzYW1sMj0iLi4uIj4uLi48L3NhbWwyOkFzc2VydGlvbj4
&subject_token_type=urn:ietf:params:oauth:token-type:saml2
~~~

~~~ json
{
  "issued_token_type": "urn:ietf:params:oauth:token-type:access_token",
  "access_token": "mF_9.B5f-4.1JqM",
  "token_type": "Bearer",
  "scope": "payments.read payments.write",
  "expires_in": 3600
}
~~~

Alternatively, the client can request a refresh token to bootstrap a migrated
session:

~~~ http
POST /token HTTP/1.1
Host: login.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&requested_token_type=urn:ietf:params:oauth:token-type:refresh_token
&scope=openid+offline_access+profile+email
&subject_token=PHNhbWwycDpSZXNwb25zZSB4bWxuczpzYW1sMj0iLi4uIiB4bWxuczpzYW1sMnA9Ii4uLiI-Li4uPC9zYW1sMnA6UmVzcG9uc2U-
&subject_token_type=urn:ietf:params:oauth:token-type:saml2
~~~

~~~ json
{
  "issued_token_type": "urn:ietf:params:oauth:token-type:refresh_token",
  "access_token": "vF9dft4qmTcXkZ26zL8b6u",
  "token_type": "N_A",
  "scope": "openid offline_access profile email",
  "expires_in": 1209600
}
~~~

# InCommon and eduPerson Deployment Guidance

This appendix is non-normative.

Many InCommon deployments use `eduPerson` attributes, including the REFEDS
Research and Scholarship attribute bundle. InCommon deployments commonly use
`urn:oid:` Name values in addition to friendly names; the mapping rules in
{{attribute-claims}} govern how such OID-based names are resolved to OpenID
Connect claims. The core rules in {{claim-mapping}} still govern claim typing,
precedence, and subject handling. Within that framework, deployments commonly
use the following mappings and constraints:

* `mail` to `email`
* `displayName` to `name`
* `givenName` to `given_name`
* `sn` or `surname` to `family_name`
* `eduPersonNickname` to `nickname`, when locally useful
* `telephoneNumber` to `phone_number`, when locally useful
* `eduPersonPrincipalName` to `preferred_username` only when local policy
  permits release as a user-facing identifier; it should not determine `sub`
* `eduPersonScopedAffiliation` in a collision-resistant private claim that
  preserves its multi-valued syntax
* `eduPersonAffiliation` and `eduPersonPrimaryAffiliation` in collision-
  resistant private claims rather than overloaded standard claims
* `eduPersonEntitlement` in a collision-resistant private claim; it should not
  be translated directly into OAuth `scope`
* `eduPersonAssurance` in a collision-resistant private claim or as input to
  local assurance policy; it should not be copied directly into `amr` and
  should not overwrite `acr` unless local policy explicitly defines that
  translation
* `eduPersonUniqueId` as a stable account-linking or subject-continuity input,
  and only secondarily as a private claim when needed
* `eduPersonTargetedID` as a legacy pairwise subject continuity input, not as a
  general-purpose user-facing claim
* `eduPersonOrcid` in a collision-resistant private claim
* `eduPersonPrincipalNamePrior` only by explicit policy for relying parties that
  require historical identifier information

For InCommon-style bundles used only to preserve legacy application behavior,
the authorization server SHOULD prefer standard OpenID Connect claims when
there is a good semantic match and SHOULD retain the original attribute
semantics in private claims rather than mapping them to unrelated standard
claims such as `groups`, `roles`, or `scope`.

# Acknowledgments
{:numbered="false"}

This draft was informed by the SAML interoperability work in the OAuth working
group identity assertion draft and by the deployed semantics of the OASIS SAML
subject identifier profile.
