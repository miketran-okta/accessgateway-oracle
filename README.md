# Okta Access Gateway for Oracle Fusion Middlware and Apps

This article will describes how to configure the Okta Acccess Gateway to secure Oracle applications such as Application Development Framework (ADF), Oracle WebCenter, Business Intelligence Enterprise Edition (OBIEE), SOA Suite, and other applications that are deployed within a WebLogic container.  In this example, OAG will be used to control access to the WebLogic Administration console but the same steps can be reused for other Oracle applications deployed in WebLogic.

## Background

In the past, Single-Sign On and IDM for Oracle applications was achieved by employing Oracle Identity Management along with a number of other components including

- Oracle Access Manager
- an LDAP directory such as Oracle Internet Directory, Oracle Unified Directory or Directory Server Enterprise Edition (DSEE)
- Oracle Virtual Directory
- Oracle Database
- Oracle HTTP Server

along with the underlying infrastructure comprising server compute, storage, networking and high availability.

Today's large enterprise organizations are looking to futher exploit the benefits of cloud services, retire their legacy on-premise services and eliminate technical debt.  Okta's Identity Cloud and it's Access Gateway can help organizations leverage a modern cloud-native security service for their legacy apps and include components such as strong Multi-Factor Authentication, and contextual access incorporating modern signals such as device posture, risk, behavior and threat intelligence.

The following guide outlines how to secure Oracle applications that have been integrated with Oracle Identity Management and secure them with Okta's Identity Cloud and Okta Access Gateway instead.

# Prerequisites

- Setup the Okta Identity Cloud
- Setup the Okta Access Gateway and setup Okta as the IDP.  OAG documentation can be found <a href="https://help.okta.com/en/prod/Content/Topics/Access-Gateway/ag-main.htm">here</a>.
- An application deployed within a WebLogic container ie OBIEE, SOA, WebCenter.  This example will use Okta Access Gateway to protect the WebLogic administration console.  If you do not have WebLogic, you can find instructions to install it <a href="https://github.com/miketran-okta/weblogic">here</a>
- Support for TLS 1.2 for your WebLogic container


# Okta Access Gateway

This section assumes at you have an Okta tenet setup and the Okta Access Gateway deloyed in your environment.

## Add the hostname of the WebLogic container where your app is sitting to the OAG hosts file

- SSH into your OAG instance
- Select **1 - Network** and then **4 - Edit /etc/hosts**

<img src="https://github.com/miketran-okta/accessgateway-oracle/blob/master/oag/1.png"/>
<br>
<img src="https://github.com/miketran-okta/accessgateway-oracle/blob/master/oag/2.png"/>

- add the IP adderess and hostname of the WebLogic server where your application is deployed
- Save the new host edits and commit the changes.

# WebLogic

## WebLogic 10.3.6 - SSL Setup

Okta leverages TLS1.2 for its various endpoints including the LDAP Interface which will be used to populate the JAAS principals and subjects which Oracle uses for authorization.  This section outlines how to setup SSL for TLSv1.2 and WebLogic 10.3.6, but if you are on WebLogic 12c+ then you can skip to the next section.

- Log into the administrator console of your WebLogic instnace. Typically that is at `http://localhost:7001/console`
- Select **base_domain -> Environments -> Servers -> AdminServer(admin)**
- Select **SSL -> Advanced**
- Update **Hostname Verification** to `Custom Hostname Verifier`
- Enter under **Custom Hostname Verifier** to `weblogic.security.utils.SSLWLSWildcardHostnameVerifier`
- Select the checkbox for **Use JSSE SSL**

<img src="https://github.com/miketran-okta/accessgateway-oracle/blob/master/1.png"/>

## OAM Identity Asserter

If you already are using Oracle Access Manager, then you can leave your existing OAMIdentityAsserter as is.  The OAM Identity Asserter enables header-based authentication by looking for a HTTP header named `OAM_REMOTE_USER` with the username of the authenticated user.  In this setup, that header will populated via Okta's Access Gateway.  If you do not have an OAM Identity Asserter, follow the steps below to add it.

- Select **base_domain -> Security Realms -> myrealm**
- Select the **Providers** tab and then click **New**
- Enter the name of the provider ie `OAMIdentityAsserter`, select **OAMIdentityAsserter** under `Type` and then click OK

<img src="https://github.com/miketran-okta/accessgateway-oracle/blob/master/2.png"/>

- Select **Sufficient** for the `Control Flag` and add **OAM_REMOTE_USER** under the Choosen selection.  Click Save

