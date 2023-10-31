# SAML Advanced Use Cases - Part 4: Configuring APM for IdP Chaining

## Use cases for SAML IdP chaining
A possible use case for IdP chaining would be the scenario of a contractor company doing business with you and you want to give them access to your SaaS project management platform or your on-prem version control system (named sp.zerotrust.works in the picture below).
The contractor might have their own IdP (idp-external.zerotrust.works) that checks username, password, second factor and maybe also does some device posture checks.
So there is a need to convert to SAML token issued by the contractors IdP to a token trusted by your application.
We need to bring your IdP (idp-internal.zerotrust.works) into this chain to make have trusted relationships on all sides.

![IdP_Chaining](/assets/IdP_Chaining.png)

## Step-by-step guide
This setup requires two virtual server, one for the IdP of the contractor, one for the IdP of your company. The later will transform the SAML token issued by the IdP of the contractor. For the SimpleSAMLphp server (the SP) we won't need a virtual server, we can access it directly.

### Creating a virtual servers for IdP chaining