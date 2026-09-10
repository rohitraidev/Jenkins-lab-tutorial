# 02 - Install Jenkins on Ubuntu

Use this guide to install Jenkins on the Ubuntu controller VM.

## Verify Java

```bash
java -version
```

For current Jenkins releases, use a supported Java version. Java 21 is a practical choice for this lab.

## Install Jenkins

Configure the official Jenkins Debian repository, then install Jenkins:

```bash
sudo apt update
sudo apt install fontconfig openjdk-21-jre -y
java -version

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install jenkins -y
```

## Start Jenkins

```bash
sudo systemctl enable --now jenkins
sudo systemctl status jenkins
```

## Initial administrator password

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Open Jenkins on port `8080` and complete the initial setup.

> If Jenkins is exposed through Cloudflare Tunnel, do not expose Jenkins unnecessarily through a public firewall rule. Keep the tunnel configuration separate from the private agent network.
