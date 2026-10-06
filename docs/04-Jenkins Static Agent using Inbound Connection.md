
--------------------------------

**Temporary Static Node Addition**


curl -sO http://192.168.50.128:8080/jnlpJars/agent.jar

java -jar agent.jar \
-url http://192.168.50.128:8080/ \
-secret <60c974033373e....> \
-name "jenkins-agent-jnlp" \
-webSocket \
-workDir "/home/rohit/agent"

-----------------------------------------------------


**Permanent Static Agent: Background Method**

curl -sO http://192.168.50.128:8080/jnlpJars/agent.jar

nohup java -jar agent.jar \
-url http://192.168.50.128:8080/ \
-secret <60c974033373e....> \
-name "jenkins-agent-jnlp" \
-webSocket \
-workDir "/home/rohit/agent" \
> agent.log 2>&1 &



-----------------------------------------------------


**Permanent Static Agent: systemd Method**

sudo -i 
/etc/jenkins-agent.env
JENKINS_SECRET=<60c974033373e....>


cat > /etc/systemd/system/jenkins-agent.service <<'EOF'
[Unit]
Description=Jenkins Inbound Agent
After=network.target

[Service]
User=rohit
WorkingDirectory=/home/rohit/agent

EnvironmentFile=/etc/jenkins-agent.env

ExecStart=/usr/bin/java -jar /home/rohit/agent.jar \
-connectTo 192.168.50.128:50000 \
-secret ${rohit_SECRET} \
-name "jenkins-agent-jnlp" \
-workDir "/home/rohit/agent"

Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF


systemctl daemon-reload
systemctl enable --now jenkins-agent
systemctl restart jenkins-agent

systemctl status jenkins-agent
journalctl -u jenkins-agent -f
