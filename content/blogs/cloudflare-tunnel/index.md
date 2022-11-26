---
title: "Cloudflare Tunnel"
summary: "Using Cloudflare tunnels with Kubernetes"
showSummary: true
heroStyle: "basic"
date: "2022-11-25"
series: ["Using Cloudflare"]
series_order: 1
tags: ["Kubernetes","Cloudflare","SelfHosted"]
---

### Overview

I've been running Kubernetes in my Home Lab for a long time now. There have been some rough patches, but on the whole it works well for me, and through trying to figure out how to best self host a variety of software I've learned a lot about how things work. 

This post isn't specifically about Kubernetes, but understanding how I have things set up is useful context. I have two `nginx` ingress controllers running. One is labeled `internal`, and the other `external`. If it's not obvious, one is for services that I access from my local LAN, and the other is for services that I expose to the internet, with the internal one being the default. Until recently that meant that I had ports 80 & 443 forwarded via my firewall to the `LoadBalancer` IP of the external ingress controller. To put some guardrails around that, I restricted access to the [Cloudflare IPs](https://www.cloudflare.com/ips/), and set the Cloudflare DNS record to `Proxied`. My external IP rarely changes, so that worked fine. However, I've recently discovered Cloudflare Tunnels (formerly called Argo Tunnels, and now called Cloudflare Zero Trust Access Tunnels ... yeah, I don't understand it either). This post is about setting up a Cloudflare Tunnel in a Kubernetes environment to work with a specific ingress controller. 

So, what's the advantage of using a Cloudflare Tunnel? In short, it means you don't need to port forward 80 & 443 on your firewall. The way it works is there's a `cloudflared` process that runs locally that connects, via configuration, to a named Cloudflare Tunnel (yes, you can have more than one). One can think of it as a specialized VPN connection. The `cloudflared` configuration contains rules that can direct traffic to different targets. In my case this is simplied since I just need one rule to send everything to the ingress controller designated for external services. Once the `cloudflared` client is connected, the DNS for the service needs to be updated to point to the generated FQDN of the Cloudflare Tunnel. Clients get directed to Cloudflare's external endpoint, traverse the tunnel, and ultimately connect to the locally running service. 

### Setup

#### Cloudflare

While a tunnel can be created via the `Create a tunnel` button at `Cloudflare Zero Trust -> Access -> Tunnels`, I find it easier to use the `cloudflared` CLI. It is available via the [cloudflared GitHub repository](https://github.com/cloudflare/cloudflared) or via various package management systems. I'm on a Mac, so I used `brew install cloudflare/cloudflare/cloudflared`. 

Once installed `cloudflared login` will launch a web browser so you can provide your Cloudflare credentials. When complete, creating a new tunnel is as simple as:

`cloudflare tunnel create NAME`

The important thing here is a credentials JSON file will be created:

`$HOME/.cloudflared/CloudflareTunnelGUID.json`

along with a certifictate:

`$HOME/.cloudflared/cert.pem`

The JSON file contains the token information to allow `cloudflared` to connect to the named tunnel. Since it will be running in Kubernetes, the file will need to be made availabile to the Pod at a mount point of `/etc/cloudflared/creds/credentials.json`. There are many ways to achieve this. In my case, I just made the file a Secret in the namespace that `cloudflared` will be running in:

`kubectl create secret generic -n cloudflared tunnel-credentials --from-file=credentials.json=$HOME/.cloudflared/CloudflareTunnelGUID.json`

I do the same with the cert file:

`kubectl create secret generic -n cloudflared tunnel-cert --from-file=$HOME/.cloudflared/cert.pem`

#### Kubernetes

When it comes to the rest of the Kubernetes setup, there are only two components, the actual Deployment, and a ConfigMap. 

The ConfigMap, which defines a `config.yaml` for cloudflared, is straight forward, and contains the following elements:

* `tunnel`
	* This is the NAME of the tunnel from where it was created.
* `credentials-file`
	* The full path to the credentials.json file.
* `no-autoupdate`
	* This is set to `true` since it doesn't make sense for the software to try to update itself in the context of a container. 
* `ingress`
	* This is the *cloudflared* ingress configuration section. While multiple rules can be placed here, I want it to send all traffic to a specific endpoint. 
	* `service`
		* This specifies the default endpoint, and I point it to the internal cluster name of my external ingress controller. 

My ConfigMap file can be seen [here](https://github.com/ttyS0/kubernetes/blob/main/cloudflared/configmap.yaml). 

The Deployment is also straight forward. I don't guarentee any uptimes for anything I self host, so I'm less concerned about HA. Therefore, I'm fine with a single tunnel, and a single `cloudflared` replica. 

The Secrets for the `credentials.json` and `cert.pem` are mounted into the Pod, as is the `config.yaml` ConfigMap. The container is given run arguments of `tunnel --config /etc/cloudflared/config/config.yaml run` , and that's all there is to it. 

My Deployment file can be seen [here](https://github.com/ttyS0/kubernetes/blob/main/cloudflared/deployment.yaml)

### Post Setup

Once it's up and running, verification of the tunnel connection can be done via `kubectl logs` inspection of the running Pod and/or via the Cloudlfare Tunnel web UI. Assuming everything is connected, the next bit is to update the DNS records for the services that need to be accessible via the internet. 

The records need to be of type `CNAME`, have `CloudflareTunnelGUID.cfargotunnel.com` set as the target, and the `Proxy status` enabled. Once that's done, external clients will be able to connect to any Ingresses that are configure with the `IngressClass` of the external ingress controller via the `cloudflared` tunnel. Also, if you're using `Let's Encrypt` certs (which you should be doing, by the way ... I suppose I should write something up about how to setup `cert-manager`), then you can (and should) also enable the `Full (strict)` setting on the domain's `SSL/TLS` Cloudflare configuration. 

### Final Notes

Another advantage to this approach is the Cloudflare Tunnel will act as an IPv6 gateway of sorts. My Kubernetes cluster is IPv4 only, but an IPv6 client will get an AAAA record that resolves to the Cloudflare Tunnel endpoint. In the access logs of the service pod, the client's IPv6 IP will be recorded.

While this approach doesn't guarentee more security than port forwarding ports 80 & 443, I do like that I no longer need to do that. And in the event that my public IP ends up changing, that has no bearing on any of the configuration since the local `cloudflared` client is the one initiating the connection. 

---

Cover image from: https://blog.cloudflare.com/highly-available-and-highly-scalable-cloudflare-tunnels/