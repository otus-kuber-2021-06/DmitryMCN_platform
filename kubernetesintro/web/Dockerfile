FROM nginx
ENV UID=1001
WORKDIR /app
RUN usermod -u ${UID} nginx && groupmod -g ${UID} nginx && \
    chown -R nginx. -R /var/cache/nginx /var/log/nginx /app 

COPY ./nginx.conf /etc/nginx/nginx.conf
EXPOSE 8000
USER ${UID}
ENTRYPOINT ["nginx","-g", "daemon off;"]
