# Token Exchange Scenario

This demonstrates a user authenticating directly to an IDP. Their id_token is then exchanged first by SAP Cloud Identity then XSUAA which is then used to authenticate to a CPI iflow. In this demo flow Okta was used as the token originator.

## Prerequisites
> admin access is required in all systems
- Okta
- SAP Cloud Identity Services
- BTP
- Cloud Integration Suite


## Steps
### Configure OIDC Trust between Cloud Identity Services and Okta
Follow the steps detailed [here](https://community.sap.com/t5/technology-blogs-by-sap/connect-sap-cloud-identity-authentication-service-as-a-proxy-to-okta-using/ba-p/13554417)

### Configure OIDC Trust between BTP and Cloud Identity Services
Follow the steps detailed [here](https://help.sap.com/docs/btp/sap-business-technology-platform/establishing-trust-automatically)

### Create Application API in Cloud Identity Services
- In this application created in the previous step choose…
  -	Application APIs -> Client Authentication
  -	Add new Secrets for with OpenID API Access
  -	Set the client id and secret as export parameters as well as the url to your Cloud Identity tenant
    
```
export IAS_TENANT='<your cloud identity tenant>'
export IAS_clientid='<your client_id>'
export IAS_clientSecret='<your client_secret>'
export IAS_ENCODED_CREDENTIALS=$(echo -n "$IAS_clientid:$IAS_clientSecret" | base64)
```

### Create Process Integration Runtime Instance
In BTP create a Process Integration Runtime service instance with the grant type **urn:ietf:params:oauth:grant-type:jwt-bearer**
> **Note**: Instead of using ESBMessaging.send a custom role may be preferred
```
{
    "grant-types": [
        "urn:ietf:params:oauth:grant-type:jwt-bearer"
    ],
    "redirect-uris": [],
    "roles": [
        "ESBMessaging.send"
    ]
}
```

Using the values from the service instance set the clientid, clientsecret, tokenurl

```
export PIR_CLIENTID='<your Process Integration Runtime client id>'
export PIR_CLIENTSECRET='<your Process Integration Runtime client secret>'
export PIR_TOKENURL='<your Process Integration Runtime token url>'
```

### Obtain **id_token** from Okta and set as parameter
This can be acheived either by using [oidc-sample-app](https://github.com/jcawley5/oidc-sample-app) or by following the [Okta guide](https://support.okta.com/help/s/article/How-to-get-tokens-for-an-OIDC-application-without-a-browser-using-curlPostman?language=en_US)
> **Note**: the application you obtain the token from must be trusted by the Cloud Identity Services

- Set the id_token value as a parameter
```
export ID_TOKEN_OKTA='<your okta id_token>'
```

### Call Cloud Identity to exchange the token id
```
curl --location $IAS_TENANT'/oauth2/token' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--header 'Authorization: Basic '$IAS_ENCODED_CREDENTIALS \
--data-urlencode 'grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer' \
--data-urlencode 'assertion='$ID_TOKEN_OKTA
```
- Set the id_token value as a parameter

```
export ID_TOKEN_IAS='<your ias id_token>'
```

### Call XSUAA for a token exchange
```
curl --location $PIR_TOKENURL \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'client_id='$PIR_CLIENTID \
--data-urlencode 'client_secret='$PIR_CLIENTSECRET \
--data-urlencode 'assertion='$ID_TOKEN_IAS \
--data-urlencode 'grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer'
```
- Set the access_tokenn value as a parameter

```
export ACCESS_TOKEN_XSUAA='<your XSUAA access_token>'
```

### Call the flow
```
curl --location --request POST 'https://<your example flow>/http/ppuser' \
--header 'Authorization: Bearer '$ACCESS_TOKEN_XSUAA
```

### Use API Management
- Within your CIS tenant, choose Configure -> APIs
- Choose Key Value Maps
- Create a Key Value Map named **exchangeDemo** with the following data
  
|  Name  |  Value|
| -------- | ------- |
|IAS-clientid|The client id value from the step [Create Application API in Cloud Identity Services](#create-application-api-in-cloud-identity-services)|
|IAS-clientSecret|The client secret value from the step [Create Application API in Cloud Identity Services](#create-application-api-in-cloud-identity-services)|
|IAS-tokenurl|The token url value **without https://** from the step [Create Application API in Cloud Identity Services](#create-application-api-in-cloud-identity-services)|
|XSUAA-clientid|The client id value from the step [Process Integration Runtime](#create-process-integration-runtime-instance)|
|XSUAA-clientSecret|The client secret value from the step [Process Integration Runtime](#create-process-integration-runtime-instance)|
|XSUAA-tokenurl|The token url value **without https://** from the step [Process Integration Runtime](#create-process-integration-runtime-instance)|

- Download the [ExchangeDemo API Proxy](../../raw/main/ExchangeDemo.zip) example.
- Choose API Proxies, Import API and choose ExchangeDemo.zip
- Edit the Target EndPoint to point to your desired Integration flow
- Save and deploy
### Call the API Proxy
- Set the url to the proxy
  
```
export API_Proxy_URL='<your API Proxy URL>'
```

- Call the URL
```
curl --location --request POST $API_Proxy_URL \
--header 'assertion: '$ID_TOKEN_OKTA \
--header 'Content-Length: 0'
```
