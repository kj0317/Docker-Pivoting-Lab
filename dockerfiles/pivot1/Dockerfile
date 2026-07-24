FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

# ──────────────────────────────────────────────
# 1. System packages
# ──────────────────────────────────────────────
RUN apt-get update && apt-get install -y --no-install-recommends \
    # SSH server (for method 1: SSH tunneling, and sshuttle)
    openssh-server \
    # Web app runtime
    python3 \
    python3-pip \
    # Tools the learner will need after getting a foothold
    curl \
    wget \
    # socat — for relay chaining and sshuttle's Python transport
    socat \
    # Basic utilities (what you'd find on a real server)
    iputils-ping \
    iproute2 \
    net-tools \
    procps \
    && rm -rf /var/lib/apt/lists/*

# ──────────────────────────────────────────────
# 2. Flask for the vulnerable web app
# ──────────────────────────────────────────────
RUN pip3 install --no-cache-dir flask

# ──────────────────────────────────────────────
# 3. SSH configuration
#    - Weak root password (crackable with hydra + our wordlist)
#    - GatewayPorts yes: allows remote port forwards to bind
#      on all interfaces (required for SSH -R pivoting)
#    - PermitRootLogin yes: keeps the lab simple
# ──────────────────────────────────────────────
RUN mkdir -p /var/run/sshd && \
    echo 'root:pivotme' | chpasswd && \
    sed -i 's/#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    sed -i 's/#PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config && \
    echo 'GatewayPorts yes' >> /etc/ssh/sshd_config

# ──────────────────────────────────────────────
# 4. Vulnerable web application
# ──────────────────────────────────────────────
COPY app.py /opt/webapp/app.py

# ──────────────────────────────────────────────
# 5. Startup script — runs both SSH and the web app
# ──────────────────────────────────────────────
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

EXPOSE 22 5000

CMD ["/entrypoint.sh"]
