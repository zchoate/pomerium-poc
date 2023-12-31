# Using Pomerium as an authentication proxy with Azure AD with Pomerium issued/signed JWTs

This guide is a quick demonstration of getting Pomerium working with Azure AD.

## Pomerium Overview and Limitations
[Pomerium](https://www.pomerium.com/) is an authentication proxy. Originally you could use Traefik or another reverse proxy and forward authentication requests to Pomerium. This functionality is now deprecated and will be [no longer supported in v0.21.0 and on](https://0-20-0.docs.pomerium.com/docs/reference/forward-auth).

Pomerium is appropriate for securing applications with no authentication or supports those that can be configured to handle authentication via headers or JWTs. There's a [guide on the Pomerium site for the setup of Cockpit](https://0-20-0.docs.pomerium.com/docs/guides/cockpit) but as noted there, Cockpit uses PAM for authentication. Pomerium in that situation sits in front of Cockpit but once you authenticate to your IDP and as a result that route in Pomerium, you'll still need to login normally with the Cockpit page. This is still useful in eliminating a VPN as a requirement for accessing Cockpit.

Pomerium also notes that it can be used to secure some other TCP services outside of HTTP/S. These seem to be a better fit for something like [Teleport](https://goteleport.com/) which also has a OSS version. Teleport's OSS solution doesn't support integrating with your IdP unless you spring for the Enterprise tier.

## Assumptions
- Publicly accessible machine with Docker and Docker Compose installed
- Machine is reachable with 2 addresses (auth.whatever.tld and verify.whatever.tld)
- Access to an Azure AD tenant to create an app registration

## Steps
1. Clone this repository to the server: `git clone https://github.com/zchoate/pomerium-poc.git`
2. Create the prerequisite directories within the directory newly created by git:
    ```bash
    cd pomerium-poc
    mkdir -p pomerium/data/jwks pomerium/data/le
    ```
    These directories will store the JWT signing key that we'll create in the next step and the LetsEncrypt certificates, respectively.
3. Create the key that will be used to sign the JWT tokens:
    ```bash
    openssl genrsa -out ./pomerium/data/jwks/jwks.pem 4096
    ```
    This will create an RSA private key that will be used to sign the JWTs. Protect this as it is a private key.
4. Create a copy of `pomerium.env.example` as `pomerium.env`.
5. Create a cookie secret that is at least 32 characters long using this command:
    ```bash
    head -c32 /dev/urandom | base64
    ```
    Update the `pomerium.env` file with this `COOKIE_SECRET`. We'll come back to `pomerium.env` to update it with additional values.
6. In the meantime, create a file at `./pomerium/config.yaml`. This will store our routes and policies for Pomerium.
    ```yaml
    routes:
      - from: https://verify.<replace with domain>
        to: http://verify:8000
        policy:
          - allow:
              and:
                - claim/groups: '<replace with group guid>'
        pass_identity_headers: true
    ```
    If you don't have the group GUID, that's fine, just come back and edit once you've got that setup.
7. Navigate to Azure AD and create a new App Registration.
8. Go through the *Register an application* wizard.
    - Create a name for the application (this is primarily for your own identification).
    - Maintain the defautl of `Accounts in this organizational directory only` for who can use the application.
    - Choose *Web* for the *Redirect URI* and input the Pomerium redirect URL: `https://auth.<replace with domain>/oauth2/callback`
    </br></br>![Register an Application Wizard](./assets/8/register-an-application.png)
9. Note the *Application (client) ID* and *Directory (tenant) ID*. These will be used in `pomerium.env` for `IDP_CLIENT_ID` and replace the section *"replacewithtenantid"* in the `IDP_PROVIDER_URL`. </br></br>![App Registration Details](./assets/9/app-registration-details.png)
10. Create a *New client secret* under *Certificates & secrets*. Note this and update the associated `IDP_CLIENT_SECRET` value in `pomerium.env`.
11. Navigate to API Permissions and add the following permissions beyond the default `User.Read`: 
    - Delegated > OpenID > email
    - Delegated > OpenID > offline_access
    - Delegated > OpenID > openid
    - Delegated > OpenID > profile
    </br></br>
    ![API Permissions](./assets/11/api-permissions.png)
12. We'll enable groups to be transmitted with the claim. To keep it simple we'll use the default group object ID, but you have additional options on how to transmit the group data such as sAMAccountName. Navigate to *Token configuration* and select *Add groups claim* under *Optional claims*. </br></br>![Optional Claims > Groups Claims](./assets/12/groups-claims.png)
13. At this point, the only items left to update in `pomerium.env` are the `AUTHENTICATE_SERVICE_URL` and `AUTOCERT_EMAIL`. The `AUTHENTICATE_SERVICE_URL` will be the first part of the redirect URL described in step 8 without `/oauth2/callback` (just `https://auth.<replace with domain>`). The `AUTOCERT_EMAIL` should be an email address that you'd like to receive notifications regarding certificate expiry on the auth.x and verify.x URLs. Pomerium will handle renewal via ACME as long as these URLs are accessible from the public internet.
14. Review the `docker-compose.yaml` if you haven't already. Once done, you should be ready to start the Docker Compose environment.
15. Navigate to the URL used for the Verify service (`https://verify.<replace with domain>`). You should be promptly directed to the Microsoft login page.
16. Login and verify the account access that Pomerium is requesting is appropriate.
17. On successful login, you'll be redirected to the Verify service. The page should look similar to the example below. That's all the configuration required on the Pomerium side. </br></br>![Pomerium Verify Service page](./assets/17/verify-service.png)