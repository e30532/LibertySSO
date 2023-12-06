This document describes an example of setting up SSO for Liberty running on OpenShift with external id provider(github enterprise).

1. refer to step 1 described in https://github.com/e30532/RHDG/blob/main/README.md and provision OCP 4.13 cluster.   

2. create a simple secure application. 
In this scenario, I used the following application which contains a secured servlet.   
https://github.com/e30532/SimpleApps/blob/master/SimpleSecureWeb/src/simple/secure/SimpleServlet.java.  

3. build and publish a image.

Dockerfile
```
FROM icr.io/appcafe/open-liberty
COPY --chown=1001:0  SimpleSecure.ear /config/apps/SimpleSecure.ear
COPY --chown=1001:0  server.xml /config/server.xml
ARG VERBOSE=true
RUN configure.sh
```

server.xml
```
<server description="new server">

    <!-- Enable features -->
    <featureManager>
        <feature>localConnector-1.0</feature>
        <feature>javaee-7.0</feature>
    </featureManager>

    <!-- To access this server from a remote client add a host attribute to the following element, e.g. host="*" -->
    <httpEndpoint host="*" httpPort="9080" httpsPort="9443" id="defaultHttpEndpoint"/>
                  
    <!-- Automatically expand WAR files and EAR files -->
    <applicationManager autoExpand="true"/>

    <basicRegistry id="basic" realm="WebRealm">
        <user name="admin" password="admin"/> 
    </basicRegistry>


    <application location="SimpleSecure.ear">
        <application-bnd>
           <security-role id="MyRole" name="MyRole">
              <user id="admin" name="admin"></user>
           </security-role>
        </application-bnd>
    </application>
</server>
```

```
# podman build -t simplesecure .

# oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge

# oc get route -n openshift-image-registry

# oc new-project defaulte

# podman tag simplesecure:latest 'default-route-openshift-image-registry.apps.*.*.*.*.com/defaulte/simplesecure'

# vi /etc/containers/registries.conf
:
:

[[registry]]
location = "default-route-openshift-image-registry.apps.*.*.*.*.com"
insecure = true

:
:

# podman login -u kubeadmin -p $(oc whoami -t) https://default-route-openshift-image-registry.apps.*.*.*.*.com

# podman push default-route-openshift-image-registry.apps.*.*.*.*.com/defaulte/simplesecure
```

4. deploy the application without using liberty operator.

```
# oc get is | grep simple
simplesecure   default-route-openshift-image-registry.apps.*.*.*.*.com/defaulte/simplesecure   latest   6 minutes ago

# oc new-app -i simplesecure

# oc create route passthrough simplesecure --service=simplesecure --port=9443
```

Now you will be able to access the application with the credential defined in server.xml. 
https://simplesecure-defaulte.apps.*.*.*.*.com/SimpleSecureWeb/SimpleServlet



The following steps describes how to use github enterprise as an identity provider.   

5. create a github client id and secret which are used by the liberty container image.   
<img width="1111" alt="image" src="https://media.github.ibm.com/user/24674/files/e3999f3b-e56a-4871-84bc-b8ea0a152ab1">.  
<img width="780" alt="image" src="https://media.github.ibm.com/user/24674/files/3ee8a176-8d79-4ca8-ae31-36dcf3d28e32">

Homepage URL:   
https://simplesecure-defaulte.apps.*.*.*.*.com/SimpleSecureWeb/SimpleServlet.  

Authorization callback URL:   
https://simplesecure-defaulte.apps.*.*.*.*.com/ibm/api/social-login/redirect/githubLogin.  

<img width="695" alt="image" src="https://media.github.ibm.com/user/24674/files/0ebc3fe4-3f42-4fbd-b661-a64751198d62">
<img width="740" alt="image" src="https://media.github.ibm.com/user/24674/files/acd3b0fc-b37c-408d-bdc3-2343d16b94bf">

6. rebuild the container image with SSO specific settings.

