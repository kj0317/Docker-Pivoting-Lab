FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

# ──────────────────────────────────────────────
# 1. System packages
#    NOTE: No SSH server — this is deliberate.
#    Forces learners to use Chisel/Ligolo-ng for
#    the double pivot, showing their advantage.
# ──────────────────────────────────────────────
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 \
    python3-pip \
    curl \
    wget \
    # socat — for relay chaining (Method 4) and as a netsh Linux analogue
    socat \
    iputils-ping \
    iproute2 \
    net-tools \
    procps \
    && rm -rf /var/lib/apt/lists/*

# ──────────────────────────────────────────────
# 2. Flask for the internal admin dashboard
# ──────────────────────────────────────────────
RUN pip3 install --no-cache-dir flask

# ──────────────────────────────────────────────
# 3. Application files
# ──────────────────────────────────────────────
COPY app.py /opt/webapp/app.py
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

EXPOSE 8080

CMD ["/entrypoint.sh"]
