# Jenkins Lab: Expose Jenkins with Cloudflare Tunnel

This document is the companion guide for the video where Jenkins is installed on an Ubuntu ARM64 VM and exposed securely over the Internet using Cloudflare Tunnel.

The goal is to make Jenkins reachable at a public HTTPS hostname without exposing Jenkins port `8080` directly to the Internet and without configuring router port forwarding.

> **Example environment used in this lab**
>
> - VM OS: Ubuntu 26.04 LTS, ARM64
> - Jenkins: `http://localhost:8080`
> - Jenkins VM private IP: `192.168.50.128`
> - Domain: `jenkinsdevops.space`
> - Jenkins hostname: `jenkinslab.jenkinsdevops.space`
> - Cloudflare Tunnel name: `jenkins-lab-jenkinsdevop-space`
>
> Replace these values with your own if your environment is different.

---

## 1. What We Are Building

The final architecture looks like this:

```text
                         Internet
                            |
                            v
              jenkinslab.jenkinsdevops.space
                            |
                            v
                       Cloudflare
                            |
                            | HTTPS / Tunnel
                            v
                  Cloudflare Tunnel
              jenkins-lab-jenkinsdevop-space
                            |
                            v
                  Jenkins Controller VM
                     192.168.50.128
                            |
                            v
                    localhost:8080
                            |
                            v
                         Jenkins
```

The important point is that the Jenkins VM does **not** need a public IP address and port `8080` does **not** need to be forwarded through the home router.

`cloudflared` creates outbound connections from the VM to Cloudflare. Cloudflare then uses those connections to send requests to Jenkins.

---

# Part 1: Install `cloudflared`

## 2. Check the VM Architecture

For an ARM64 Ubuntu VM:

```bash
uname -m
```

Expected output:

```text
aarch64
```

For this VM, use the ARM64 `.deb` package.

### Package selection

| Environment | Package type |
|---|---|
| Debian / Ubuntu | `.deb` |
| Fedora / RHEL | `.rpm` |
| ARM64 Linux | ARM64 package |
| x86-64 Linux | amd64 package |

Cloudflare publishes current `cloudflared` downloads from its official documentation and GitHub releases.

Official downloads:

- https://developers.cloudflare.com/tunnel/downloads/
- https://github.com/cloudflare/cloudflared/releases/latest

## 3. Download the ARM64 Debian Package

Example:

```bash
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64.deb
```

Install it:

```bash
sudo dpkg -i cloudflared-linux-arm64.deb
```

Verify the installation:

```bash
cloudflared --version
```

If the package reports missing dependencies, fix them with:

```bash
sudo apt -f install
```

Then verify again:

```bash
cloudflared --version
```

---

# Part 2: Test Jenkins with a Temporary Tunnel

Before creating a permanent named tunnel, it is useful to prove that `cloudflared` can reach Jenkins.

## 4. Verify Jenkins Locally

First check that Jenkins is running:

```bash
sudo systemctl status jenkins
```

Test the local HTTP endpoint:

```bash
curl http://localhost:8080
```

You should receive Jenkins HTML output.

You can also verify that something is listening on port `8080`:

```bash
sudo ss -lntp | grep 8080
```

## 5. Start a Quick Tunnel

Run:

```bash
cloudflared tunnel --url http://localhost:8080
```

Cloudflare will generate a temporary public URL, normally under a `trycloudflare.com` hostname.

Open the generated URL in a browser.

If the Jenkins login page appears, the following path has been proven:

```text
Internet
   |
   v
Cloudflare Quick Tunnel
   |
   v
cloudflared
   |
   v
localhost:8080
   |
   v
Jenkins
```

### Important

A Quick Tunnel is for temporary testing. It is not the permanent hostname setup used in this lab.

Stop it with:

```text
Ctrl+C
```

---

# Part 3: Add the Domain to Cloudflare

## 6. Create or Sign In to a Cloudflare Account

Create/sign in to your Cloudflare account and add the domain:

```text
jenkinsdevops.space
```

If the domain was purchased from another registrar, Cloudflare will provide two nameservers. Replace the registrar's existing nameservers with the Cloudflare nameservers.

For this lab, the Cloudflare nameservers became:

```text
jonah.ns.cloudflare.com
melinda.ns.cloudflare.com
```

Your assigned nameservers may be different. Always use the two nameservers Cloudflare shows for your domain.

## 7. Verify Nameserver Delegation

After changing nameservers, check from the Jenkins VM:

```bash
dig @8.8.8.8 NS jenkinsdevops.space
```

Also check Cloudflare DNS:

```bash
dig @1.1.1.1 NS jenkinsdevops.space
```

The expected result is that both resolvers eventually return the Cloudflare nameservers, for example:

