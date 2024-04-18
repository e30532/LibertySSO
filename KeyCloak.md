Here is the simplest step to configure keycloak as an identity manager for liberty's openid connector and SAML2.0.


# Setting up Keycloak

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
<img width="400" alt="image" src="https://media.github.ibm.com/user/24674/files/9bddbd1b-4e7c-47f8-a978-76ff9ea0166f">.  

4. Create a keycloak instance.  
<img width="400" alt="image" src="https://media.github.ibm.com/user/24674/files/0d228b61-7dcc-4605-8388-9e01cf3a0a1e">
<img width="400" alt="image" src="https://media.github.ibm.com/user/24674/files/b9185f1f-682a-49c5-bc27-c40c629614c5">

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
<img width="400" alt="image" src="https://media.github.ibm.com/user/24674/files/f130339a-18d8-4031-a7b3-00b8e7a2851e">
toggle off "Use discovery endpoint".  
<img width="400" alt="image" src="https://media.github.ibm.com/user/24674/files/b9e31ddf-6ddf-4937-bfe6-0593e00115b2">
<img width="400" alt="image" src="https://media.github.ibm.com/user/24674/files/3a580739-b801-439b-be1a-514531160f4d">

8. create a client.
<img width="400" alt="image" src="https://media.github.ibm.com/user/24674/files/d16c6977-5052-4a15-a7f2-bd53f204a7d2">
<img width="400" alt="image" src="https://media.github.ibm.com/user/24674/files/f15683e2-650c-42a4-a0c5-0874e724c830">
<img width="400" alt="image" src="https://media.github.ibm.com/user/24674/files/c9a4dd5e-851a-4f69-8932-8e469f82b16b">


9. create a user.
<img width="400" alt="image" src="https://media.github.ibm.com/user/24674/files/2c8044c3-4259-4711-b651-fb04eba1c3a2">
<img width="400" alt="image" src="https://media.github.ibm.com/user/24674/files/7bf43a7b-dfdd-46fd-80d7-ddded9a15b6d">
<img width="400" alt="image" src="https://media.github.ibm.com/user/24674/files/2a2d0d83-87e3-413a-979a-2d90102688b4">
<img width="400" alt="image" src="https://media.github.ibm.com/user/24674/files/b6bfaea4-02d9-4bd8-aa5a-643face35174">

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

13. access the application
https://***.***.ibm.com:9443/SimpleSecureWeb/SimpleServlet

The request is redirected to the ID provider's login page.   
<img width="400" alt="image" src="https://media.github.ibm.com/user/24674/files/6dc812bf-dda6-42b6-b9d0-a622effecf57"><br>
<img width="400" alt="image" src="https://media.github.ibm.com/user/24674/files/91ff8736-c15c-4809-a6c2-a1e8ac7493d6"><br>
<img width="400" alt="image" src="https://media.github.ibm.com/user/24674/files/441391ac-80d4-4578-bb2a-e74e32e1e958"><br>


https://jwt.io/

<img width="400" alt="image" src="https://media.github.ibm.com/user/24674/files/b5206f64-a4df-4951-9bb0-213ddef75313"><br>




# SAML
1. create a SAML id provider.   
<img width="400" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/d4b64abf-2f73-4d33-a8ae-376b9c269faa"><br>

2. uncheck "Use entity descriptor" and specify "Single Sing-On service URL"  
<img width="400" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/beb7287b-267b-4562-83af-179a51aa28c8"><br>

3. enable samlWeb-2.0 feature on liberty and start the server. 
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

4. download service metada.  
<img width="400" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/e4b455ba-354b-4d47-b0a8-c92acb41b505"><br>

5. create a client by using the service metada.  
<img width="400" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/040b4a74-7456-45a8-b681-4334fc5d1915"><br>

<img width="400" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/3fc71411-3a90-4c95-84e8-ca86cae4d564"><br>

6. create a user   
<img width="400" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/190cb7e1-b637-4dbb-bfd0-80fa31bee082"><br>

Note!!! You need to specify a dummy email.   
<img width="400" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/925ed50e-efee-4236-a6d8-0ded27f314bb"> <br> 
<img width="400" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/be1aa1dc-0b86-4ba9-afac-d248f6031e33"> <br> 