```
<server description="new server">

    <!-- Enable features -->
    <featureManager>
        <feature>jsp-2.3</feature>
        <feature>localConnector-1.0</feature>
        <feature>javaee-7.0</feature>
    </featureManager>

    <!-- To access this server from a remote client add a host attribute to the following element, e.g. host="*" -->
    <httpEndpoint host="*" httpPort="9080" httpsPort="9443" id="defaultHttpEndpoint"/>
                  
    <!-- Automatically expand WAR files and EAR files -->
    <applicationManager autoExpand="true"/>

    <basicRegistry id="basic" realm="WebRealm">
        <user name="admin" password="admin"/> 
    </basicRegistry>


    <application location="SimpleSecure.ear">
        <application-bnd>
            <security-role id="MyRole" name="MyRole">
                <special-subject type="ALL_AUTHENTICATED_USERS"></special-subject>
            </security-role>
        </application-bnd>
    </application>
</server>
```
I updated special-subject with ALL_AUTHENTICATED_USERS so that the github user can access the application. 


And I also added SSO specific settings in Containerfile.
```
FROM icr.io/appcafe/open-liberty
COPY --chown=1001:0  SimpleSecure.ear /config/apps/SimpleSecure.ear
COPY --chown=1001:0  server.xml /config/server.xml
ARG VERBOSE=true
ARG SEC_SSO_PROVIDERS="github"
ARG TLS=true
ENV SEC_SSO_GITHUB_CLIENTID="b**************a"
ENV SEC_SSO_GITHUB_CLIENTSECRET="0*********************f"
ENV SEC_SSO_GITHUB_HOSTNAME="github.ibm.com"
ENV SEC_TLS_TRUSTDEFAULTCERTS=true
ENV SEC_IMPORT_K8S_CERTS=true

RUN configure.sh
```


```
# podman build -t simplesecure .

# podman tag simplesecure:latest 'default-route-openshift-image-registry.apps.*.*.*.*.com/defaulte/simplesecure'

# podman push default-route-openshift-image-registry.apps.*.*.*.*.com/defaulte/simplesecure
```


7. access the application.
Now the request will be redirected to the github enterprise login page. After the login, you will be asked if you allow the oauth app to link to your account at the first login attempt.   
<img width="620" alt="image" src="https://media.github.ibm.com/user/24674/files/7edfe633-dc58-487b-989f-f43ae6cce945">.  
<img width="767" alt="image" src="https://media.github.ibm.com/user/24674/files/56ba3a6e-6dff-4e4d-8a5a-9ba858b23771">.  


Now you can add SSO with the external id provider WITHOUT any application change.   



Lastly, I'll show you how we can separate SSO specific setting from your image by using Liberty Operator.   


With the setting "SEC_SSO_PROVIDERS" and "TLS" in the container file, the image will become ready for SSO. To be more precise, SSO specific applications like "Social Login" will also be running on the container. 
```
FROM icr.io/appcafe/open-liberty
COPY --chown=1001:0  SimpleSecure.ear /config/apps/SimpleSecure.ear
COPY --chown=1001:0  server.xml /config/server.xml
ARG VERBOSE=true
ARG SEC_SSO_PROVIDERS="github"
ARG TLS=true
```


