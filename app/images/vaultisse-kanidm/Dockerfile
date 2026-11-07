# Derived image: kanidm advertises ES256 as its only id-token signing alg,
# while the bundled openid-client (v5) defaults to verifying RS256
# (id_token_signed_response_alg, no env override exists). Patch the default
# alg so the OIDC callback validates kanidm's ES256 tokens.
FROM ghcr.io/albertamat/vaultisse:1.3.8

RUN sed -i "s/id_token_signed_response_alg: 'RS256',/id_token_signed_response_alg: 'ES256',/" \
    /app/node_modules/openid-client/lib/client.js \
    && grep -q "id_token_signed_response_alg: 'ES256'" \
    /app/node_modules/openid-client/lib/client.js
