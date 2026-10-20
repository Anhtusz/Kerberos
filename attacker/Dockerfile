FROM python:3.12-slim
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y \
      krb5-user smbclient dnsutils && \
    rm -rf /var/lib/apt/lists/* && \
    python -m pip install --no-cache-dir impacket
COPY krb5.conf /etc/krb5.conf
CMD ["sleep", "infinity"]