```text
jenkinsdevops.space.    IN    NS    jonah.ns.cloudflare.com.
jenkinsdevops.space.    IN    NS    melinda.ns.cloudflare.com.
```

### DNS propagation note

Immediately after changing nameservers, different recursive DNS resolvers can temporarily return different results because they cache DNS delegation information.

For example, during propagation you might see:

```text
8.8.8.8  -> old registrar nameservers
1.1.1.1  -> Cloudflare nameservers
```

That does not necessarily mean your configuration is wrong.

You can bypass your local DNS cache by querying a resolver directly:

```bash
dig @8.8.8.8 NS jenkinsdevops.space
```

```bash
dig @1.1.1.1 NS jenkinsdevops.space
```

You can also query the authoritative nameserver directly once you know it:

```bash
dig @jonah.ns.cloudflare.com NS jenkinsdevops.space
```

You do **not** need to clear a remote DNS resolver's cache yourself.

---

# Part 4: Create a Permanent Named Tunnel

## 8. Authenticate `cloudflared`

Run:

```bash
cloudflared tunnel login
```

Cloudflare will provide a URL.

Open the URL in a browser and authorize the Cloudflare domain you want to use.

After successful authentication, `cloudflared` creates:

```text
~/.cloudflared/cert.pem
```

Verify:

```bash
ls -l ~/.cloudflared/
```

The `cert.pem` file is used by locally managed tunnel commands that need account authorization, such as creating tunnels and creating DNS routes.

> Never commit `cert.pem` to GitHub.

## 9. Create the Named Tunnel

Create the tunnel:

```bash
cloudflared tunnel create jenkins-lab-jenkinsdevop-space
```

Cloudflare will return a tunnel UUID similar to:

```text
12345678-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

The command also creates a credentials file similar to:

```text
~/.cloudflared/12345678-xxxx-xxxx-xxxx-xxxxxxxxxxxx.json
```

The UUID and credentials filename will be different in your environment.

## 10. Verify the Tunnel

Run:

```bash
cloudflared tunnel list
```

You should see your new tunnel, for example:

```text
ID                                   NAME
12345678-xxxx-xxxx-xxxx-xxxxxxxxxxxx jenkins-lab-jenkinsdevop-space
```

You can also inspect it with:

```bash
cloudflared tunnel info jenkins-lab-jenkinsdevop-space
```

---

# Part 5: Create the Public DNS Route

## 11. Create the Jenkins Hostname

For this lab we will use:

```text
jenkinslab.jenkinsdevops.space
```

Create the DNS route:

```bash
cloudflared tunnel route dns jenkins-lab-jenkinsdevop-space jenkinslab.jenkinsdevops.space
```

This creates a DNS record that points the hostname to the tunnel's Cloudflare endpoint.

You can verify the record with:

```bash
dig jenkinslab.jenkinsdevops.space
```

Or query Cloudflare directly:

```bash
dig @1.1.1.1 jenkinslab.jenkinsdevops.space
```

The record created for a locally managed Cloudflare Tunnel is a CNAME pointing to a hostname under `cfargotunnel.com`.

### Do we need an A record?

No.

Do **not** create an A record pointing to:

```text
192.168.50.128
```

That is a private LAN address and is not publicly routable.

---

# Part 6: Create `config.yml`

## 12. Create the Configuration File

Create the configuration directory if required:

```bash
mkdir -p ~/.cloudflared
```

Create the file:

```bash
nano ~/.cloudflared/config.yml
```

Use:

```yaml
tunnel: 12345678-xxxx-xxxx-xxxx-xxxxxxxxxxxx
credentials-file: /home/rohit/.cloudflared/12345678-xxxx-xxxx-xxxx-xxxxxxxxxxxx.json

ingress:
  - hostname: jenkinslab.jenkinsdevops.space
    service: http://localhost:8080

  - service: http_status:404
```

Replace:

```text
12345678-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

with your actual tunnel UUID.

The credentials file must also use your actual UUID.

## 13. Understand the Configuration

### `tunnel`

