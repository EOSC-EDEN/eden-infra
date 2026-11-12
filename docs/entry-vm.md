# EDEN entry VM

`eden-entry` is the only EDEN VM with a public IP. nginx on `eden-entry`
terminates TLS for every service hostname and proxies the requests to the
service VMs on the private network `eden-net`. Admins connect to the service
VMs over SSH through the `jump` account on `eden-entry`.

```
Internet --80/443--> eden-entry 62.3.175.206 (nginx, certbot, jump)
                       |  HTTP over eden-net 192.168.0.0/24
                       +--> ffis        192.168.0.203:8000
                       +--> <service>   192.168.0.x:<port>

SSH: laptop --> jump@62.3.175.206 --> ubuntu@192.168.0.x
```

- Deploying a new service: [Part A](#part-a---deploying-a-new-service)
- Publishing a service on the entry VM: [Part B](#part-b---publishing-a-service)

## Live values

| Item                 | Value                      |
| -------------------- | -------------------------- |
| Entry VM name        | `eden-entry`               |
| Entry VM private IP  | `192.168.0.111`            |
| Entry VM flavour     | `M1-NET-1vCPU-4R`          |
| Floating IP          | `62.3.175.206`             |
| Login user           | `ubuntu`                   |
| Jump user            | `jump`                     |
| Network              | `eden-net`                 |
| Subnet               | `192.168.0.0/24`           |
| Router               | `eden-router`              |
| External network     | `PSNC-EXT-PUB1-EDU`        |
| Certificate e-mail   | mattias.levlin@csc.fi      |
| OpenStack dashboard  | https://eu-1.iaas.open-science-cloud.ec.europa.eu/ |

| Service | VM     | Private IP      | Port | Hostname                   |
| ------- | ------ | --------------- | ---- | -------------------------- |
| FFIS    | `ffis` | `192.168.0.203` | 8000 | `eden-ffis.vm.fedcloud.eu` |

## People

| Role                                 | Who                     |
| ------------------------------------ | ----------------------- |
| Entry VM owner                       | Mattias Levlin (CSC)    |
| Entry VM backup owner                | Wesley Middelbos (CERN) |
| Entry VM admin                       | Svein H.                |
| Entry VM admin                       | Laurens Tobias (DANS)   |
| Service VM, hostname and site file   | The service team        |
| `eden-fidelis.eu` domain             | Trust-IT, see [project-domain.md](project-domain.md) |

Owner, backup owner and admins have sudo on `eden-entry` and install site
files and certificates.

## Conventions

- Every service has one nginx site file, `/etc/nginx/sites-available/<service>`.
  The service team writes the site file and an admin installs the site file.
- Every service has a separate certificate. Adding or removing a service
  leaves the other certificates unchanged.
- `limit_req_zone` names start with the service name, for example
  `ffis_identify`. All zones are defined in the shared `http` context, so two
  services with the same zone name break the nginx config.
- Run `sudo nginx -t` before every reload. A syntax error in one site file
  blocks reloads for all services.

---

## Part A - Deploying a new service

Steps marked **(admin)** are done by an entry VM admin. The service team does
the other steps.

1. **Project access.** Ask for membership in the EDEN project on the
   [OpenStack dashboard](https://eu-1.iaas.open-science-cloud.ec.europa.eu/).
2. **SSH key.** Send your SSH public key to an admin.
3. **(admin) Add the key to `jump`**, with the name in the comment:

   ```bash
   ssh ubuntu@62.3.175.206
   echo '<public-key> <name, org>' | sudo tee -a /home/jump/.ssh/authorized_keys
   ```

4. **Allow your IP.** Find your IPv4 address with `curl -4 ifconfig.me`. In
   the dashboard, open **Network -> Security Groups -> eden-entry -> Manage
   Rules -> Add Rule** and add port 22 from CIDR `<your-IPv4>/32`, with your
   name as the description. Add a new rule when your IP address changes.
5. **Backend port.** For a port other than 8000, add a rule to `eden-service`:
   port `<port>`, **Remote: Security Group**, **Security Group: eden-entry**.
6. **Launch the VM** in **Compute -> Instances -> Launch Instance**:

   | Field          | Value                              |
   | -------------- | ---------------------------------- |
   | Source         | `Ubuntu 24.04 LTS`                 |
   | Network        | `eden-net`                         |
   | Security group | `eden-service` (remove `default`)  |
   | Key pair       | your own                           |
   | Floating IP    | none                               |

   Note the private IP. Bind the service to the private IP. A service bound
   to `0.0.0.0` listens on every interface, and a service bound to loopback
   is unreachable from the entry VM.
7. **Log in** through the entry VM:

   ```bash
   ssh -J jump@62.3.175.206 ubuntu@<private-IP>
   ```

   `apt` and `docker pull` work through the router.
8. **Hostname.** Log in at https://nsupdate.fedcloud.eu/ with EGI Check-in,
   add the host `eden-<service>` in the domain `vm.fedcloud.eu`, and keep the
   secret. Point the hostname at the entry VM:

   ```bash
   curl "https://<hostname>:<secret>@nsupdate.fedcloud.eu/nic/update?myip=62.3.175.206"
   dig +short <hostname>    # 62.3.175.206
   ```

   Always pass `myip`. Without `myip`, nsupdate records the source address of
   the request, and on a service VM the source address is the router address.
9. **Send an admin** the hostname, `<private-IP>:<port>` and an nginx site
   file. Use [`nginx-ffis.conf`](https://github.com/EOSC-EDEN/eden-ffis/blob/main/deploy/nginx-ffis.conf)
   as the template: set `client_max_body_size`, `proxy_read_timeout` and the
   `<service>_` prefix on the rate limit zone.
10. **(admin) Publish the service** as in [Part B](#part-b---publishing-a-service).

Optional `~/.ssh/config` entry:

```
Host eden-entry
    HostName 62.3.175.206
    User jump

Host eden-<service>
    HostName <private-IP>
    User ubuntu
    ProxyJump eden-entry
```

Do not use agent forwarding (`ssh -A`). `ProxyJump` works without an agent on
the entry VM.

---

## Part B - Publishing a service

Run on `eden-entry` as `ubuntu`. The commands use FFIS as the example.

### B1. Check DNS and the backend

```bash
dig +short eden-ffis.vm.fedcloud.eu        # 62.3.175.206
curl -s http://192.168.0.203:8000/health
```

If `curl` times out, `eden-service` lacks a rule for the backend port. Wait
for the correct DNS answer before B3: certbot fails on a wrong answer, and
failed attempts count against the Let's Encrypt rate limits.

### B2. Install the site file

Fill in the hostname and the backend IP in the site file first.

```bash
sudo cp nginx-ffis.conf /etc/nginx/sites-available/ffis
sudo ln -sf /etc/nginx/sites-available/ffis /etc/nginx/sites-enabled/ffis
sudo nginx -t && sudo systemctl reload nginx
```

nginx defaults to a 1 MB upload limit and a 60 s read timeout. A site file
with only `proxy_pass` returns 413 for FFIS uploads over 1 MB.

### B3. Certificate

```bash
sudo certbot certonly --nginx -d eden-ffis.vm.fedcloud.eu --dry-run
sudo certbot --nginx -d eden-ffis.vm.fedcloud.eu \
  --agree-tos --no-eff-email -m mattias.levlin@csc.fi --redirect
```

### B4. Verify from outside

```bash
curl -I http://eden-ffis.vm.fedcloud.eu/         # 301 to https
curl -s https://eden-ffis.vm.fedcloud.eu/health
```

### B5. Removing a service

```bash
sudo rm /etc/nginx/sites-enabled/<service> /etc/nginx/sites-available/<service>
sudo certbot delete --cert-name <hostname>
sudo nginx -t && sudo systemctl reload nginx
```

The service team deletes the host in nsupdate.

---

## Part C - Building the entry VM

The entry VM exists (see [Live values](#live-values)). Part C is the record
for a rebuild.

### C1. Network and router

1. **Network -> Networks -> Create Network**: name `eden-net`, subnet
   `eden-subnet`, network address `192.168.0.0/24`.
2. **Network -> Routers -> Create Router**: name `eden-router`, external
   network `PSNC-EXT-PUB1-EDU`. `PSNC-EXT-IPV6-PUB2-EDU` is IPv6 only.
3. Open `eden-router` -> **Interfaces -> Add Interface** -> `eden-subnet`.

Outbound traffic from the service VMs is sent through the router's external
gateway (SNAT). Keep SNAT enabled.

### C2. Security groups

Internal rules use a remote security group instead of a CIDR, so private IPs
are not listed in the rules.

`eden-entry`, on the entry VM only:

| Port | Remote              | Purpose                                 |
| ---- | ------------------- | --------------------------------------- |
| 22   | `<admin-IPv4>/32`   | SSH, one rule for every admin location  |
| 22   | `193.166.83.240/28` | SSH from CSC                            |
| 80   | `0.0.0.0/0`         | Let's Encrypt HTTP-01, redirect to 443  |
| 443  | `0.0.0.0/0`         | All services                            |

Never open port 22 to `0.0.0.0/0`.

`eden-service`, on every service VM:

| Port     | Remote                     | Purpose                    |
| -------- | -------------------------- | -------------------------- |
| 22       | security group `eden-entry` | SSH through the entry VM  |
| 8000     | security group `eden-entry` | FFIS backend              |
| `<port>` | security group `eden-entry` | Other backends            |

### C3. Launch

| Field          | Value                             |
| -------------- | --------------------------------- |
| Name           | `eden-entry`                      |
| Source         | `Ubuntu 24.04 LTS`                |
| Volume         | 20 GB                             |
| Flavour        | `M1-NET-1vCPU-4R`                 |
| vCPU           | 1                                 |
| RAM            | 4 GB                              |
| Network        | `eden-net`                        |
| Security group | `eden-entry` (remove `default`)   |
| Key pair       | the owner's                       |

nginx buffers uploads on the entry VM (up to 100 MB for FFIS), so 4 GB RAM
is the minimum. Associate the floating IP in **Network -> Floating IPs**,
pool `PSNC-EXT-PUB1-EDU`.

```bash
sudo apt update && sudo apt -y upgrade
sudo apt -y install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

### C4. Jump account

The `jump` account forwards SSH connections and has no shell and no sudo.

```bash
sudo adduser --disabled-password --gecos "" jump
sudo install -d -m 700 -o jump -g jump /home/jump/.ssh
sudo install -m 600 -o jump -g jump /dev/null /home/jump/.ssh/authorized_keys

sudo tee /etc/ssh/sshd_config.d/50-jump.conf >/dev/null <<'EOF'
Match User jump
    PermitTTY no
    X11Forwarding no
    AllowAgentForwarding no
    AllowTcpForwarding yes
    ForceCommand /usr/sbin/nologin
EOF
sudo sshd -t && sudo systemctl reload ssh
```

### C5. nginx and certbot

```bash
sudo apt -y install nginx
sudo systemctl enable --now nginx

sudo cp nginx/entry-default.conf /etc/nginx/sites-available/00-default
sudo ln -sf /etc/nginx/sites-available/00-default /etc/nginx/sites-enabled/00-default
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx

sudo snap install core; sudo snap refresh core
sudo snap install --classic certbot
sudo ln -sf /snap/bin/certbot /usr/local/bin/certbot
systemctl list-timers | grep -i certbot       # renewal timer
```

With `entry-default.conf`, nginx closes connections for unknown hostnames.
`curl -I http://62.3.175.206/` returns an empty reply.

---

## Part D - Operations

- **Single point of failure.** When `eden-entry` is down, all EDEN services
  are unreachable from the internet. Accepted for the pilot phase.
- **Backups.** `/etc/nginx/sites-available/` and `/etc/letsencrypt/`. Part C
  covers the rest.
- **Logs.** `/var/log/nginx/<service>.access.log` and
  `/var/log/nginx/<service>.error.log` on `eden-entry`.
- **Site file changes.** The service team changes the site file in its own
  repository, and an admin installs the new version (B2).

| Symptom                               | Cause                                                        |
| ------------------------------------- | ------------------------------------------------------------ |
| Empty reply                           | Hostname missing from every `server_name`                    |
| 502                                   | Backend down, or bound to loopback                           |
| 504, or `curl` to the backend times out | `eden-service` lacks the backend port from `eden-entry`    |
| certbot HTTP-01 fails                 | Hostname resolves to another IP than `62.3.175.206`          |
| Hostname resolves to the router IP    | nsupdate was called from a service VM without `myip`         |
| SSH hangs at `jump@62.3.175.206`      | Your IPv4 address is missing from `eden-entry`               |
| SSH fails at the second hop           | `eden-service` lacks port 22, or wrong user or key           |
| `Too many authentication failures`    | The SSH agent offers too many keys: use `-i <key> -o IdentitiesOnly=yes` |
| 413 on upload                         | `client_max_body_size` missing or too low                    |

---

## Part E - History and open tasks

On 2026-11-05 the floating IP `62.3.175.206` was moved from the standalone
FFIS VM (`ffis-standalone`) to `eden-entry`, and FFIS was moved to the service
VM `ffis`. `ffis-standalone` is shut off.

Open tasks:

1. Delete the temporary `eden-entry` rule: port 22 from security group
   `ffis-web`.
2. Delete the `ffis-standalone` instance, its volume, and then the security
   group `ffis-web`.
3. Harvester: `eden-harvester.vm.fedcloud.eu` resolves to `62.3.175.110`, a
   floating IP in another OpenStack project. Recommended: the harvester team
   rebuilds the harvester as a service VM in the EDEN project (Part A), which
   frees `62.3.175.110`.
4. Project domain: see [project-domain.md](project-domain.md).
