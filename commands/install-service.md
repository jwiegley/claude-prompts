I want you to setup the $ARGUMENTS service on this NixOS host, either as a native NixOS service or as a quadlet container, whichever is best for my configuration.

I want to accomplish the following:

1. manage any needed secrets with SOPS

  * never reveal secrets in this chat; ask me to create and install them for you

2. setup an nginx virtual host for this service, creating a TLS certificate for the new domain so it can be accessed using HTTPS

  * do not try to generate the certificate yourself, but ask me to generate it for you when you get to that point

3. setup Prometheus monitor to gather metrics about the service

4. setup Alertmanager to ensure the health of the service

5. setup a link under an appropriate section on my Glance dashboard

6. test to ensure the newly installed service is working before you finish your work

Everything you do should be coherent with the other services on this NixOS machine. Do not reveal ANY secrets during this chat, and always ask me if you need to create a new SOPS secret or you need to create a Web SSL certificate.

Use Web Search and Perplexity MCP as needed to discover what is the best way to setup and configure the $ARGUMENTS service. Some further notes:

* If there is a choice of backing database, I prefer to use the PostgreSQL and Redis services already running on this server, even though you will likely need to create new users and databases within those services.
