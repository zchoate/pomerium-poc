# Pomerium with JWT and Azure AD

1. Create directory structure:
    - `mkdir -p pomerium/data/jwks pomerium/data/le`
2. Create config.yaml at `./pomerium/config.yaml`
3. Run `openssl ecparam -genkey -name prime256v1 -noout -out ./pomerium/data/jwks/jwks.pem`