```
[12/6/23 14:27:07:128 UTC] 0000002c com.ibm.ws.security.social.internal.SocialLoginServiceImpl   I CWWKS5400I: The social login configuration [SocialLoginService] was successfully processed.
[12/6/23 14:27:07:133 UTC] 0000002c com.ibm.ws.security.social.web.EndpointServices              I CWWKS5407I: The Social Login Version 1.0 endpoint service is activated.
[12/6/23 14:27:07:135 UTC] 0000002c com.ibm.ws.security.social.internal.Oauth2LoginConfigImpl    I CWWKS5400I: The social login configuration [githubLogin] was successfully processed.
[12/6/23 14:27:07:153 UTC] 0000002c com.ibm.ws.security.oauth20.web.OAuth20EndpointServices      I CWWKS1410I: The OAuth endpoint service is activated.

[12/6/23 14:27:07:626 UTC] 00000065 com.ibm.ws.webcontainer.osgi.webapp.WebGroup                 I SRVE0169I: Loading Web Module: com.ibm.ws.security.jwt.
[12/6/23 14:27:07:627 UTC] 00000066 com.ibm.ws.webcontainer.osgi.webapp.WebGroup                 I SRVE0169I: Loading Web Module: Social Login 1.0 Endpoint Servlet.
[12/6/23 14:27:07:627 UTC] 00000067 com.ibm.ws.webcontainer.osgi.webapp.WebGroup                 I SRVE0169I: Loading Web Module: com.ibm.oauth.test.war.
[12/6/23 14:27:07:628 UTC] 00000065 com.ibm.ws.webcontainer                                      I SRVE0250I: Web Module com.ibm.ws.security.jwt has been bound to default_host.
[12/6/23 14:27:07:628 UTC] 00000066 com.ibm.ws.webcontainer                                      I SRVE0250I: Web Module Social Login 1.0 Endpoint Servlet has been bound to default_host.
[12/6/23 14:27:07:628 UTC] 00000067 com.ibm.ws.webcontainer                                      I SRVE0250I: Web Module com.ibm.oauth.test.war has been bound to default_host.
[12/6/23 14:27:07:628 UTC] 00000065 com.ibm.ws.http.internal.VirtualHostImpl                     A CWWKT0016I: Web application available (default_host): http://simplesecure-6f985f75f9-rxkbz:9080/jwt/
[12/6/23 14:27:07:629 UTC] 00000066 com.ibm.ws.http.internal.VirtualHostImpl                     A CWWKT0016I: Web application available (default_host): http://simplesecure-6f985f75f9-rxkbz:9080/ibm/api/social-login/
[12/6/23 14:27:07:630 UTC] 00000067 com.ibm.ws.http.internal.VirtualHostImpl                     A CWWKT0016I: Web application available (default_host): http://simplesecure-6f985f75f9-rxkbz:9080/oauth2/

[12/6/23 14:27:07:784 UTC] 00000068 com.ibm.ws.webcontainer.osgi.webapp.WebGroup                 I SRVE0169I: Loading Web Module: SimpleSecureWeb.
[12/6/23 14:27:07:785 UTC] 00000068 com.ibm.ws.webcontainer                                      I SRVE0250I: Web Module SimpleSecureWeb has been bound to default_host.
[12/6/23 14:27:07:785 UTC] 00000068 com.ibm.ws.http.internal.VirtualHostImpl                     A CWWKT0016I: Web application available (default_host): http://simplesecure-6f985f75f9-rxkbz:9080/SimpleSecureWeb/

[12/6/23 14:27:08:045 UTC] 00000039 com.ibm.ws.kernel.feature.internal.FeatureManager            A CWWKF0012I: The server installed the following features: [... socialLogin-1.0, ....].
```

For SEC_SSO_GITHUB_CLIENTID and SEC_SSO_GITHUB_CLIENTSECRET, we can specify this as a secret. The secret name should be <application_name>-olapp-sso. If you are using WebSphere operator, the name will be <application_name>-wlapp-sso. 

```
oc create secret generic simplesecure-olapp-sso --from-literal=github-clientId=b*********a --from-literal=github-clientSecret=0*****f

```

If you would like to specify other environment variables, you can specify them as spec.env of LibertyApplication CRD.

```
:
:
kind: OpenLibertyApplication
metadata:
  name: simplesecure
  namespace: defaulte
spec:
  applicationImage: default-route-openshift-image-registry.apps.*.*.*.*.com/defaulte/simplesecure
  env:
    - name: SEC_TLS_TRUSTDEFAULTCERTS
      value: "true"
    - name: SEC_IMPORT_K8S_CERTS
      value: "true"
 :
 :
```

Some of them might be configurable graphically through Operator.

<img width="1008" alt="image" src="https://media.github.ibm.com/user/24674/files/41aa4f65-147e-4d21-9021-195b2640e29e">




