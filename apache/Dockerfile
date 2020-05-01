FROM httpd:2.4.33-alpine

RUN apk update; \
    apk upgrade;

COPY apache_php.conf /usr/local/apache2/conf/apache_php.conf
RUN echo "Include /usr/local/apache2/conf/apache_php.conf" \
    >> /usr/local/apache2/conf/httpd.conf
