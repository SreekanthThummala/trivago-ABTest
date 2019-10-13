# trivago-ABTest

Split Clients for A/B Testing
The Split Clients module allows you to split incoming traffic between upstream groups based on a request characteristic of your choice. You define the split as the percentage of incoming traffic to forward to the different upstream groups. A common use case is testing the new version of an application by sending a small proportion of traffic to it and the remainder to the current version. In our example, we’re sending 5% of the traffic to the upstream group for the new version, appversion2, and the remainder (95%) to the current version, appversion1.

We’re splitting the traffic based on the client IP address in the request, so we set the split_clients directive’s first parameter to the NGINX variable $remote_addr. With the second parameter we set the variable $upstream to the name of the upstream group.

Here’s the basic configuration:

split_clients $remote_addr $upstream {
    5% appversion2;
    *  appversion1;
}

upstream appversion1 {
   # ...
}

upstream appversion2 {
   # ...
}

server {
    listen 80;
    location / {
        proxy_pass http://$upstream;
    }
}
