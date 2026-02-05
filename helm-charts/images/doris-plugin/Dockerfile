# syntax=docker/dockerfile:1
# DOGStack Grafana Assets Image
# Contains: Doris App Plugin + Dashboards
#
# Build:
#   cd DOG-k8s/images/doris-plugin
#   docker build -t your-registry/dog-grafana-assets:v1.0.0 .
#
# Build with custom plugin URL:
#   docker build -t dog-grafana-assets:v1.0.0 \
#     --build-arg DORIS_APP_URL=https://your-s3-bucket/doris-app-1.0.0.zip .

FROM alpine:3.19

# Plugin download URL (COS, S3, GitHub Releases, etc.)
ARG DORIS_APP_URL=https://justtmp-1308700295.cos.ap-hongkong.myqcloud.com/doris-app-1.0.0.zip
ARG DORIS_APP_VERSION=1.0.0

# Install dependencies
RUN apk add --no-cache wget unzip

# Create directories
RUN mkdir -p /assets/plugins /assets/dashboards

# Download and extract doris-app plugin
RUN wget -q -O /tmp/doris-app.zip "${DORIS_APP_URL}" && \
    unzip -q /tmp/doris-app.zip -d /assets/plugins/ && \
    rm /tmp/doris-app.zip && \
    chmod -R 755 /assets/plugins/

# Copy dashboards (these are small JSON files, keep in repo)
COPY dashboards/ /assets/dashboards/
RUN chmod -R 755 /assets/dashboards/

# Metadata
LABEL org.opencontainers.image.title="DOGStack Grafana Assets"
LABEL org.opencontainers.image.description="Doris App Plugin and Dashboards for Grafana"
LABEL org.opencontainers.image.version="${DORIS_APP_VERSION}"

CMD ["sh", "-c", "cp -r /assets/plugins/* /target/plugins/ && cp -r /assets/dashboards/* /target/dashboards/ && echo 'Assets copied successfully'"]