7. Link the user with the saml id provider    
<img width="400" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/5c4d18c4-dcfb-4032-a9c2-0e44cb1d2130"> <br>  
<img width="400" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/9e05f86d-7d8a-45af-8420-7d6e18002a81"> <br>  
<img width="400" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/e34f4367-1b2c-4f1a-a3e2-ff6866222781"> <br>  

8. download the id provider's metadata   
<img width="400" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/626a7283-b129-42ad-9c7e-84a17635b4d9"> <br> 
<img width="400" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/c15bde16-4519-4395-b6d5-70cea6349616"> <br> 
<img width="400" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/4c9f49ce-caf7-4241-9e26-8c8461814b3a"> <br>  
<img width="400" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/23192aa7-b088-452c-9d95-849eeff7f606"> <br>  

9. save the id provider's metadata in idpMetadata.xml under liberty server config/resources/security.   
```
[root@c81981v1 bin]# vi ../usr/servers/keycloak/resources/security/idpMetadata.xml 
[root@c81981v1 bin]# cat ../usr/servers/keycloak/resources/security/idpMetadata.xml
<md:EntityDescriptor xmlns="urn:oasis:names:tc:SAML:2.0:metadata" xmlns:md="urn:
```


<img width="400" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/844614e3-9e35-4cdb-998d-1dde175df3f9"><br> 
<img width="400" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/ba1562d3-9830-46ce-afd3-9af21cf730bc"><br>

https://samltool.io/

<img width="400" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/c7b7a05a-5687-4c18-a2a0-81433aa3a6bb"><br>  





Memo (setup for IHS):
In the keyclok console, you need to replace all liberty_hsot:9443 with IHS_host.

./pluginUtility merge --sourcePath=../usr/servers/keycloak/logs/state/plugin-cfg.xml,../usr/servers/keycloak2/logs/state/plugin-cfg.xml --targetPath=/tmp/plugin-cfg.xml

keytool -exportcert -rfc -alias default -file /tmp/ca-keycloak.cert -keystore /opt/IBM/wlp24.0.0.2/wlp/usr/servers/keycloak/resources/security/key.p12
keytool -exportcert -rfc -alias default -file /tmp/ca-keycloak2.cert -keystore /opt/IBM/wlp24.0.0.2/wlp/usr/servers/keycloak2/resources/security/key.p12

/opt/IBM/HTTPServer90/bin/gskcapicmd -cert -add -db /opt/IBM/WebSphere/Plugins90//config/webserver1/plugin-key.kdb -stashed -label ca-keycloak -file /tmp/ca-keycloak.cert
/opt/IBM/HTTPServer90/bin/gskcapicmd -cert -add -db /opt/IBM/WebSphere/Plugins90//config/webserver1/plugin-key.kdb -stashed -label ca-keycloak2 -file /tmp/ca-keycloak2.cert




# SAML for tWAS
You can also use the keycloak for tWAS SAML SSO with a few steps.   

1. deploy "<WAS_ROOT>/installableApps/WebSphereSamlSP.ear". <br>
<img width="891" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/0afe9812-9fa5-4e29-96e2-a05f39189787"><br>

2. configure SAML TAI.
<img width="887" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/5cff7dcd-9b92-4cb9-9c85-c6f509bf12af"><br>
<img width="887" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/e1a8d828-6760-4ae4-88b9-1a23e57d5825"><br>
```
com.ibm.ws.security.web.saml.ACSTrustAssociationInterceptor
```

<img width="719" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/35cd133f-3425-4e97-88db-0c707b97b284"><br>
```
sso_1.sp.acsUrl : https://c8***.com/samlsps/acs
sso_1.sp.idMap :  idAssertion
```
<img width="889" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/83359a88-2302-4b8a-a356-492fc092ffde"><br>
```
com.ibm.websphere.security.DeferTAItoSSO : com.ibm.ws.security.web.saml.ACSTrustAssociationInterceptor
com.ibm.websphere.security.InvokeTAIbeforeSSO : com.ibm.ws.security.web.saml.ACSTrustAssociationInterceptor
```

