FROM nginx:latest
# Copy the static content to the Nginx HTML directory
COPY ./content /usr/share/nginx/html
# Expose port 80 for the Nginx server
EXPOSE 80
# Set the default command to run Nginx in the foreground
CMD ["nginx", "-g", "daemon off;"]