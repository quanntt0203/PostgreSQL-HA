FROM ubuntu:24.04

# 1. Install pgBackRest and SSH server
RUN apt-get update && apt-get install -y \
    pgbackrest \
    openssh-server \
    sudo \
    tzdata \
    && rm -rf /var/lib/apt/lists/*

# 2. Create the pgbackrest user and directories
RUN useradd -m -d /home/pgbackrest -s /bin/bash pgbackrest \
    && mkdir -p /var/lib/pgbackrest /var/log/pgbackrest /etc/pgbackrest /run/sshd \
    && chown pgbackrest:pgbackrest /var/lib/pgbackrest /var/log/pgbackrest /etc/pgbackrest

# 3. Configure SSH for passwordless access
# We'll mount the keys later, but we need to ensure permissions are ready
RUN mkdir -p /home/pgbackrest/.ssh && chmod 700 /home/pgbackrest/.ssh \
    && chown pgbackrest:pgbackrest /home/pgbackrest/.ssh

# 4. Set the default configuration location
ENV PGBACKREST_CONFIG=/etc/pgbackrest/pgbackrest.conf

# Switch to pgbackrest user
USER pgbackrest

# 5. Expose SSH port
EXPOSE 22

# 6. Start SSH service in the foreground
CMD ["/usr/sbin/sshd", "-D"]