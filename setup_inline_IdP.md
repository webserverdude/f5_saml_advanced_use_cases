# SAML Advanced Use Cases - Part 3: Configuring APM for Inline SSO

## Use cases for SAML inline SSO

The common use case for SAML SSO is having one Identity Provider (Azure AD, Okta or similar) and one or many Service Provider (typically a web application). With BIG-IP you see two common scenarios:
1. The BIG-IP APM is the SAML SP and transforms the token to something the web application would accept for authentication, like a HTTP header, a Kerberos ticket or another supported SSO object.
2. The BIG-IP is not part of the authentication and just loadbalances to the web application server.

The second use case is not a good implementation from the security perspective, the web application is exposed without further protection. An attacker might try an attack on an URL that doesn't required authentication.
The first use case is a better implementation from the security perspective, but the some of the SSO methods are either outdated or cumbersome of impossible to configure.

There is a third use case I want to present in this write-up is using SAML inline. The BIG-IP APM would be both, SAML SP and IdP at the same time.
On the public facing side the BIG-IP would be SP and have a federation with one of the well know IdPs. On the internal side the BIG-IP would transform the SAML token and become to IdP to the web application which resides on the internal networks.

![Inline_IdP](/assets/Inline_IdP.png)

## Requirements for configuring APM as a SAML IdP for inline SSO
This kind of deployment has a couple of requirements that you should pay attention to:

* The external DNS should point the apps/SP's hostname to the APM. For example, if the internal SP is named app.domain.com, it should resolve to the APM virtual server externally.
* After configuring the SP connector (preferably with metadata), edit and set its SP Location Settings to `Internal`.
* SSO is configured on Access Profile SSO/Auth Domains configuration page.

## Step-by-step guide

This guide will take you to the steps required to configure APM for Inline SSO, this steps are:
1. Create the required configuration objects in LTM, like virtual servers, pool, SSL profiles
2. Configure SAML SP and IdP in APM
3. Test our configuration

### Creating a virtual server for SAML inline SSO

This setup requires two virtuals, one for the inline SAML and one that we use to mimic the public IdP.

First the SSL profile, the virtual and to pool for the inline SAML.

1. Import SSL certificate & private key you created during the setup of your SimpleSAMLphp server to your BIG-IP.
2. Create a client SSL profile with this certificate & private key.
```b
ltm profile client-ssl pr_clientssl_sp_zerotrust_works {
    app-service none
    cert-key-chain {
        sp_zerotrust_works_0 {
            cert sp_zerotrust_works
            key sp_zerotrust_works
        }
    }
    defaults-from pr_clientssl
    inherit-ca-certkeychain true
    inherit-certkeychain false
}
```
3. Create a virtual server and a pool with the following settings.
```
ltm virtual vs_sp_zerotrust_works {
    destination 10.100.155.160:https
    ip-protocol tcp
    mask 255.255.255.255
    pool pl_sp_zerotrust_works
    profiles {
        f5-tcp-progressive { }
        http { }
        pr_clientssl_sp_zerotrust_works {
            context clientside
        }
        serverssl {
            context serverside
        }
    }
    serverssl-use-sni disabled
    source 0.0.0.0/0
    source-address-translation {
        type automap
    }
    translate-address enabled
    translate-port enabled
}
ltm pool pl_sp_zerotrust_works {
    members {
        10.100.153.70:https {
            address 10.100.153.70
            session monitor-enabled
            state up
        }
    }
    monitor https
}
```

Next the virtual server that we will use to mimic the public IdP. In a real world scenario this would be Okta, AzureAD, OneLogin or similar.

### Configure SAML SP and IdP in APM

1. Export metadata from SimpleSAMLphp server (SimpleSAMLphp WebGUI)

    Open the following URL on your SimpleSAMLphp server `/simplesaml/admin/index.php` and navigate to the `Federation` tab. There you can download the SP metadata that you will have to import to your BIG-IP. Save it to an XML file.

2. Import metadata from SimpleSAMLphp server (External SP Connectors)

    On your BIG-IP go to `Access  ››  Federation : SAML Identity Provider : External SP Connectors` and import the metadata you aquired in step 1.

3. Create internal SAML IdP Service (Local IdP Services)

    Your next steps is to go to `Access  ››  Federation : SAML Identity Provider : Local IdP Services` and to create a new IdP Service. This will be the one to federate with your SimpleSAMLphp server.
    As `IdP Entity ID` I choose the FQDN of the virtual and of the SimpleSAMLphp, `https://sp.zerotrust.works`, as you can see in the diagram above. The entity ID always includes the protol - `https://`.
    And as the value for `Host` I also used the FQDN, this time without the protocol.

    ![create_internal_IdP](/assets/create_internal_IdP.png)

    Pay attention to the following settings. In the `Assertation Settings` choose `Transient Identifier` for `Assertation Subject Type`.

    ![Assertation_Subject_Type](/assets/Assertation_Subject_Type.png)

    And in `Security Settings` choose the `Signing Key` and `Signing Key` you imported from SimpleSAMLphp while creating the virtual server. This is important, because SimpleSAMLphp expects signed repsonses and asserations.

