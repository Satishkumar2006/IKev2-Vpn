# IKEv2 VPN on Google Cloud using strongSwan

**Author:** SATISH KUMAR S
**Level:** Beginner – Step-by-Step
**Platform:** Ubuntu 24.04 LTS (Google Cloud VM)
**Method:** Certificate (MachineCert) Authentication only
**Access Interface:** Google Cloud SSH Web Console (recommended)

---

## 1. Introduction

### What is IKEv2 VPN?

IKEv2 (Internet Key Exchange version 2) is a secure, efficient VPN protocol. It’s ideal for modern clients (Windows, Android, iOS) and automatically reconnects when switching networks.

### Why use strongSwan + Google Cloud?

* Open-source, secure, and stable.
* Works with built-in VPN clients.
* No third-party client apps required.
* Perfect for personal or small business private VPNs.

---

## 2. Google Cloud VM Setup

### Create VM

1. Go to **Google Cloud Console → Compute Engine → VM Instances → Create Instance**.
2. Choose:

   * **Region:** Closest to you
   * **Machine type:** e2-micro (Free tier)
   * **OS Image:** Ubuntu 24.04 LTS
3. Click **Allow HTTP/HTTPS traffic**.

### Firewall Rules

Allow VPN ports:

```bash
sudo gcloud compute firewall-rules create ikev2-udp \
  --allow udp:500,udp:4500 --direction=INGRESS --priority=1000 \
  --network=default --target-tags=ikev2-vpn
```

Then tag your instance:

```bash
gcloud compute instances add-tags <INSTANCE_NAME> --tags=ikev2-vpn
```

### Enable IP Forwarding

```bash
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

**Expected Output:**

```
net.ipv4.ip_forward = 1
```

---

## 3. Install strongSwan

```bash
sudo apt update && sudo apt install -y strongswan strongswan-pki libcharon-extra-plugins
```

**Expected Output:**

```
Starting strongSwan IPsec... done.
```

---

## 4. Certificate Authority and Server Certificates

### Create Directory Structure

```bash
sudo mkdir -p /etc/ipsec.d/pki/{cacerts,certs,private}
cd /etc/ipsec.d/pki
```

### Root CA

```bash
sudo ipsec pki --gen --type rsa --size 4096 --outform pem > private/ca-key.pem
sudo ipsec pki --self --ca --lifetime 3650 --in private/ca-key.pem --type rsa \
  --dn "CN=IKEv2-CA" --outform pem > cacerts/ca-cert.pem
```

### Server Certificate

```bash
sudo ipsec pki --gen --type rsa --size 4096 --outform pem > private/server-key.pem
sudo ipsec pki --pub --in private/server-key.pem --type rsa \
  | sudo ipsec pki --issue --lifetime 3650 \
    --cacert cacerts/ca-cert.pem --cakey private/ca-key.pem \
    --dn "CN=<YOUR_VM_EXTERNAL_IP>" --san "<YOUR_VM_EXTERNAL_IP>" \
    --flag serverAuth --outform pem > certs/server-cert.pem
```

**Example Output:**

```
issued certificate 'CN=<YOUR_VM_EXTERNAL_IP>' valid from 2025-11-06 08:00:00 to 2035-11-03 08:00:00
```

---

## 5. Generate Client Certificate

```bash
sudo ipsec pki --gen --type rsa --size 2048 --outform pem > private/client1-key.pem
sudo ipsec pki --pub --in private/client1-key.pem --type rsa \
  | sudo ipsec pki --issue --lifetime 3650 \
    --cacert cacerts/ca-cert.pem --cakey private/ca-key.pem \
    --dn "CN=client1" --san "client1" --flag clientAuth --outform pem > certs/client1-cert.pem
```

Package it for Windows/Android:

```bash
sudo openssl pkcs12 -export -inkey private/client1-key.pem \
  -in certs/client1-cert.pem -certfile cacerts/ca-cert.pem \
  -name "client1" -out /home/<user>/client1.p12
```

**Example Output:**

```
Enter Export Password:
Verifying - Enter Export Password:
```

---

## 6. Server Configuration

### Edit `/etc/ipsec.conf`

```bash
config setup
  charondebug="cfg,ike,knl,net"
  uniqueids=no

conn ikev2-vpn
  keyexchange=ikev2
  ike=aes256-sha256-modp1024,aes128-sha256-modp1024,aes256-sha1-modp1024!
  esp=aes256-sha256,aes256-sha1!
  dpdaction=clear
  dpddelay=300s
  rekey=no
  left=%any
  leftid=<YOUR_VM_EXTERNAL_IP>
  leftcert=server-cert.pem
  leftsendcert=always
  leftsubnet=0.0.0.0/0
  right=%any
  rightid=%any
  rightauth=pubkey
  rightsourceip=10.10.10.0/24
  rightdns=1.1.1.1,8.8.8.8
  auto=add
