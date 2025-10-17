I want you to setup the $ARGUMENTS service on this NixOS host, either as a native NixOS service or as a quadlet container, whichever is best for my configuration.

I want to accomplish the following:

1. Manage any needed secrets with SOPS

  * Never reveal secrets in this chat; ask me to create and install them for you

2. Setup an nginx virtual host for this service, creating a TLS certificate for the new domain so it can be accessed using HTTPS

  * DO NOT try to generate the certificate yourself, but ask me to generate it for you when you get to that point

3. Setup Prometheus monitor to gather metrics about the service

4. Setup Alertmanager to ensure the health of the service

5. If this service presents a full set of new metrics, create a Grafana dashboard for visualizing those metrics. Use Perplexity MCP to search for possible existing dashboard for this service that may be used, if available.

6. Setup a link under an appropriate section on my Glance dashboard

7. If a new filesystem is being created to support this service, add it to the set of available Samba mounts

8. Test to ensure the newly installed service is working before you finish your work

Everything you do should be coherent with the other services on this NixOS machine. Do not reveal ANY secrets during this chat, and always ask me if you need to create a new SOPS secret or you need to create a Web SSL certificate.

Use Web Search and Perplexity MCP as needed to discover what is the best way to setup and configure the $ARGUMENTS service. Some further notes:

* If there is a choice of backing database, I prefer to use the PostgreSQL and Redis services already running on this server, even though you will likely need to create new users and databases within those services.

Use nix-pro to perform your work on these installation steps. Take as long as needed to ensure that the service is well integrated, coherent with the rest of this machine’s configuration, and functioning well before you are finished.

- Do not, under any circumstances, decrypt the SOPS secrets.yaml file.
- Use Web Search and Perplexity as need for research and discovering resources.
- Use sequential-thinking when appropriate to break down tasks further.
- Use context7 whenever code examples might help.
- Use memory-keeper with channel: <project_name>
- Save progress at every major milestone
- Document all decisions with category: "decision"
- Track implementation status with category: "progress"