```yaml
tunnel: 12345678-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

Identifies the named Cloudflare Tunnel.

### `credentials-file`

```yaml
credentials-file: /home/rohit/.cloudflared/12345678-xxxx-xxxx-xxxx-xxxxxxxxxxxx.json
```

Tells `cloudflared` which credentials belong to this tunnel.

### `hostname`

```yaml
hostname: jenkinslab.jenkinsdevops.space
```

This is the public hostname users will visit.

### `service`

```yaml
service: http://localhost:8080
```

This is the local Jenkins service.

Because `cloudflared` runs on the Jenkins VM, `localhost:8080` refers to Jenkins on that same VM.

### Catch-all rule

```yaml
- service: http_status:404
```

This should be the final ingress rule. It returns HTTP 404 for requests that do not match an earlier hostname rule.

---

# Part 7: Validate and Test the Tunnel

## 14. Validate the Ingress Rules

Run:

```bash
cloudflared tunnel ingress validate
```

The command should report that the configuration is valid.

You can also test which rule would match the public hostname:

```bash
cloudflared tunnel ingress rule https://jenkinslab.jenkinsdevops.space
```

## 15. Run the Tunnel Manually

Start the tunnel:

```bash
cloudflared tunnel run jenkins-lab-jenkinsdevop-space
```

You should see `cloudflared` establish connections to Cloudflare.

Now open:

```text
https://jenkinslab.jenkinsdevops.space
```

The Jenkins login page should appear.

### Important

At this stage the tunnel is running only in the current terminal session.

If you close the terminal or stop the process with `Ctrl+C`, the tunnel process stops.

The DNS record can remain in Cloudflare even while the tunnel is stopped. If the tunnel is unavailable, visitors will not be able to reach Jenkins.

Stop the manual test with:

```text
Ctrl+C
```

---

# Part 8: Install `cloudflared` as a System Service

The final setup should start `cloudflared` automatically when the Ubuntu VM boots.

## 16. Install the Service

Because the configuration is in `/home/rohit/.cloudflared/config.yml`, explicitly provide the path when using `sudo`:

```bash
sudo cloudflared --config /home/rohit/.cloudflared/config.yml service install
```

This explicit `--config` is important because when `sudo` is used, `$HOME` can resolve to `/root`, causing `cloudflared` to look for `/root/.cloudflared/config.yml` instead of the configuration under `/home/rohit`.

## 17. Enable and Start the Service

Enable it at boot:

```bash
sudo systemctl enable cloudflared
```

Start it:

```bash
sudo systemctl start cloudflared
```

Or do both in one command:

```bash
sudo systemctl enable --now cloudflared
```

## 18. Check the Service

Run:

```bash
sudo systemctl status cloudflared
```

You want to see the service in an active/running state.

For live logs:

```bash
sudo journalctl -u cloudflared -f
```

Press `Ctrl+C` to leave the log view.

Useful additional commands:

```bash
sudo systemctl restart cloudflared
```

```bash
sudo systemctl stop cloudflared
```

```bash
sudo systemctl start cloudflared
```

```bash
sudo systemctl disable cloudflared
```

---

# Part 9: Final Verification

## 19. Verify Jenkins Locally

```bash
curl -I http://localhost:8080
```

## 20. Verify DNS

```bash
dig @1.1.1.1 jenkinslab.jenkinsdevops.space
```

## 21. Verify the Tunnel

```bash
cloudflared tunnel list
```

Then:

```bash
cloudflared tunnel info jenkins-lab-jenkinsdevop-space
```

## 22. Verify the System Service

```bash
sudo systemctl is-active cloudflared
```

Expected:

```text
active
```

## 23. Open Jenkins

Open:

```text
https://jenkinslab.jenkinsdevops.space
```

You should now be able to access Jenkins without entering the private IP address or port `8080`.

---

# Part 10: Optional Apex Domain Route

You can point the root domain to the same tunnel if you specifically want:

```text
https://jenkinsdevops.space
```

to open Jenkins.

Create another DNS route:

```bash
cloudflared tunnel route dns jenkins-lab-jenkinsdevop-space jenkinsdevops.space
```

Then add another ingress rule to `config.yml`:

```yaml
ingress:
  - hostname: jenkinslab.jenkinsdevops.space
    service: http://localhost:8080

  - hostname: jenkinsdevops.space
    service: http://localhost:8080

  - service: http_status:404
