ARG BUILD_FROM
FROM $BUILD_FROM

ARG EUFY_SECURITY_WS_VERSION

WORKDIR /usr/src/app
RUN \
    set -x \
    && apk add --no-cache jq nodejs npm \
    && node --version \
    && case "$(node --version)" in v24.*) ;; *) echo "Expected Node 24, got $(node --version)" >&2; exit 1 ;; esac \
    && npm install --force "eufy-security-ws@${EUFY_SECURITY_WS_VERSION}"

# --- PATCH: raise P2P heartbeat tolerance so a busy HomeBase (streaming a
# --- detection event) does not get torn down. MAX_RETRIES 10 -> 24, i.e.
# --- ~120s of missed pongs before "connection lost" instead of ~50s.
# --- Build fails loudly if the target string is not found.
RUN set -x \
    && F="$(find /usr/src/app/node_modules/eufy-security-client -path '*p2p*session.js' | head -1)" \
    && test -n "$F" \
    && sed -i -E 's/MAX_RETRIES = 10([^0-9])/MAX_RETRIES = 24\1/g' "$F" \
    && grep -q 'MAX_RETRIES = 24' "$F" \
    && echo "HEARTBEAT PATCH OK -> $F"

COPY run.sh /
RUN chmod a+x /run.sh
WORKDIR /data

CMD [ "/run.sh" ]
