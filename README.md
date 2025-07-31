# AWS-T-pot Using Linux with Kibana/Elastic, Cowrie & tcpdump
<img width="800" alt="image" src="https://github.com/user-attachments/assets/c8947fbb-7efc-4ff2-992c-d03c66b8ecfc">


# 🛠 Tools Used

- **AWS Security Groups**
- **AWS EC2**
- **tcpdump**
- **Cowrie**
- **Kibana / Elasticsearch**
- **Linux (Debian 12 distribution)**

> While some processes overlapped, I deliberately used multiple analysis methods to compare their effectiveness. This helped me understand when to use `tcpdump` versus container logs versus Kibana for different types of security investigations.

---

# ☁️ AWS EC2 Setup

I selected **Debian 12** as my operating system.

<img width="1616" alt="image" src="https://github.com/user-attachments/assets/c442c769-7d33-45ec-a24e-0226f7abb9f0">

I chose a **t3.xlarge instance**, which exceeds T-Pot's recommended minimums of:
- **8–16 GB RAM**
- **128 GB free disk space**
<img width="1616" alt="image" src="https://github.com/user-attachments/assets/47fbfd6c-3fd9-455e-9168-b53f4134ce91">

<img width="1616" alt="image" src="https://github.com/user-attachments/assets/2a373c4f-1946-4381-b874-39ae89ed02a4">




## Pre-Launch Configuration

- Created a **key pair** for SSH authentication
- Restricted **SSH access to my IP only**
- Allocated **128 GB of storage**
- Adjusted **inbound rules**:
  - Opened TCP **ports 1–64000** from any IP (for attack traffic)
  - Allowed management ports & SSH from my IP

<img width="1616" alt="image" src="https://github.com/user-attachments/assets/7eca83b8-07b5-4183-bce8-cb2ada094b6e">

<img width="1616" alt="image" src="https://github.com/user-attachments/assets/53caa947-3cb6-4ed5-9cf7-b0f17a6d2039">


---

# 🔧 T-Pot Installation

```bash
# SSH into instance
ssh -i "[path to key]" -p 22 admin@[tpot_public_ip]

# Prepare environment
sudo apt-get update
sudo apt-get install -y git

# Download T-Pot Community Edition
git clone https://github.com/telekom-security/tpotce
cd tpotce
./install.sh
# Select: H (Hive)
# Set login credentials and record them

# Reboot after install
sudo reboot
```

<img width="1616" alt="image" src="https://github.com/user-attachments/assets/eed17f31-360a-4d78-a5bd-5e51f12185d9">

<img width="1616" alt="image" src="https://github.com/user-attachments/assets/035ba693-8a84-44a4-b7da-7007cc8136a4">



```bash
# SSH back in using new T-Pot SSH port (example)
ssh -i "[path to key]" -p 64295 admin@[tpot_public_ip]

# Check and start T-Pot
systemctl status tpot
sudo systemctl start tpot
```

Access the T-Pot dashboard:
```
https://[tpot_public_ip]:64297
```
<img width="1616" alt="image" src="https://github.com/user-attachments/assets/71419777-83ad-4a8a-912e-08b3f416470a">




---

# 🧹 Post-Install Fixes

I noticed a lingering container (`mailoney`) was using port 25. To resolve:

```bash
sudo systemctl stop exim4
sudo systemctl start tpot
systemctl status tpot
```

Checked container health:

```bash
sudo docker ps
```

Waited for attacks to begin.

---


# 🔍 Analyzing Attacker Behavior with Cowrie Logs

```bash
sudo docker logs cowrie | grep -E "(New connection|login attempt.*failed|login attempt.*succeeded|CMD:)"
```
<img width="1616" alt="image" src="https://github.com/user-attachments/assets/c67bc565-e217-4fbd-b09a-0e34479d61d9">

This command revealed:
- New connections
- Login attempts (both failed and successful)
- Commands executed by attackers.  I found the commands executed by the IP "8.219.248.7" to be interesting.
---
# 📡 Traffic Capture with tcpdump