```

### Edit `/etc/ipsec.secrets`

```bash
: RSA "server-key.pem"
```

Restart service:

```bash
sudo systemctl restart strongswan-starter
sudo systemctl enable strongswan-starter
sudo systemctl status strongswan-starter
```

**Expected Output:**

```
Starting strongSwan IKEv2 daemon...  [OK]
```

---

## 7. IP Tables and NAT Configuration

Enable forwarding through your main NIC (`ens4`):

```bash
sudo iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o ens4 -j MASQUERADE
sudo iptables -I FORWARD 1 -s 10.10.10.0/24 -o ens4 -m conntrack --ctstate NEW,ESTABLISHED,RELATED -j ACCEPT
sudo iptables -I FORWARD 2 -d 10.10.10.0/24 -i ens4 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

Verify:

```bash
sudo iptables -t nat -L POSTROUTING -n -v | grep 10.10.10.0
```

**Expected Output:**

```
MASQUERADE  all  --  10.10.10.0/24  0.0.0.0/0
```

---

## 8. Client Setup

### Windows 10/11

1. Open **Settings → Network & Internet → VPN → Add VPN**
2. VPN Provider: *Windows (built-in)*
3. VPN Type: *IKEv2*
4. Authentication: *Machine Certificate*
5. Server name: `<YOUR_VM_EXTERNAL_IP>`
6. Click **Save → Connect**

**Expected:** Connected, Status: *IKEv2 VPN Connected*

### Android/iOS

1. Copy `client1.p12` and `ca-cert.pem` to your device.
2. Import via Settings → VPN → Add IKEv2.
3. Choose *Certificate authentication* → Select `client1`.

---

## 9. Testing

### From Windows

```powershell
ping 8.8.8.8 -n 5
ping google.com -n 5
```

**Expected Output:**

```
Reply from 8.8.8.8: bytes=32 time=60ms TTL=121
Ping request could not find host google.com. (Fix: add DNS)
```

---

## 10. Troubleshooting

| Problem                                         | Cause                 | Fix                              |
| ----------------------------------------------- | --------------------- | -------------------------------- |
| AUTH_FAILED                                     | Wrong or missing cert | Check CA and client certs        |
| NO_PROPOSAL_CHOSEN                              | Cipher mismatch       | Match `ike` and `esp` proposals  |
| No Internet                                     | Missing NAT or DNS    | Verify `iptables` and `rightdns` |
| Windows shows “IKE authentication unacceptable” | Wrong auth type       | Ensure Machine Cert selected     |

---

## 11. Backup and Maintenance

### Backup all certs

```bash
sudo tar -czf /home/<user>/ikev2-certs-$(date +%F).tar.gz \
  /etc/ipsec.d/pki/cacerts/ca-cert.pem \
  /etc/ipsec.d/pki/certs/server-cert.pem \
  /etc/ipsec.d/pki/private/server-key.pem \
  /etc/ipsec.d/pki/certs/client1-cert.pem \
  /etc/ipsec.d/pki/private/client1-key.pem \
  /home/<user>/client1.p12
```

Download using SSH console’s **Download file** option or via CLI:

```bash
gcloud compute scp <INSTANCE_NAME>:/home/<user>/client1.p12 ./
```

---

## 12. Professional Network Diagram (Summary)

```
[ Windows / Android / iOS Clients ]
          🔒
     (IKEv2 Encrypted Tunnel)
          🔒
   [ GCP VM – strongSwan Server ]
          ↕  (NAT + Routing)
       Internet Access
```

---

## 13. Appendix

### File Paths

```
/etc/ipsec.d/pki/
├── cacerts/ca-cert.pem
├── certs/server-cert.pem
├── certs/client1-cert.pem
├── private/server-key.pem
└── private/client1-key.pem
```

### Common strongSwan plugins

| Plugin         | Function                      |
| -------------- | ----------------------------- |
| eap-mschapv2   | EAP authentication            |
| x509           | Certificate parsing           |
| kernel-netlink | Linux kernel IPsec interface  |
| attr           | Handles virtual IP assignment |

---

**✅ Setup Complete!**
Your IKEv2 VPN server is now fully functional and supports multiple clients securely using certificate-based authentication.

---

© 2025 SATISH KUMAR S – All Rights Reserved.