```

Then validate and restart:

```bash
cloudflared tunnel ingress validate
sudo systemctl restart cloudflared
```

For this lab, using only `jenkinslab.jenkinsdevops.space` is simpler and keeps the root domain available for other future services.

---

# Part 11: Troubleshooting

## Problem: `Cannot determine default configuration path`

Example:

```text
Cannot determine default configuration path. No file [config.yml config.yaml] in [~/.cloudflared ...]
```

Cause: `cloudflared` was run with `sudo`, so it may be looking under `/root/.cloudflared` instead of `/home/rohit/.cloudflared`.

Fix:

```bash
sudo cloudflared --config /home/rohit/.cloudflared/config.yml service install
```

---

## Problem: Tunnel is created but Jenkins does not load

First check Jenkins:

```bash
sudo systemctl status jenkins
```

Then:

```bash
curl http://localhost:8080
```

If this fails, fix Jenkins first.

Then check the tunnel:

```bash
sudo systemctl status cloudflared
```

And:

```bash
sudo journalctl -u cloudflared -n 100 --no-pager
```

---

## Problem: DNS hostname does not resolve

Check nameserver delegation:

```bash
dig @1.1.1.1 NS jenkinsdevops.space
```

Check the tunnel hostname:

```bash
dig @1.1.1.1 jenkinslab.jenkinsdevops.space
```

If nameserver results differ between resolvers immediately after changing nameservers, allow time for DNS delegation caches to expire.

---

## Problem: `NXDOMAIN` after buying the domain

A newly registered domain can take some time before registration and DNS delegation are visible consistently through recursive resolvers.

Check:

```bash
dig @8.8.8.8 NS jenkinsdevops.space
```

```bash
dig @1.1.1.1 NS jenkinsdevops.space
```

Once the domain is delegated to Cloudflare, both should eventually return the Cloudflare nameservers.

---

## Problem: Cloudflare Tunnel works manually but not as a service

Check the service:

```bash
sudo systemctl status cloudflared
```

Check logs:

```bash
sudo journalctl -u cloudflared -n 100 --no-pager
```

The most common issue in this setup is the configuration path or credentials file path.

Verify:

```bash
ls -l /home/rohit/.cloudflared/
```

Make sure the config contains the correct tunnel UUID and credentials path.

---

## Problem: Wrong hostname in `config.yml`

The DNS route and ingress hostname must match.

For this lab:

```text
DNS hostname:
jenkinslab.jenkinsdevops.space
```

and:

```yaml
hostname: jenkinslab.jenkinsdevops.space
```

Do not accidentally use:

```text
jjenkinslab.jenkinsdevops.space
```

---

# Part 12: Security Notes

## Never publish the private Jenkins IP

Do not create a public DNS A record such as:

```text
jenkinsdevops.space -> 192.168.50.128
```

`192.168.50.128` is a private address used inside the local network.

Cloudflare Tunnel provides the public-to-private connection without requiring that private address to be publicly routable.

## Do not expose Jenkins port 8080 directly

Avoid router port forwarding such as:

```text
Internet:8080 -> 192.168.50.128:8080
```

The tunnel removes the need for this.



The tunnel credentials JSON and `cert.pem` contain sensitive authentication material.


---

# Part 13: Useful Commands Cheat Sheet

## Install

```bash
uname -m
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64.deb
sudo dpkg -i cloudflared-linux-arm64.deb
cloudflared --version
```

## Temporary tunnel

```bash
cloudflared tunnel --url http://localhost:8080
```

## Login

```bash
cloudflared tunnel login
```

## Create tunnel

```bash
cloudflared tunnel create jenkins-lab-jenkinsdevop-space
```

## List tunnels

```bash
cloudflared tunnel list
```

## Create DNS route

```bash
cloudflared tunnel route dns jenkins-lab-jenkinsdevop-space jenkinslab.jenkinsdevops.space
```

## Validate config

```bash
cloudflared tunnel ingress validate
```

## Test tunnel manually

```bash
cloudflared tunnel run jenkins-lab-jenkinsdevop-space
```

## Install systemd service

```bash
sudo cloudflared --config /home/rohit/.cloudflared/config.yml service install
```

## Manage service

```bash
sudo systemctl enable --now cloudflared
sudo systemctl status cloudflared
sudo systemctl restart cloudflared
sudo journalctl -u cloudflared -f
```

## DNS troubleshooting

```bash
dig @8.8.8.8 NS jenkinsdevops.space
dig @1.1.1.1 NS jenkinsdevops.space
dig @1.1.1.1 jenkinslab.jenkinsdevops.space
```

## Tunnel troubleshooting

```bash
cloudflared tunnel info jenkins-lab-jenkinsdevop-space
sudo journalctl -u cloudflared -n 100 --no-pager
```

---

# Final Result

At the end of this setup:

```text
Browser
   |
   | HTTPS
   v
https://jenkinslab.jenkinsdevops.space
   |
   v
Cloudflare
   |
   v
Cloudflare Tunnel
   |
   | outbound connection from Jenkins VM
   v
cloudflared
   |
   v
http://localhost:8080
   |
   v
Jenkins
```

The tunnel runs as a systemd service, so it can automatically start when the VM boots.

The setup provides a clean way to expose the Jenkins controller UI for a home lab and later use the public Jenkins URL for integrations such as GitHub webhooks.

---

## Official References

- Cloudflare Tunnel downloads: https://developers.cloudflare.com/tunnel/downloads/
- Create a locally managed tunnel: https://developers.cloudflare.com/tunnel/advanced/local-management/create-local-tunnel/
- Cloudflare Tunnel configuration file: https://developers.cloudflare.com/tunnel/advanced/local-management/configuration-file/
- Run `cloudflared` as a Linux service: https://developers.cloudflare.com/tunnel/advanced/local-management/as-a-service/linux/
- Cloudflare Tunnel DNS routing: https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/routing-to-tunnel/dns/