4. Export metadata from internal SAML IdP Service (Local IdP Services)

    Now while still in the `Access  ››  Federation : SAML Identity Provider : Local IdP Services` export the metadata from the IdP you just created.

5. Import metadata from internal SAML IdP Service to SimpleSAMLphp server (SimpleSAMLphp WebGUI)

    To import this metadata to the SimpleSAMLphp server go back to the `Federation` tab on the WebGUI and use the `XML to SimpleSAMLphp metadata converter` in the `Tools` section.
    Copy the output to the clipboard and SSH to your server. Go to the folder where your SimpleSAMLphp installation is and create a copy of the `saml20-idp-hosted.php.dist` file.

    ```bash
    root@simpleSAMLphp:/var/www/simplesamlphp# cd metadata/
    root@simpleSAMLphp:/var/www/simplesamlphp/metadata# cp saml20-idp-remote.php.dist saml20-idp-remote.php
    ```

    Now modify the `saml20-idp-remote.php` and append the metadata you received from the converter.

6. Binding metadata from SimpleSAMLphp server to internal SAML IdP Service (Local IdP Services)

    Go back to your BIG-IP and navigate to `Access  ››  Federation : SAML Identity Provider : Local IdP Services`, click on Bind/Unbind SP Connectors.
    Now bind the external SP connector from the SimpleSAMLphp server and the IdP for internal use.
    The next step is __important__: Go to `Access  ››  Federation : SAML Identity Provider : External SP Connectors` and modify the connector you just created.

    First change the `Security Settings` and check the boxes for `Response must be signed` and `Assertation must be signed`.

    ![Edit_SP_connector1](/assets/Edit_SP_connector1.png)

    And then go to `SP Location Settings` and change the value for `Service Provider Location` to `internal`.

    ![Edit_SP_connector2](/assets/Edit_SP_connector2.png)

7. Create external SAML IdP Service (Local IdP Services)

    Go back to your BIG-IP and go to `Access  ››  Federation : SAML Identity Provider : Local IdP Services` and to create another new IdP Service. This will be the one to mimic the public SAML IdP.
    Again the `IdP Entity ID` is with the protocol - in my example `https://idp.zerotrust.works` and the `Host` is the same without protocol.

8. Create external SAML SP Service (Local SP Services)

    Next you go to `Access  ››  Federation : SAML Service Provider : Local SP Services` and create the _public_ or _Internet-facing_ `SAML Service Provider`.
    Again the  Entity ID` is with the protocol - in my example `https://sp.zerotrust.works` and the `Host` is the same without protocol.

9. Bind external SAML SP & IdP (Local IdP Services)

    Now bind these two together. No hidden magic here.

10. Create SAML Resources (SAML Resources)

    The final step in the SAML configuration is to put all the configuration objects created above together.
    Navigate to `Access  ››  Federation : SAML Resources` and create two `SAML Resources`, one for the internal and one for the external binding.
    The value for the `SSO Configuration ` is the name of the IdP objected created in `Access  ››  Federation : SAML Identity Provider : Local IdP Services`.

11. Configuring an access policy for the SAML IdP service

    For the virtual that will mimic the public SAML IdP I created an Access Policy of the type __ALL__ with __Modern__ customization.
    
    ![AP_access_idp_zerotrust_works](/assets/AP_access_idp_zerotrust_works.png)
    
    This uses a local DB for the purpose if this being a local demo environment. And in the `Resource Assign` policy action I assign the __SAML Resource__ created above, bing the external IdP and the external SP configuration objects.
    __Don't forget__ to bind this access policy to the virtual server.

12. Configuring an access policy for the SAML inline virtual server

    For the virtual server the will to the inline SAML and the transformation of the token, I created another Access Policy of the type __ALL__ with __Modern__ customization.
    
    ![AP_access_sp_zerotrust_works](/assets/AP_access_sp_zerotrust_works.png)
    
    First this Access Policy validates the SAML token issued by the Access Policy from the step above.
    Then we transform the data from this token in a `Variable Assign` policy action, as follows
    ```
    session.logon.last.username = Session Variable session.saml.last.identity
    session.logon.last.logonname = Session Variable session.saml.last.identity
    ```
    
    ![AP_variable_assign](/assets/AP_variable_assign.png)

    And then we do again a `Resource Assign` and issue the SAML token to the backend.
    __Also don't forget__ to bind this access policy to the virtual server.

## Putting our work to a test

Now we access the following URI on our server: `/simplesaml/module.php/admin/test`. Since we protected the whole server, we will already be redirected to the IdP.
Authenticate to the Login page and, if our configuration is good, we continue to this URI.

![simpleSAMLphp_test](/assets/simpleSAMLphp_test.png)

Click on the link to `default-sp` and follow the steps, you should again go through all the redirects and end up in a page like this:

![simpleSAMLphp_success](/assets/simpleSAMLphp_success.png)

That's it. You successfully configured inline SAML authentication with BIG-IP APM.
