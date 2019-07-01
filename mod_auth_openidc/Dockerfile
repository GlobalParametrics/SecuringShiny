FROM centos

RUN yum install -y mod_auth_openidc

RUN mkdir -p /etc/httpd/conf/sites-enabled
RUN echo "Include conf/sites-enabled/*.conf" \
  >> /etc/httpd/conf/httpd.conf

RUN echo "httpd -k start && tail -f /var/log/httpd/error_log" > startup.sh

ENTRYPOINT ["startup.sh"]
