FROM nginxinc/nginx-unprivileged:alpine3.24
USER root
# The proxy uses only built-in HTTP/FastCGI modules. apk also removes orphaned
# XML/XSLT, JavaScript, GeoIP, and image-processing dependencies.
RUN apk del --no-network nginx-module-xslt nginx-module-njs nginx-module-geoip nginx-module-image-filter
RUN apk upgrade --no-cache
# The base pins nginx in /etc/apk/world; replace that constraint to receive security fixes.
RUN apk add --no-cache --upgrade nginx
# Fail the build if a future base-image change retains the XML libraries.
RUN ! apk info --exists libxml2 && ! apk info --exists libxslt
COPY nginx.conf /etc/nginx/nginx.conf
USER 101:101
EXPOSE 8080
# Set the startup error log too, before nginx reads its configuration.
CMD ["nginx", "-e", "/dev/stderr", "-g", "daemon off;"]
