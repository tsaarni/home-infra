# https://hub.docker.com/r/nodered/node-red/tags
# tags https://github.com//node-red/node-red-docker
#   release plan https://nodered.org/about/releases/
# https://flows.nodered.org/node/node-red-contrib-prometheus-exporter
FROM nodered/node-red:3.0.2-18
RUN npm install node-red-contrib-prometheus-exporter@1.0.5 && \
    npm install node-red-contrib-bigtimer@2.8.1 && \
    npm install tsaarni/node-red-contrib-interpolate && \
    npm install json-rpc-2.0@1.4.2
