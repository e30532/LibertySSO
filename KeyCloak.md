Here is the simplest step to configure keycloak as an identity manager for liberty's openid connector.


## Setting up Keycloak

1. Provision OCP cluster. (I used 4.15.3)
2. create a secret

```
[root@c81981v1 keycloak]# pwd
/root/keycloak
[root@c81981v1 keycloak]# oc login --token=sha256~****** --server=https://api.**.**.**.ibm.com:6443
[root@c81981v1 keycloak]# oc new-project e30532
[root@c81981v1 keycloak]# openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out certificate.pem
Generating a RSA private key
.......................................................+++++
.....+++++
writing new private key to 'key.pem'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:US
State or Province Name (full name) []:
Locality Name (eg, city) [Default City]:
Organization Name (eg, company) [Default Company Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:example-keycloak.apps.**.**.**.ibm.com
Email Address []:
[root@c81981v1 keycloak]# ls
certificate.pem  key.pem
[root@c81981v1 keycloak]# oc create secret tls my-tls-secret --cert certificate.pem --key key.pem
secret/my-tls-secret created
[root@c81981v1 keycloak]# 
```

3. Install keycloak operator. 
<img width="1201" alt="image" src="https://media.github.ibm.com/user/24674/files/9bddbd1b-4e7c-47f8-a978-76ff9ea0166f">. 

4. Create a keycloak instance.  
<img width="835" alt="image" src="https://media.github.ibm.com/user/24674/files/0d228b61-7dcc-4605-8388-9e01cf3a0a1e">
<img width="707" alt="image" src="https://media.github.ibm.com/user/24674/files/b9185f1f-682a-49c5-bc27-c40c629614c5">

5. check the status
```
[root@c81981v1 keycloak]# oc get all
Warning: apps.openshift.io/v1 DeploymentConfig is deprecated in v4.14+, unavailable in v4.10000+
NAME                                    READY   STATUS    RESTARTS   AGE
pod/example-keycloak-0                  1/1     Running   0          66s
pod/keycloak-operator-dc5579989-xtksx   1/1     Running   0          8m3s

NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/example-keycloak-discovery   ClusterIP   None             <none>        7800/TCP   66s
service/example-keycloak-service     ClusterIP   172.30.169.164   <none>        8443/TCP   66s

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/keycloak-operator   1/1     1            1           8m4s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/keycloak-operator-dc5579989   1         1         1       8m4s

NAME                                READY   AGE
statefulset.apps/example-keycloak   1/1     67s

NAME                                                      HOST/PORT                                         PATH   SERVICES                   PORT    TERMINATION            WILDCARD
route.route.openshift.io/example-keycloak-ingress-mcdxm   example-keycloak.apps.***.**.**.ibm.com          example-keycloak-service   https   passthrough/Redirect   None
[root@c81981v1 keycloak]# 
```

6. login to the keycloak console

You can confirm the credential with the command below.   
```
oc get secret example-keycloak-initial-admin -o jsonpath='{.data.username}' | base64 --decode
oc get secret example-keycloak-initial-admin -o jsonpath='{.data.password}' | base64 --decode
```


## OpenID connect
7. create an Identify provider.
<img width="1264" alt="image" src="https://media.github.ibm.com/user/24674/files/f130339a-18d8-4031-a7b3-00b8e7a2851e">
toggle off "Use discovery endpoint".  
<img width="1001" alt="image" src="https://media.github.ibm.com/user/24674/files/b9e31ddf-6ddf-4937-bfe6-0593e00115b2">
<img width="1016" alt="image" src="https://media.github.ibm.com/user/24674/files/3a580739-b801-439b-be1a-514531160f4d">

8. create a client.
<img width="948" alt="image" src="https://media.github.ibm.com/user/24674/files/d16c6977-5052-4a15-a7f2-bd53f204a7d2">
<img width="981" alt="image" src="https://media.github.ibm.com/user/24674/files/f15683e2-650c-42a4-a0c5-0874e724c830">
<img width="998" alt="image" src="https://media.github.ibm.com/user/24674/files/c9a4dd5e-851a-4f69-8932-8e469f82b16b">


9. create a user.
<img width="1143" alt="image" src="https://media.github.ibm.com/user/24674/files/2c8044c3-4259-4711-b651-fb04eba1c3a2">
<img width="817" alt="image" src="https://media.github.ibm.com/user/24674/files/7bf43a7b-dfdd-46fd-80d7-ddded9a15b6d">
<img width="843" alt="image" src="https://media.github.ibm.com/user/24674/files/2a2d0d83-87e3-413a-979a-2d90102688b4">
<img width="983" alt="image" src="https://media.github.ibm.com/user/24674/files/b6bfaea4-02d9-4bd8-aa5a-643face35174">

10. configure a liberty server
```
<?xml version="1.0" encoding="UTF-8"?>
<server description="new server">

    <!-- Enable features -->
    <featureManager>
        <feature>jsp-2.3</feature>
        <feature>openidConnectClient-1.0</feature>
        <feature>transportSecurity-1.0</feature>
    </featureManager>

    <!-- To access this server from a remote client add a host attribute to the following element, e.g. host="*" -->
    <httpEndpoint id="defaultHttpEndpoint"
                  host="*"
                  httpPort="9080"
                  httpsPort="9443" />

    <keyStore id="defaultKeyStore" password="WebASWebAS"/>    
    <openidConnectClient id="client01"
	clientId="client01"
    	clientSecret="client01"
        signatureAlgorithm="RS256"
        jwkEndpointUrl="https://example-keycloak.apps.**.**.**.ibm.com/realms/master/protocol/openid-connect/certs"
        issuerIdentifier="https://example-keycloak.apps.**.**.**.ibm.com/realms/master"
    	authorizationEndpointUrl="https://example-keycloak.apps.**.**.**.ibm.com/realms/master/protocol/openid-connect/auth"
    	tokenEndpointUrl="https://example-keycloak.apps.**.**.**.ibm.com/realms/master/protocol/openid-connect/token">
    </openidConnectClient>
                  
    <!-- Automatically expand WAR files and EAR files -->
    <applicationManager autoExpand="true"/>


</server>

```

