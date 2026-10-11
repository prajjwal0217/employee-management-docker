FROM nginx:mainline-alpine3.24

# Remove the default Nginx virtual host
RUN rm /etc/nginx/conf.d/default.conf

# Copy Angular production build
COPY employee-web/ /var/www/employee-web/

# Copy SSL certificates
COPY ssl/ /etc/nginx/ssl/

# Copy application Nginx configuration
COPY employee-web.conf /etc/nginx/conf.d/employee-web.conf

# Nginx listens on these container ports
EXPOSE 80 443