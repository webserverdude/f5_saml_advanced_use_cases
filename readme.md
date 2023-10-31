# SAML Advanced Use Cases - Part 1: Overview

In this series of three step-by-step guides I will walk you through two different advanced use cases for SAML authentication with F5 BIG-IP.

## Remind me - what was this SAML thingie again?

SAML is a standard for exchanging authentication and authorization, it is typically used for Single Sign-On (SSO), it eliminates the need for users to remember multiple credentials for every application.
Furthermore SAML is considered as a modern authentication method, while Basic, Kerberos or NTLM are considered as legacy.
The typical use case for SAML involves two parties and a user. One organisation is offering a service, therefore it is called Service Provider (SP), usually a web application or SaaS application.
Another organisation knows about the identity and attributes of the user, this is called the Identity Provider (IdP).
The IdP and the SP agree to mutually trust each other. Afterwards the SP would identify a user by a token issued by a trusted IdP.
When trying to login to the web application (SP), the user will be redirected to the IdP. The user will authenticate against the IdP and will again be redirected (this time with a valid SAML token) to the SP. The SP will validate the token, identify the user and grant permissions based on the attributes in the token.

An organization can store all their user identities in an Identity Provider, like Azure AD or Okta, and build a mutual trust relationship with many Service Providers.
The user will access the first SaaS application in the morning, will authenticate against the IdP and will remain authenticated for the rest of the day (or week). Whatever next application the user will access, he (or she) will be authenticed automatically, because the user is already authenticated to the Identity Provider.

This is the typical flow for such a simple example of how SAML can be used for SSO.

![SAML_flow](/assets/SAML_flow.png)

Another benefit of this mutual trust relationshop, the Service Provider (SP) doesn't need to know anything about the user (not even a password).
And __bonus__: Identity Providers usually also offer strong authentication with MFA.

## Advanced use cases

I will discuss two advancded use cases, one of IdP chaining and one of inline SSO.

The use case of inline SSO would be usefull for on premise web applications. The BIG-IP would sit between the IdP and the internal web app, which is the SP.
If the internal web app is the SP the BIG-IP would be only a loadbalancer and would not add any security. Or it would try to transform the modern SAML token to a legacy authentication method like Kerberos, HTTP headers or NTLM.
With inline SAML SSL the BIG-IP will accept and validate the token from the public IdP and transform it to another token trusted by the web application (SP).
The benefits are: you can keep a modern authentication method all the way and still use the security features of an identity-aware proxy (BIG-IP APM).
The details are described in this guide: [SAML Advanced Use Cases - Part 3: Configuring APM for Inline SSO](/setup_inline_IdP.md)

![flow_inline_IdP](/assets/flow_inline_IdP.png)

The other use case is called IdP chaining. In this use case an organization would operate their own Identity Provider on BIG-IP APM and would build a mutual trust with another IdP, maybe Azure AD or Okta. The second IdP would then autenticate the user to the web application. In this use case the second IdP would be SP and IdP at the same time.
The benefit of this use could be, that the first organisation, running their own IdP, would not have to give their password to the IdP service provider. Or they could perform some additional checks on the client with the help of BIG-IP APM.
The details are described in this guide: [SAML Advanced Use Cases - Part 4: Configuring APM for IdP Chaining](/setup_IdP_chaining.md)

![flow_IdP_chaining](/assets/flow_IdP_chaining.png)

__But before we can start__ with the use cases I described above, we need to setup an application that will serve us as a Service Provider.
Therefore we start with this guide: [SAML Advanced Use Cases - Part 2: Setting up SimpleSAMLphp](/setup_SimpleSAMLphp.md)