<img src="https://github.com/miketran-okta/accessgateway-oracle/blob/master/3.png"/>

## LDAP Authentication Provider

After authentication (via HTTP header), Oracle applications require a JAAS subject and principals to support downstream authorization within the Oracle apps.  This process typically entials querying Oracle Internet Directory but in this case we will be using Okta's LDAP Interface.

- Add a new Provider with the name `Okta`, select **LDAPAuthenticator** under `Type` and then click OK

<img src="https://github.com/miketran-okta/accessgateway-oracle/blob/master/4.png"/>

- Select **Sufficient** for the `Control Flag` and click Save.

<img src="https://github.com/miketran-okta/accessgateway-oracle/blob/master/5.png"/>

- Select the `Provider Specific` tab
- Enter the following fields 

  - **Host:** `<okta_tenet_subdomain>.ldap.oktapreview.com` ie dev-123456.ldap.oktapreview.com
  - **Port:** `636`
  - **Principal:** `uid=<okta_admin_username>,dc=<okta_tenet_subdomain>,dc=oktapreview,dc=com` ie uid=ldap_service@novusstream.com,dc=dev-989484,dc=oktapreview,dc=com
  - **Credential** and **Confirm Credential**: okta admin password
  - check **SSLEnabled**
  - **User Base DN:** `ou=users,dc=<okta_tenet_subdomain>,dc=oktapreview,dc=com`
  - **User From Name Filter:** `(uid=%u)`
  
  <img src="https://github.com/miketran-okta/accessgateway-oracle/blob/master/6.png"/>
  
  - **Group Base DN:** `ou=groups,dc=dev-989484,dc=oktapreview,dc=com`
  - **Group From Name Filter:** `(cn=%g)` 
  - Uncheck **Cache Enabled**
  - Click Save
  
  <img src="https://github.com/miketran-okta/accessgateway-oracle/blob/master/7.png"/>
  
## Reorder WebLogic Providers
  
- Go back to the Providers menu and select **DefaultAuthenticator**.  Update the `Control Flag` to **Sufficient**

  <img src="https://github.com/miketran-okta/accessgateway-oracle/blob/master/8.png"/>
  
- Go back to the Providers menu and reorder the providers to as follows

  <img src="https://github.com/miketran-okta/accessgateway-oracle/blob/master/9.png"/>

- Restart the WebLogic Server

# Okta Access Gateway

- Log into the Okta Access Gateway administration console and click plus symbol to add a new application

<img src="https://github.com/miketran-okta/accessgateway-oracle/blob/master/oag/3.png"/>

- Select **Header Based** and then click **Create**

<img src="https://github.com/miketran-okta/accessgateway-oracle/blob/master/oag/4.png"/>

- Enter the following fields 

  - **Label:** `Some Label` ie WebLogic
  - **Public Domain:** The client-facing URL where OAG will expose the application sitting in WebLogic ie `oag-wls.takolive.com`
  - **Protected Web Resource:** The hostname of the WLS server that was added to the OAG hostfile and the port of the deplyed application ie `https://wls.takolive.com:7002`
  - Add `Everyone` under **Groups**
  
  <img src="https://github.com/miketran-okta/accessgateway-oracle/blob/master/oag/5.png"/>
  
  - Click Advanced, and ensure **Content Rewrite** is checked.  Click Next.
  
  <img src="https://github.com/miketran-okta/accessgateway-oracle/blob/master/oag/6.png"/>
  
  - Add a header with the name **OAM_REMOTE_USER** and set its value to **login**
  
    <img src="https://github.com/miketran-okta/accessgateway-oracle/blob/master/oag/7.png"/>
  
  - Click Next and then Done to complete the application setup

# Testing

Access the application via OAG at the configured URL ie https://oag-wls.takolive.com/console/ (ensure the URL resolves to the IP address of OAG which can be done via DNS or updating your local hosts file)

You will be redirected to Okta for authentication and and upon successful validation be redirected to the WebLogic server administration console

<img src="https://github.com/miketran-okta/accessgateway-oracle/blob/master/10.png"/>

**Special thanks to <a href="https://www.linkedin.com/in/fredericohakamine/">Frederico Hakamine</a> and his efforts on putting together the  <a href="https://www.okta.com/security-blog/2018/10/learn-to-integrate-okta-and-oracle-weblogic-with-the-ldap-interface/">Okta LDAP Interface guide</a> which was resused for this article.**

  
  







