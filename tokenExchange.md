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
- In this application choose…
  -	Application APIs -> Client Authentication
  -	Add new Secrets for with OpenID API Access
  -	Set the client id and secret as export parameters as well as the url to your Cloud Identity tenant
    
```
export IAS_TENANT='<your cloud identity tenant>'
export client_id='<your client_id>'
export client_secret='<your client_secret>'
export IAS_ENCODED_CREDENTIALS=$(echo -n "$client_id:$client_secret" | base64)
```

### Create Process Integration Runtime Instance
In BTP create a Process Integration Runtime service instance with the grant type **urn:ietf:params:oauth:grant-type:jwt-bearer**
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
export PIR_CLIENTID='<your okta id_token>'
export PIR_CLIENTSECRET='<your okta id_token>'
export PIR_TOKENURL='<your okta id_token>'
```

### Obtain **id_token** from Okta and set as parameter
This can be acheived either by using [oidc-sample-app](https://github.com/jcawley5/oidc-sample-app) or by following the [Okta guide](https://support.okta.com/help/s/article/How-to-get-tokens-for-an-OIDC-application-without-a-browser-using-curlPostman?language=en_US)
> **Note**: the application you obtain the token from must be trusted by the Cloud Identity Services

- Set the id_token value as a parameter
```
export ID_TOKEN_OKTA='<your okta id_token>'
```

#### Call Cloud Identity to exchange the token id
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

#### Call XSUAA for a token exchange
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

#### Call the flow
```
curl --location --request POST 'https://<your example flow>/http/ppuser' \
--header 'Authorization: Bearer '$ACCESS_TOKEN_XSUAA