11. add the keycloak certificate to the liberty's keystore and restart the liberty.
```
/opt/IBM/WebSphere/AppServer90ND/java/8.0/bin/keytool -importcert -keystore ../usr/servers/keycloak/resources/security/key.p12 -storepass WebASWebAS -file /root/keycloak/certificate.pem
```

12. deploy a secure application on liberty
```
[root@c81981v1 bin]# cat ../usr/servers/keycloak/apps/expanded/SimpleSecure.ear/META-INF/ibm-application-bnd.xml 
<?xml version="1.0" encoding="UTF-8"?>
<application-bnd
	xmlns="http://websphere.ibm.com/xml/ns/javaee"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://websphere.ibm.com/xml/ns/javaee http://websphere.ibm.com/xml/ns/javaee/ibm-application-bnd_1_2.xsd"
	version="1.2">

	<security-role name="MyRole">
		<special-subject type="ALL_AUTHENTICATED_USERS" />
	</security-role>
</application-bnd>
[root@c81981v1 bin]# 
```

12. access the application
https://***.***.ibm.com:9443/SimpleSecureWeb/SimpleServlet

The request is redirected to the ID provider's login page.
<img width="876" alt="image" src="https://media.github.ibm.com/user/24674/files/6dc812bf-dda6-42b6-b9d0-a622effecf57">
<img width="649" alt="image" src="https://media.github.ibm.com/user/24674/files/91ff8736-c15c-4809-a6c2-a1e8ac7493d6">
<img width="524" alt="image" src="https://media.github.ibm.com/user/24674/files/441391ac-80d4-4578-bb2a-e74e32e1e958">


https://jwt.io/

<img width="1202" alt="image" src="https://media.github.ibm.com/user/24674/files/b5206f64-a4df-4951-9bb0-213ddef75313">


## SAML

<img width="1285" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/d4b64abf-2f73-4d33-a8ae-376b9c269faa">  
<img width="1087" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/beb7287b-267b-4562-83af-179a51aa28c8">

```
[root@c81981v1 bin]# cat ../usr/servers/keycloak/server.xml 
<?xml version="1.0" encoding="UTF-8"?>
<server description="new server">

    <!-- Enable features -->
    <featureManager>
        <feature>jsp-2.3</feature>
        <feature>javaee-7.0</feature>
        <feature>samlWeb-2.0</feature>
    </featureManager>

    <!-- To access this server from a remote client add a host attribute to the following element, e.g. host="*" -->
    <httpEndpoint id="defaultHttpEndpoint"
                  host="*"
                  httpPort="9080"
                  httpsPort="9443" />

    <keyStore id="defaultKeyStore" password="WebASWebAS"/>    
    <quickStartSecurity userName="admin" userPassword="admin" />                  
    <!-- Automatically expand WAR files and EAR files -->
    <applicationManager autoExpand="true"/>
    <samlWebSso20 disableLtpaCookie="false"/>
    <httpSession useContextRootAsCookiePath="true"/>
    <pluginConfiguration pluginInstallRoot="/opt/IBM/WebSphere/Plugins90"/>
</server>
```
<img width="725" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/e4b455ba-354b-4d47-b0a8-c92acb41b505">
<img width="1008" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/040b4a74-7456-45a8-b681-4334fc5d1915">
<img width="731" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/3fc71411-3a90-4c95-84e8-ca86cae4d564">
<img width="900" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/190cb7e1-b637-4dbb-bfd0-80fa31bee082">
<img width="871" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/925ed50e-efee-4236-a6d8-0ded27f314bb">
<img width="911" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/be1aa1dc-0b86-4ba9-afac-d248f6031e33">
<img width="702" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/5c4d18c4-dcfb-4032-a9c2-0e44cb1d2130">
<img width="1079" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/9e05f86d-7d8a-45af-8420-7d6e18002a81">
<img width="562" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/e34f4367-1b2c-4f1a-a3e2-ff6866222781">
<img width="1122" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/626a7283-b129-42ad-9c7e-84a17635b4d9">
<img width="998" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/c15bde16-4519-4395-b6d5-70cea6349616">
<img width="862" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/4c9f49ce-caf7-4241-9e26-8c8461814b3a">
<img width="921" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/23192aa7-b088-452c-9d95-849eeff7f606">
```
[root@c81981v1 bin]# vi ../usr/servers/keycloak/resources/security/idpMetadata.xml 
[root@c81981v1 bin]# cat ../usr/servers/keycloak/resources/security/idpMetadata.xml
<md:EntityDescriptor xmlns="urn:oasis:names:tc:SAML:2.0:metadata" xmlns:md="urn:
```
<img width="885" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/844614e3-9e35-4cdb-998d-1dde175df3f9">
<img width="775" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/ba1562d3-9830-46ce-afd3-9af21cf730bc">

https://samltool.io/
<img width="443" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/c7b7a05a-5687-4c18-a2a0-81433aa3a6bb">



Optional:
If you would like to
