# Pomerium with JWT and Azure AD

1. Create directory structure:
    - `mkdir -p pomerium/data/jwks pomerium/data/le`
2. Run `openssl ecparam -genkey -name prime256v1 -noout -out ./pomerium/data/jwks/jwks.pem`
3. Run `head -c32 /dev/urandom | base64` and use the output for the cookie secret.
4. Create config.yaml at `./pomerium/config.yaml`