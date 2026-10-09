# 🛡️ AWS Mini SOC

![AWS](https://img.shields.io/badge/AWS-EC2%20%7C%20VPC-0d1117?style=flat-square&logo=amazonwebservices&logoColor=FF9900)
![Ubuntu](https://img.shields.io/badge/Ubuntu-Linux-0d1117?style=flat-square&logo=ubuntu&logoColor=E95420)
![Wazuh](https://img.shields.io/badge/Wazuh-SIEM-0d1117?style=flat-square&logo=wazuh&logoColor=3D8FF7)
![Nginx](https://img.shields.io/badge/Nginx-Web%20Server-0d1117?style=flat-square&logo=nginx&logoColor=009639)
![MITRE ATT&CK](https://img.shields.io/badge/MITRE-ATT%26CK-0d1117?style=flat-square)

A small cloud-based **Security Operations Center (SOC)** lab built in AWS. A public Linux web server is monitored by a separate security server running **Wazuh**, which collects its logs, turns suspicious activity into alerts, and maps them to **MITRE ATT&CK**.

The lab caught both **simulated attacks** (failed SSH logins I generated) and **real attacks from the internet** (automated scanners probing the public web server).

<p align="center">
    <img src="screenshots/aws-soc-picture.png" alt="Wazuh dashboard showing the soc-web-server agent" width="95%">
</p>

---

## 🏗️ Architecture

```text
AWS VPC (10.0.0.0/16)
│
├── Public Subnet A
│   └── soc-web-server (EC2)
│       ├── Ubuntu Linux
│       ├── Nginx
│       └── Wazuh Agent ───────────┐
│                                  │  logs sent over private
│                                  │  VPC networking
└── Public Subnet B                │
    └── soc-security-server (EC2)  │
        ├── Ubuntu 22.04 LTS       │
        ├── Wazuh Manager ◄────────┘
        ├── Wazuh Indexer
        ├── Wazuh Dashboard (HTTPS)
        └── Filebeat
```

| Component | Role |
|---|---|
| **soc-web-server** | Acts like a normal company server. Hosts a public Nginx website and generates logs from SSH logins, web traffic, and system activity. |
| **soc-security-server** | The SOC. Wazuh collects and analyzes the logs, raises alerts, and shows them in a dashboard. |
| **Private networking** | The Wazuh agent talks to the manager using **private VPC IP addresses**, so security traffic never travels over the public internet. |

### AWS networking

- Custom **VPC** (`10.0.0.0/16`) with two **public subnets** in different Availability Zones
- **Internet Gateway** and a public **route table** (`0.0.0.0/0` → IGW)
- Separate **Security Groups** for each server
- Web server Security Group allows: SSH (22) **from my IP only**, HTTP (80), HTTPS (443), ICMP for testing, and Wazuh traffic **only from inside the VPC**

---

## 🔍 What Wazuh Detected

### 1. Simulated attack: failed SSH login

I generated a failed login against the web server using a user that doesn't exist:

```bash
ssh fakeuser@<WEB_SERVER_PUBLIC_IP>
```

Wazuh raised the alert **`sshd: Attempt to login using a non-existent user`** and recorded the source IP, attempted username, and authentication failure. It mapped the event to MITRE ATT&CK:

| Tactic | Technique |
|---|---|
| Credential Access | Password Guessing |
| Lateral Movement | SSH |

### 2. Real attack: internet reconnaissance

Once the web server was public, **automated scanners found it on their own**. Wazuh detected requests from an internet scanner (`CensysInspect`) probing paths like `/security.txt`, which Nginx answered with `404`.

Repeated `400`/`404` errors were flagged and classified as:

| Tactic | Technique |
|---|---|
| Reconnaissance | **T1595.002**: Active Scanning, Vulnerability Scanning |

Wazuh also enriched these events with approximate **IP geolocation** (city, region, country). These are estimates based on IP address, not exact locations.

### 3. Vulnerability detection

Wazuh scanned the web server's installed packages and reported known vulnerabilities (CVEs). Most findings came from the `linux-aws` kernel package, which shows the next step for a real server: **keep the OS and kernel patched**.

> **Takeaway:** a server on the public internet gets discovered and scanned within a short time of going live, even if nobody knows it exists. Monitoring isn't optional.

---

## 🧰 Troubleshooting (what went wrong and how I fixed it)

| Problem | Symptom | Fix |
|---|---|---|
| **Unsupported OS** | Wazuh installer failed on Ubuntu 26.04 (`wazuh-keystore: No such file or directory`, `wazuh-manager could not be started`) | Rebuilt the security server on **Ubuntu 22.04 LTS** |
| **Not enough disk space** | `No space left on device` on an 8 GB root volume | Increased the **EBS volume to 40 GB**, then expanded the partition and ext4 filesystem (~37 GB free) |
| **Broken package state** | Failed installs left `wazuh-manager` half-installed; `prerm`/`postrm` scripts failed because `/var/ossec` was already gone | Moved the broken package scripts aside so `dpkg` could purge the package, confirmed with `dpkg -l \| grep wazuh`, then did a clean install |
| **Subnet sizes** | The VPC wizard created `/20` subnets instead of the planned `/24` | Kept them, since the lab worked correctly with the larger ranges |
| **GitHub authentication** | Password login rejected for HTTPS Git | Created a **fine-grained Personal Access Token** (Contents: read/write, Metadata: read-only) |
| **Git history conflict** | Push rejected because the remote already had commits | Pulled and merged the remote history before pushing |

---

## 🔐 Security Practices

- SSH restricted to **my IP address only**
- Wazuh agent ↔ manager traffic kept **inside the VPC**
- Wazuh dashboard accessed over **HTTPS**
- **No secrets in this repo:** `.pem` keys, AWS credentials, Wazuh passwords, private keys, certificates, and `wazuh-install-files.tar` were intentionally excluded

---

## 📁 Repository Contents

```text
aws-mini-soc/
├── nginx.conf                    # Web server Nginx configuration
├── security-server/
│   ├── ossec.conf                # Wazuh manager configuration
│   └── setup-notes.md            # Security server setup notes
├── screenshots/                  # Wazuh dashboard screenshots
└── README.md
```

---

## 🚀 Next Steps

- [ ] Add **Fail2Ban** to automatically block IPs after repeated failed SSH logins
- [ ] Turn off `server_tokens` in Nginx so the server doesn't reveal its version
- [ ] Patch the web server and re-scan to compare vulnerability counts
- [ ] Enable **File Integrity Monitoring** and **SCA (configuration assessment)** scans on the agent
- [ ] Write custom Wazuh rules for Nginx attack patterns

---

## 🛠️ Skills Practiced

**Cloud:** AWS EC2 · VPC · Subnets · Route Tables · Internet Gateways · Security Groups · Elastic IPs · EBS volume resizing
**Linux:** Ubuntu administration · Nginx · SSH · Linux logging · Package troubleshooting (`dpkg`)
**Security:** Wazuh SIEM · Intrusion detection · Log analysis · MITRE ATT&CK · Vulnerability detection · IP geolocation
**Tools:** Git · GitHub · Personal Access