```bash
sudo tcpdump -i br-ee1fa6a51e9e -nn -s 0 port 22 or port 23
```
<img width="1616" alt="image" src="https://github.com/user-attachments/assets/f6618b4b-e043-464a-88ff-1fcf5d60c2d5">


Explanation:
- `-i` — capture from specific Docker bridge interface
- `-nn` — disable name resolution (IP and ports stay numeric)
- `-s 0` — capture full packet contents

Initially, I wasn't capturing any traffic with tcpdump because the attacks were targeting the Dockerized containers inside the T-Pot environment, not the host interface. After identifying this, I shifted focus to Cowrie, where I had observed prior SSH interactions and could review attacker commands directly. To align tcpdump with the correct network layer, I ran docker network ls to identify the bridge network ID, then used that ID in the interface parameter of my capture command.

Although my primary target IP (8.219.248.7) did not appear in this specific capture—later confirmed via Elastic logs to have completed its activity several hours earlier—I was still able to observe multiple active Telnet connection attempts from a new source IP (121.150.18.228). These were captured and analyzed to demonstrate ongoing scanning behavior and real-world attack traffic within the honeypot environment.

---

# 📊 Enriching Analysis with Kibana / Elasticsearch

Using Kibana, I searched:

```
src_ip: "8.219.248.7"
```

<img width="1616" alt="image" src="https://github.com/user-attachments/assets/45075d83-5830-4258-86c8-85e4e6b477ff">

I moved over to "Available Fields" in the left column and scrolled down to "DestPort".  I noticed "22" was the only value.  I also searched for other commands the attacker ran.


```
src_ip: "8.219.248.7 AND message : "CMD"
```

<img width="1616" alt="image" src="https://github.com/user-attachments/assets/b10ece5c-c983-4ced-af7b-5a9d3ab50c67">

This attacker attempted **1900+ SSH connections**, exclusively targeting **port 22**. As also observed in the Cowrie `input` field, I found this command issued:

```bash
uname -s -v -n -r -m
```


This command fingerprints the system:

| Flag | Description                            |
|------|----------------------------------------|
| `-s` | Kernel name (Linux, FreeBSD, etc.)     |
| `-v` | Kernel version & build date            |
| `-n` | Node name (hostname)                   |
| `-r` | Kernel release (e.g. 5.4.0-74-generic) |
| `-m` | Machine type (e.g. x86_64)             |

---

# ✅ Conclusion

This **T-Pot honeypot deployment** successfully captured attacker behavior targeting SSH via the Cowrie container.

<img width="1616" alt="image" src="https://github.com/user-attachments/assets/ef5e6972-454d-4e71-9d05-14ec3f86cad5">

As seen in the Kibana dashboard, I could have taken this investigation in a number of different directions. The T-Pot honeypot logs a wide range of attack data across multiple services and protocols including SSH, Telnet, SMB, HTTP, and more. Each container (Cowrie, Dionaea, ConPot, etc.) represents a different piece of the attack surface, and each visualization offers a launchpad for deeper analysis.

For this particular project, I narrowed my focus to a single attacker, who exhibited consistent SSH brute-force and reconnaissance behavior. This attacker made over 1900 SSH attempts and demonstrated clear post-compromise reconnaissance behavior. The attacker was likely gathering intel in preparation for privilege escalation or malware deployment.  The dashboard reveals much more: password spraying, exploitation attempts, port scans, and geographic trends, all of which could be the foundation for a future investigation.

T-Pot captures attacks in high fidelity and makes it possible to pivot and explore attacker behaviors from multiple angles. 



---
### 🧠 Mapped MITRE ATT&CK Techniques

| Tactic               | Technique Name                 | Technique ID     |
|----------------------|--------------------------------|------------------|
| **Discovery**         | System Information Discovery   | `T1082`          |
| **Discovery**         | Network Service Scanning       | `T1046`          |
| **Credential Access** | Brute Force: SSH               | `T1110.001`      |
| **Execution**         | Command & Scripting (Bash)     | `T1059.004`      |
| **Lateral Movement**  | Remote Services: Telnet        | `T1021.001`      |

