# krb5-secure-my-url
Kerberos authentication mechanism for nginx web server

# ABOUT
This project implements the Single Sign On (SSO) using kerberos configured with OpenLDAP backend used to auhthenticate clients against HTTP service.
Only an authenticated client with kerberos can have access to the server and reach the url

# KEY-CONCEPTS
- We Used 3 linux-containers (lxc/lxd) to run the kerberos-kdc-server, kerberos-client, kerberos-http :
    - Kerberos-kdc-server: 
      - consists of three parts: a database of all principals, the authentication server, and the ticket granting server.
      - For each realm there must be at least one KDC.
    - kerberos-client: Our kerberos client which will have access to any kerberized services once he has successfully logged into the system.
    - kerberos-http: The HTTP server that runs nginx configured with SPNEGO mechanism
    --> our 3 containers using the default lxc network lxdbr0 (we use the hostname as domain name [example: curl http://kerberos-http] )
 - Kerberos :
    - Realm : EXAMPLE.COM
    - Client Principal : ubuntu@EXAMPLE.COM (with no admin priviliges)
    - Admin Principal : ubuntu/admin@EXAMPLE.COM
    - Service Principal : HTTP/kerberos-http.lxd@EXAMPLE.COM
    - ldap client Principal : john@EXAMPLE.COM
  - OpenLDAP :
      - Exists on the same kerberos-kdc-server server for simplicity (taking advantage of the unix socket )
      - Can store Kerberos principals as opposed to a local on-disk database
   - Nginx : we have to add the spnego module to nginx conf in order to enable negotiation authentication
  
  # Setup
  - curl v7.81.0 (make sure supports GSS-API SPNEGO)
  - lxc v5.21
  - ubuntu 22.04 images
  - nginx v1.18.0
   
  # Configuration :
   - kerberos : We followed these steps to install and configure kerberos : [![here]()](https://ubuntu.com/server/docs/kerberos-introduction).
      - kerberos server config file : **krb5.conf**
      - kerberos kdc config file **krb5kdc.conf**
   - openLDAP : Installing and configuring slapd : [![here]()](https://ubuntu.com/server/docs/service-ldap-introduction). You can find users that were added into the ldap server in **add_content.ldif** file
   - Kerberos with OpenLDAP backend : A good documented steps : [![here]()](https://ubuntu.com/server/docs/service-kerberos-with-openldap-backend).
   - Kerberos client : Configure a Linux system as a Kerberos client  [![here]()](https://ubuntu.com/server/docs/service-kerberos-workstation-auth).
   - nginx with spnego module : Follow this cheatsheet : [![here]()](https://docs.j7k6.org/sso-nginx-kerberos-spnego-debian/) :
      - nginx config file : **nginx.conf**
      - default http server (on port 80) that is protected : **default**
# Workflow :
   - dd ubuntu@EXAMPLE.COM and ubuntu/admin@EXAMPLE.COM as Principals in kerberos
   - add HTTP/kerberos-http.lxd@EXAMPLE.COM as service principal
   - Extract the key from the KDC and store it in the kerberos-http server using ktadd utility. If the kadmin utility not available in the service host , extract the keytab in the kerberos server and copy it into the http-server (using scp for example ).
      - Copy the file into /etc/krb5.keytab
      - Make sure its in mode 0600 and owned by root:root
      - A screenshot about the content of the keytab file :
      
      ![image](https://user-images.githubusercontent.com/59982299/229327165-5c4f884e-9d4a-4ebd-a033-afc30c5552b9.png)
      
      - Add the necessary configuration for nginx (auth_gss and auth_gss_keytab )
      - Rload the nginx server
      - Start the client container and log with the ubuntu user
      - generate a new TGT ticket using kinit command
      - check the ticket using the klist command
      - curl into the http server ```KRB5_TRACE=/dev/stderr curl -v -i  --negotiate -u:  http://kerberos-http```
        - run curl command with verbose flag and using the negotiate authentication mechanism
        - You got a clear output about how kerberos works using the TGT and generating a TGS
        - nginx gives a success response
        - you can examine the ticket again using klist ( service prinicapl added )
        - destroy the ticket using kdestroy command
        - curl again
        - you got an authorization required response (401 status code)
        
# Execution
you can find an execution video [![here]()](https://www.youtube.com/watch?v=WtHJgz-sNJ4).
        
   
   
   
          
