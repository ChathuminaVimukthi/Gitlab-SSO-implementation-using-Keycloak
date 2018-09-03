## SSO Configuration 

GitLab can be configured to act as a SAML 2.0 Service Provider (SP). This allows GitLab to consume assertions from a SAML 2.0 Identity Provider (IdP) to authenticate users.

For this SSO implementation, Gitlab omnibus package is used. But the source package can be used as well. The configuration for the source packge is available on https://docs.gitlab.com/ee/integration/saml.html .

1. After installing Gitlab go to /home/git/gitlab
    ```sh 
     cd /home/git/gitlab 
    ```
2. Then open the configuration file in an editor and do not close the editor till the end of the configuration
    ```sh 
     sudo vi config/gitlab.yml
    ```
3. Change the value of the `host` and `port` to the address which Gitlab starts on. The default should be
    ```sh 
     host: localhost
     port: 80
    ```
4. Install Keycloak using [https://www.keycloak.org/docs/latest/getting_started/index.html]

5. After installing Keycloak, now create a client in Keycloak by login to the Keycloak admin console which can be found on http://localhost:8180 by default.First,
    - Create a realm 
    - Inside the realm console create a client and give the client id as http://localhost:3000 which is the Gitlab host address.
    - Client protocol should be `saml`.
    - Leave client end-point as blank and save it.
6. Now create the mappers for the authentication.Mappers, as the name may suggest, allow you to map user information to parameters in the SAML 2.0 request for GitLab. An example would be to map the Username into the request for GitLab.
    - Click on the client id in the client list.
    - Go to `Mappers` tab in Client console and click create.
    - Then create the mappers as bellow.
        - Name: username
            - Mapper Type: User Property
            - Property: Username
            - Friendly Name: Username
            - SAML Attribute Name: username
            - SAML Attribute NameFormat: Basic
        - Name: email
            - Mapper Type: User Property
            - Property: Email
            - Friendly Name: Email
            - SAML Attribute Name: email
            - SAML Attribute NameFormat: Basic
        - Name: first_name
            - Mapper Type: User Property
            - Property: FirstName
            - Friendly Name: First Name
            - SAML Attribute Name: first_name
            - SAML Attribute NameFormat: Basic
        - Name: last_name
            - Mapper Type: User Property
            - Property: LastName
            - Friendly Name: Last Name
            - SAML Attribute Name: name
            - SAML Attribute NameFormat: Basic

7. Now get the `dsig:X509Certificate` for the created client. To get the certificate,
    - Go to `Installation` tab in Client console.
    - Choose the format as `SAML Metadata IDPSSODescriptor` and the xml file will appear with a download button.
    - Inside the xml find the tag `<dsig:X509Certificate>` and extract the content inside the tags. This `certificate id` is used in Gitlab config file.
   
8. Again in the Gitlab configuration file opened in the editor, copy the bellow snippet to the config file. There in the `idp_cert` replace only the `X509Certificate` with the copied `certificate id` in the previous step (do not exclude the `\n\n`).
    - `assertion_consumer_service_url` should be your Github host url. (http://github.host.url/users/auth/saml/callback)
    - `idp_sso_target_url` should be your Keycloak host url.And replace the realmName with the realm you created in the previous steps.  (http://keycloak.host.url/auth/realms/realmName/protocol/saml)
    - `issuer` and the Keycloak client Id must be identical.
    
    ```sh 
         omniauth:
           enabled: true
           allow_single_sign_on: ["saml"]
           block_auto_created_users: false
           auto_link_saml_user: true
           providers:
               - {
                   name: 'saml',
                   args: {
                          assertion_consumer_service_url: 'https://gitlab.example.com/users/auth/saml/callback',
                          idp_cert: "-----BEGIN CERTIFICATE-----\nX509Certificate\n-----END CERTIFICATE-----\n",
                          idp_sso_target_url: 'https://login.example.com/idp',
                          issuer: 'https://gitlab.example.com',
                          name_identifier_format: 'urn:oasis:names:tc:SAML:2.0:nameid-format:persistent',
                          attribute_statements: { username: ['username']}
                        },
                   label: 'Company Login' # optional label for SAML login button, defaults to "Saml"
                 }
        ```
9. Save the `gitlab.yml` file and close it.
10. Restart the gitlab service.