3. export a service metadata file.
```
[root@c81981v1 bin]# ./wsadmin.sh -user admin -password admin
WASX7209I: Connected to process "dmgr" on node c81981v1CellManager01 using SOAP connector;  The type of process is: DeploymentManager
WASX7031I: For help, enter: "print Help.help()"
wsadmin>AdminTask.exportSAMLSpMetadata('-spMetadataFileName /root/was_sp_metadata.xml -ssoId 1')
u'true'
wsadmin>quit
[root@c81981v1 bin]# cat cat /root/was_sp_metadata.xml
cat: cat: No such file or directory
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ns7:EntityDescriptor xmlns="http://www.w3.org/2005/08/addressing" xmlns:ns2="http://docs.oasis-open.org/wsfed/federation/200706" xmlns:ns3="http://docs.oasis-open.org/wsfed/authorization/200706" xmlns:ns4="http://www.w3.org/2001/04/xmlenc#" xmlns:ns5="http://www.w3.org/2000/09/xmldsig#" xmlns:ns6="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-utility-1.0.xsd" xmlns:ns7="urn:oasis:names:tc:SAML:2.0:metadata" xmlns:ns8="urn:oasis:names:tc:SAML:2.0:assertion" xmlns:ns9="http://docs.oasis-open.org/ws-sx/ws-securitypolicy/200702" entityID="https://c81****.ibm.com/samlsps/acs">
    <ns7:SPSSODescriptor AuthnRequestsSigned="true" WantAssertionsSigned="true" protocolSupportEnumeration="urn:oasis:names:tc:SAML:2.0:protocol">
        <ns7:AssertionConsumerService index="0" Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST" Location="https://c8****.ibm.com/samlsps/acs"/>
    </ns7:SPSSODescriptor>
</ns7:EntityDescriptor>
```

4. import the service metadata as a keycloak client.  
<img width="869" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/cead2d64-b758-4e87-a949-f30920ca8fbd"><br>
Sicne SP initiated SSO is used in this scenario, specify "IDP-Initiated SSO URL name".   
<img width="941" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/11f1d1c3-f8ac-428c-afaa-dce8f74a4d46"><br>

5. create a SAML ID provider
<img width="1197" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/250c6fa9-4fd4-4110-b409-543f962857d5"><br>
uncheck "Use entity descriptor" and specify "Single Sing-On service URL"

6. create a user and associate the user with ID provider.
<img width="1318" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/7893108d-bd01-45e9-b8e3-a29ed5308296"><br>
<img width="1000" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/622d6160-b392-442c-b845-38e9a2c51e97"><br>
<img width="975" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/834c1532-d487-4ce2-b04d-331dcfcac505"><br>

7. download the IDP metadata and import it to tWAS. 
<img width="811" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/86eaa818-55c0-47f7-bb78-36d71a9eff76"><br>
```
[root@c81981v1 bin]# vi /root/idp_metadata.xml
[root@c81981v1 bin]# ./wsadmin.sh -user admin -password admin
WASX7209I: Connected to process "dmgr" on node c81981v1CellManager01 using SOAP connector;  The type of process is: DeploymentManager
WASX7031I: For help, enter: "print Help.help()"
wsadmin>AdminTask.importSAMLIdpMetadata('-idpMetadataFileName /root/idp_metadata.xml -idpId 1 -ssoId 1 -signingCertAlias KC_cert')
u'true'
wsadmin>AdminConfig.save()
u''
wsadmin>quit
```

8. check the configuration. After adding the idp metadata, "sso_1.idp_1.SingleSignOnUrl" will be configured automatically. 
<img width="715" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/015871b3-e5c0-40e6-a6b1-2d0e7e4d3651"><br>
Note: I also configured "sso_1.sp.targetUrl" manually to avoid "INTERNAL ERROR: Please contact your support" error on the browser.  
```
sso_1.sp.targetUrl : https://c8***.ibm.com/SimpleSecureWeb/SimpleServlet 
```
keycloak certificate will be addeded to the trust store automatically.  
<img width="850" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/8f04fc6b-a127-482f-9f4a-474c457e50fb"><br>

9. add the keycloak realm as trusted.
<img width="595" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/98b28808-dbda-4c0a-9d3e-cf3c585cad2d"><br>
```
https://example-keycloak.apps.apps.***.ibm.com/realms/master
```

10. update the application's security role mapping with "All Authenticated in Trusted Realms" so that the user authenticated by keyclaok can acceess the secured application.
<img width="889" alt="image" src="https://github.com/e30532/LibertySSO/assets/22098113/1a739d08-1213-4ae1-b578-d992eae112c9"><br>




