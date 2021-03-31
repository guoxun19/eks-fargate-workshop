# Version:0.0.1
FROM amazonlinux2

RUN yum install -y httpd
COPY ./index.html /var/www/html/index.html

EXPOSE 80
ENTRYPOINT ["/usr/sbin/httpd", "-D", "FOREGROUND"]