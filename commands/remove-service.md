---
description: Remove a service from system, including nginx virtual hosts, monitoring, alerting, systemd services and timers, containers, Nagios, Alertmanager, Prometheus exporters, etc.
argument-hint: <service-name>
---

I want you to remove the $ARGUMENTS service from this NixOS host, including nginx virtual hosts, monitoring, alerting, systemd services and timers, containers, Nagios, Alertmanager, Prometheus exporters, etc.

Everything you do should be coherent with the other services on this NixOS machine. Do not reveal ANY secrets during this chat, and always ask me if you need to create a new SOPS secret or you need to create a Web SSL certificate.

Use Web Search and Perplexity MCP as needed to discover what is the best way to fully remove the $ARGUMENTS service. Some further notes:

Use the nixos skill to perform your work on these installation steps. Take as long as needed to ensure that the service is well integrated, coherent with the rest of this machine’s configuration, and functioning well before you are finished.
