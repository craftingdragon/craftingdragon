FROM alpine:3.23
RUN apk upgrade --no-cache
RUN apk add --no-cache nginx \
 && addgroup -S -g 10002 readmeproxy \
 && adduser -S -u 10002 -G readmeproxy readmeproxy
COPY nginx.conf /etc/nginx/nginx.conf
USER 10002:10002
EXPOSE 8080
CMD ["nginx", "-g", "daemon off;"]
