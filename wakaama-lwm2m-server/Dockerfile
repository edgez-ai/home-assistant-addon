FROM debian:bookworm-slim

ARG TARGETARCH

RUN apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        jq \
        fonts-dejavu-core \
        libmicrohttpd12 \
        libmosquitto1 \
        libglib2.0-0 \
        libcairo2 \
        libfontconfig1 \
        libfreetype6 \
        libfribidi0 \
        libgraphite2-3 \
        libharfbuzz0b \
        libpango-1.0-0 \
        libpangoft2-1.0-0 \
        libpangocairo-1.0-0 \
        libthai0 \
    && rm -rf /var/lib/apt/lists/*

RUN apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
        gstreamer1.0-tools \
        libgstreamer1.0-0 \
        libgstreamer-plugins-base1.0-0 \
        libgstrtspserver-1.0-0 \
        gstreamer1.0-plugins-base \
        gstreamer1.0-plugins-base-apps \
        gstreamer1.0-plugins-good \
        gstreamer1.0-plugins-bad \
        gstreamer1.0-plugins-ugly \
    && rm -rf /var/lib/apt/lists/*

# Diagnostics only; do not fail the build here. The runtime diagnostic in
# run.sh will produce a more useful report if anything is missing in the
# deployed image.
RUN set +e; \
    echo "=== gst-inspect verification (informational) ==="; \
    gst-inspect-1.0 videotestsrc videoconvert x264enc rtph264pay >/dev/null \
        && echo "OK: core encode pipeline elements present" \
        || echo "WARNING: core encode pipeline elements missing"; \
    gst-inspect-1.0 pango >/dev/null \
        && echo "OK: pango plugin present" \
        || echo "WARNING: pango plugin not found at build time"; \
    gst-inspect-1.0 clockoverlay textoverlay >/dev/null \
        && echo "OK: clockoverlay/textoverlay present" \
        || echo "WARNING: clockoverlay/textoverlay not found at build time"; \
    rm -rf /root/.cache/gstreamer-1.0; \
    true

COPY . /opt/addon

RUN set -eu; \
        arch="${TARGETARCH:-}"; \
        if [ -z "$arch" ]; then \
            arch="$(dpkg --print-architecture)"; \
        fi; \
        case "$arch" in \
            amd64) candidate="/opt/addon/lwm2mserver-amd64"; eddsa_candidate="/opt/addon/libeddsa-amd64.so.0" ;; \
            arm64|aarch64) candidate="/opt/addon/lwm2mserver-arm64"; eddsa_candidate="/opt/addon/libeddsa-arm64.so.0" ;; \
            *) candidate="" ;; \
        esac; \
        if [ -n "$candidate" ] && [ -f "$candidate" ]; then \
            cp "$candidate" /usr/local/bin/lwm2mserver; \
            echo "Using arch-specific binary: $candidate"; \
        else \
            cp /opt/addon/lwm2mserver /usr/local/bin/lwm2mserver; \
            echo "Using fallback binary: /opt/addon/lwm2mserver"; \
        fi; \
        if [ -n "${eddsa_candidate:-}" ] && [ -f "$eddsa_candidate" ]; then \
            cp "$eddsa_candidate" /usr/local/lib/libeddsa.so.0; \
            ln -sf /usr/local/lib/libeddsa.so.0 /usr/local/lib/libeddsa.so; \
            echo "Installed runtime lib: $eddsa_candidate"; \
        else \
            echo "WARNING: No architecture-specific libeddsa runtime found at ${eddsa_candidate:-<none>}"; \
        fi; \
        cp /opt/addon/run.sh /run.sh; \
        mkdir -p /usr/lib/lua/luci/edgez/sensors; \
        if [ -d /opt/addon/sensors ]; then \
            cp -a /opt/addon/sensors/. /usr/lib/lua/luci/edgez/sensors/; \
        fi; \
        chmod +x /usr/local/bin/lwm2mserver /run.sh

EXPOSE 5683/udp
EXPOSE 8088/tcp
EXPOSE 4001/tcp

ENV EDGEZ_CONFIG_FILE=/data/edgez/uci_fallback.conf
ENV LD_LIBRARY_PATH=/usr/local/lib:${LD_LIBRARY_PATH}
VOLUME ["/data"]

ENTRYPOINT ["/run.sh"]