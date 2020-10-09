---
layout: post
title:  Configuring a web application with Windows Authentication
categories: [IIS,Web Application,Windows Authentication]
---
This post is about configuring a simple one page web application that requires Windows Authentication. This sample web application will be used to publish to internet users through Azure AD Application Proxy.

## Web Application Content Files

- Create a folder where you will be placing your web application files. 
- Create a file **default.aspx**. Edit the file in notepad and insert the following code and save file.
    ```html
    <h2>Welcome to GreySky Learning</h2>
    You are sign-in as: <%=User.Identity.Name%>
    ```
- Create another file **web.config**. Edit the file in notepad and insert the following xml code and save file.
    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <configuration>
    <system.web>
        <authorization>
        <allow roles="contoso\SLG-Web-PortalUsers" />
        <deny users="*" />
        </authorization>
    </system.web>
    </configuration>
    ```

*"contoso\SLG-Web-PortalUsers" is the domain group whose members will get access to our web application. Replace the **contoso** to your own AD domain name*

## AD User, Group and DNS Record

- As listed in the above web.config file, create a security group named "SLG-Web-PortalUsers" in your AD domain.

- Create a test user account "webuser01" and make it member of "greysky\SLG-Web-PortalUsers" group. We'll use this test user account to login our web application.

- Although it is not necessary, but good to have a simple DNS name that will represent our on-premises web application. In your local DNS server, create a CNAME record with alias of your choice and target to FQDN of your web server that will host the web application.

    ![](/images/aadap/webapp_dns_cname.jpg)

## IIS Configurations to host web application

- if not already installed, install the Internet Information Services role on the Windows Server, select asp.net as well as windows authentication in the feature selection.

- Run the Internet Information Services (IIS) console.
- Select **Default Web Site** and click **Bindings**.
- Change the port from 80 to something else e.g. 8080 because we'll use the port 80 for our web application.
    ![](/images/aadap/webapp_defaultsite_port.png)

- Configure new web application named for example **GSWebSite**.
- Configure the bindings using DNS record as Host Name and Port 80.
    ![](/images/aadap/webapp_webapp_port.jpg)

- Select the web application **GSWebSite** and click **Authentication**. Disable Anonymous Authentication and Enable Windows Authentication.
    ![](/images/aadap/webapp_webapp_auth.jpg)

- Test the web application by browsing URL http://gswebsite.< domain_name > and enter the test username and password. After authentication, the website will show the welcome message.

<br/>
We have configured a sample web application that allow users to sign-in using their Active Directory credentials. 
Follow up my other post about how to publish this internal/on-premises web application to the internet users using <a href="https://shahbaz-ahmad.github.io/azure-application-proxy/" target="_blank">Azure AD Application Proxy</a>.

<br/>