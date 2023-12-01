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
```
:
:

[[registry]]
location = "default-route-openshift-image-registry.apps.*.*.*.*.com"
insecure = true

:
:
```

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





