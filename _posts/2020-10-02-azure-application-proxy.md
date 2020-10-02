---
layout: post
title:  Desktop Analytics Explaination
categories: [Azure,Application Proxy,Publish Web Application]
---
This article will talk about publishing of on-premises application to external users (internet connected). The users will be able to access the on-premises web-based application without connecting to the corporate VPN.

# What is Azure AD Application Proxy (AADAP)
Application Proxy is a feature of Azure AD that enables users to access on-premises web applications from a remote client. Azure Active Directory Application Proxy provides secure remote access to on-premises web applications. Users can access both **cloud and on-premises applications** through an external URL and authentication can be provided by Azure AD or Passthrough to the on-premises authentication provider like Active Directory Domain Controller.

**Note**

>The Application Proxy Connector doesn't require you to open inbound connections through your firewall. User traffic terminates at the Application Proxy Service (in Azure AD). The Application Proxy Connector (on-premises) is responsible for the rest of the communication.

*For more information https://docs.microsoft.com/en-us/azure/active-directory/manage-apps/application-proxy*

# Scenario Introduction
Customer asked to publish its in-house running web application to the users when they are not on the corporate network.

Users must be able to access to the web application without any dialing corporate VPN and users must authenticate for access.

# Lab Environment
I'll be using my Azure subscription with Azure AD **Premium** license, which is a requirement for Azure Application Proxy (AADAP).

My on-premises lab environment consists of:

1. Active Directory Domain Controller (Windows Server 2019)
2. A domain-joined server (Windows Server 2019)

    *On this server, we'll be installing Azure App Proxy Connector*
    
3. Web Server running a web application (Windows Server 2019)

    *I have configured a basic web application for this lab. You can find more details here*


# Implementing Azure AD Application Proxy (AADAP)

To get started, first you need to install the Application Proxy Connector in your on-premises data center. In short, this connector will act as a bridge between your external users and web application that is running on-premises. 

1. Sign-in portal.azure.com 
2. Browse to **Azure AD | Application Proxy**
3. Click **download a connector**

    ![](/images/aadap/aadap_download_connector.png)


4. Log on the server on which you want to install Application Proxy Connector and copy the downloaded connector file. 
5. Before installing the connector on Windows Server 2019, we must disable HTTP2 protocol support in the WinHttp component for Kerberos Constrained Delegation to properly work. This is disabled by default in earlier versions of supported operating systems. Adding the following registry key and restarting the server disables it on Windows Server 2019. Note that this is a machine-wide registry key.
    ```
    [HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Internet Settings\WinHttp] "EnableDefaultHttp2"=dword:00000000
    ```

6. We also need to enable TLS 1.2. To achieve that, set the following registry keys:
    ```
        [HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2]
        [HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2\Client]
        "DisabledByDefault"=dword:00000000
        "Enabled"=dword:00000001
        [HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2\Server]
        "DisabledByDefault"=dword:00000000
        "Enabled"=dword:00000001
        [HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\.NETFramework\v4.0.30319]
    "SchUseStrongCrypto"=dword:00000001
    ```
7. Also, ensure that the Internet Explorer Enhanced Security Configuration is set to Off 
8. Restart the server after setting up the registry keys
9. Run the **Azure AD Application Proxy Connector** executable (run as administrator)

    ![](/images/aadap/aadap_connector_installing.png)

10. During installation it will prompt for Azure tennant credentials where we'll be configuring Application Proxy. Enter credentials and in a moment you will have successful installation of the connector.

### Verify Connector
- To verify on the server where connector is installed, open **services.msc** and locate these two services and their status should be *Running*

    ![](/images/aadap/aadap_connector_services.png)

- To verify through azure portal, select **Azure Active Directory | Application Proxy**. The connector status will show as **Active**.
    ![](/images/aadap/aadap_connector_status.png)


# Publishing On-Premises Web Application
I have configured a one page web application, the details can be seen here. To publish this on-premises application to external users:

- Go to Azure **Active Directory | Enterprise Applications**
- Click **On-premises application**
    ![](/images/aadap/aadap_publish_webapp.png)

- Fill the information
    - Internal URL (*on-premises URL*): http://gswebsite.greysky.local
    - External URL (*internet accessible*): chose the application name and it will complete the external URL by concatinating the default app proxy URL. For example, https://greyskylearning-sysshell.msappproxy.net/
    - Pre-Authentication: Passthrough (or chose Azure AD)
    ![](/images/aadap/aadap_publish_webapp2.png)

# Testing Published Web Application
lets test our published application on a device that has no corporate connectivity, but internet is available.

- Open a internet browser and browse the external URL i.e. https://greyskylearning-sysshell.msappproxy.net/

    ![](/images/aadap/aadap_test_webapp1.png)

- Because I configured the Enterprise Application's authentication as **Passthrough** so browser is simple passing the request to the on-premises application for authentication which is configured as *Windows Authentication* required.
- Enter the on-premises domainname\username and password who has permissions to access this web application.
- The on-premises domain controller will authenticate & authorize the user to access the web application.
    ![](/images/aadap/aadap_test_webapp2.png)