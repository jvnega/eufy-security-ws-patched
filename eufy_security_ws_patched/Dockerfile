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

# --- CLIENT SWAP: replace the bundled (deprecated, broken since the Eufy
# --- backend migration of 2026-09-04) bropat eufy-security-client with the
# --- actively maintained martijnpoppen fork (homey-integration branch),
# --- which handles the new cloud API ("HTTP style success code 200",
# --- house_id on station/device lists, v6/mega login fixes).
# --- Pinned to commit f1c0c71 (2026-08-25, v4.2.6) for reproducible builds.
# --- Build fails loudly if the resulting client version is unexpected.
RUN set -x \
    && apk add --no-cache git \
    && git clone --branch homey-integration --single-branch https://github.com/martijnpoppen/eufy-security-client /tmp/esc \
    && cd /tmp/esc \
    && git checkout f1c0c71 \
    && npm ci \
    && npm run build:ts \
    && npm pack \
    && cd /usr/src/app \
    && npm install --force /tmp/esc/eufy-security-client-*.tgz \
    && node -e "const v=require('/usr/src/app/node_modules/eufy-security-client/package.json').version; if(v!=='4.2.6'){console.error('unexpected eufy-security-client '+v);process.exit(1)};console.log('CLIENT SWAP OK -> '+v)" \
    && rm -rf /tmp/esc /root/.npm

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
