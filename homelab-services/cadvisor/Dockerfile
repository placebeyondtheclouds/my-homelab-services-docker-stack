FROM gcr.io/cadvisor/cadvisor:v0.49.1

ARG CADVISOR_USERNAME
ARG CADVISOR_PASSWORD_HASH

# Create auth file
RUN echo "${CADVISOR_USERNAME}:${CADVISOR_PASSWORD_HASH}" >/auth

# Set proper permissions
RUN chmod 644 /auth

ENTRYPOINT ["/usr/bin/cadvisor", "-logtostderr", "--http_auth_file=/auth", "--http_auth_realm=localhost"]
