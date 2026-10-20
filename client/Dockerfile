FROM debian:bookworm-slim
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y \
      krb5-user smbclient dnsutils ca-certificates && \
    rm -rf /var/lib/apt/lists/*
COPY krb5.conf /etc/krb5.conf
CMD ["sleep", "infinity"]

