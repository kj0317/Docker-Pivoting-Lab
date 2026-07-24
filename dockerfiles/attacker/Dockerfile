FROM ubuntu:22.04

# Versions - bump these as new releases come out
ARG LIGOLO_VERSION=0.9
ARG CHISEL_VERSION=1.11.3

# Avoid interactive prompts during build
ENV DEBIAN_FRONTEND=noninteractive

# ──────────────────────────────────────────────
# 1. System packages & pentest utilities
# ──────────────────────────────────────────────
RUN apt-get update && apt-get install -y --no-install-recommends \
    # Networking / enumeration
    nmap \
    netcat-openbsd \
    iputils-ping \
    traceroute \
    dnsutils \
    curl \
    wget \
    # SSH tunneling method
    openssh-client \
    # SOCKS proxy support (SSH + Chisel methods)
    proxychains4 \
    # Credential attacks
    hydra \
    # File serving & scripting
    python3 \
    # TUN/TAP interface management (Ligolo-ng method)
    iproute2 \
    iptables \
    # socat — TCP relay chains (Method 4)
    socat \
    # sshuttle — transparent VPN over SSH (Method 5)
    sshuttle \
    # Quality of life
    vim \
    nano \
    less \
    net-tools \
    jq \
    tmux \
    # Build essentials for any ad-hoc needs
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# ──────────────────────────────────────────────
# 2. Detect architecture for multi-arch support
#    (works on both x86_64 and Apple Silicon)
# ──────────────────────────────────────────────
ARG TARGETARCH

# ──────────────────────────────────────────────
# 3. Ligolo-ng — proxy (runs here) + agent (served to targets)
# ──────────────────────────────────────────────
RUN mkdir -p /opt/ligolo && \
    # Download proxy (runs on attacker)
    wget -q "https://github.com/nicocha30/ligolo-ng/releases/download/v${LIGOLO_VERSION}/ligolo-ng_proxy_${LIGOLO_VERSION}_linux_${TARGETARCH}.tar.gz" \
        -O /tmp/ligolo-proxy.tar.gz && \
    tar -xzf /tmp/ligolo-proxy.tar.gz -C /opt/ligolo/ && \
    mv /opt/ligolo/proxy /opt/ligolo/ligolo-proxy && \
    chmod +x /opt/ligolo/ligolo-proxy && \
    # Download agent (will be served to compromised hosts)
    wget -q "https://github.com/nicocha30/ligolo-ng/releases/download/v${LIGOLO_VERSION}/ligolo-ng_agent_${LIGOLO_VERSION}_linux_${TARGETARCH}.tar.gz" \
        -O /tmp/ligolo-agent.tar.gz && \
    tar -xzf /tmp/ligolo-agent.tar.gz -C /opt/ligolo/ && \
    mv /opt/ligolo/agent /opt/ligolo/ligolo-agent && \
    chmod +x /opt/ligolo/ligolo-agent && \
    rm -f /tmp/ligolo-*.tar.gz

# ──────────────────────────────────────────────
# 4. Chisel — single binary acts as both server and client
# ──────────────────────────────────────────────
RUN mkdir -p /opt/chisel && \
    wget -q "https://github.com/jpillora/chisel/releases/download/v${CHISEL_VERSION}/chisel_${CHISEL_VERSION}_linux_${TARGETARCH}.gz" \
        -O /tmp/chisel.gz && \
    gunzip /tmp/chisel.gz && \
    mv /tmp/chisel /opt/chisel/chisel && \
    chmod +x /opt/chisel/chisel

# ──────────────────────────────────────────────
# 5. Staging directory for serving tools to targets
#    In a real pentest you'd host these via python http.server
# ──────────────────────────────────────────────
RUN mkdir -p /opt/serve && \
    ln -s /opt/ligolo/ligolo-agent /opt/serve/ligolo-agent && \
    ln -s /opt/chisel/chisel /opt/serve/chisel && \
    ln -s /usr/bin/socat /opt/serve/socat

# ──────────────────────────────────────────────
# 6. Proxychains configuration
# ──────────────────────────────────────────────
COPY proxychains.conf /etc/proxychains4.conf

# ──────────────────────────────────────────────
# 7. Small wordlist for credential attacks
# ──────────────────────────────────────────────
COPY wordlist.txt /opt/wordlists/wordlist.txt

# ──────────────────────────────────────────────
# 8. Convenience: add tools to PATH
# ──────────────────────────────────────────────
ENV PATH="/opt/ligolo:/opt/chisel:${PATH}"

# ──────────────────────────────────────────────
# 9. Welcome message with lab orientation
# ──────────────────────────────────────────────
RUN echo '#!/bin/bash' > /etc/profile.d/lab-banner.sh && \
    echo 'echo ""' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo "============================================="' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo "  PIVOTING LAB — Attacker Machine"' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo "============================================="' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo ""' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo "  Network:    10.10.1.0/24 (Net A)"' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo "  Your IP:    10.10.1.10"' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo ""' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo "  Tools:"' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo "    Ligolo-ng proxy : /opt/ligolo/ligolo-proxy"' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo "    Ligolo-ng agent : /opt/ligolo/ligolo-agent"' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo "    Chisel          : /opt/chisel/chisel"' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo "    socat           : /usr/bin/socat"' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo "    sshuttle        : /usr/bin/sshuttle"' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo "    Serve dir       : /opt/serve/ (ligolo-agent, chisel, socat)"' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo ""' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo "  To serve tools to targets:"' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo "    cd /opt/serve && python3 -m http.server 8000"' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo ""' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo "  Tunneling methods available:"' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo "    1. SSH + proxychains  (traditional SOCKS)"' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo "    2. Chisel             (SOCKS over HTTP)"' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo "    3. Ligolo-ng          (TUN interface, cleanest)"' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo "    4. socat              (TCP relay chains)"' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo "    5. sshuttle           (transparent VPN over SSH)"' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo "    6. netsh              (Windows built-in — see README)"' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo ""' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo "============================================="' >> /etc/profile.d/lab-banner.sh && \
    echo 'echo ""' >> /etc/profile.d/lab-banner.sh && \
    chmod +x /etc/profile.d/lab-banner.sh

WORKDIR /root

CMD ["sleep", "infinity"]
