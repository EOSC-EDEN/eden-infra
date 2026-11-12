# EDEN project domain

Plan to move EDEN hostnames from `eden-<service>.vm.fedcloud.eu` to
`<service>.services.eden-fidelis.eu`. The service VMs stay unchanged; only
DNS, nginx and the certificates on the entry VM change.

## Status (2026-11-02)

| Step                                  | Status                                       | Date       |
| ------------------------------------- | -------------------------------------------- | ---------- |
| 1. Trust-IT adds the `NS` records     | Done                                         | 2026-10-19 |
| 2. EGI creates the project domain     | To do: EGI name servers answer `REFUSED`     |            |
| 3. Verify the delegation              | To do                                        |            |
| 4. Service teams register their hosts | To do                                        |            |
| 5. Entry VM serves both names         | To do                                        |            |
| 6. Switch to the new names            | To do                                        |            |

`eden-fidelis.eu` has zero CAA records, so Let's Encrypt can issue
certificates for the new names.

## Setup

```
eden-fidelis.eu            Trust-IT
 +-- services              NS -> ns02, ns03, ns04.fedcloud.eu
      +-- ffis             A -> 62.3.175.206 (nsupdate, FFIS team)
      +-- harvester        A -> 62.3.175.206 (nsupdate, harvester team)
```

EGI hosts `services.eden-fidelis.eu` on nsupdate.fedcloud.eu as a project
domain. Service teams register hosts in the new domain in the same way as in
`vm.fedcloud.eu`, with one secret for every host.

| Role                        | Who                                              |
| --------------------------- | ------------------------------------------------ |
| `eden-fidelis.eu` DNS       | Trust-IT: Luca di Fiore, Alessandro Petroceli    |
| Trust-IT contact            | Rita Meneses                                     |
| EGI Dynamic DNS             | EGI Helpdesk                                     |
| Helpdesk ticket, switch     | Mattias Levlin (CSC)                             |
| Hosts in the new domain     | The service teams                                |

## Open questions

1. **Domain lifetime.** All names under `eden-fidelis.eu` stop resolving if
   the domain lapses. Find out who pays for the renewal and until which date,
   before the names are used in papers, deliverables or registries.
2. **nsupdate managers.** At least two EGI Check-in accounts should manage the
   domain.

## Steps

### 1. NS records (done 2026-10-19)

```
services.eden-fidelis.eu.   3600  IN  NS  ns02.fedcloud.eu.
services.eden-fidelis.eu.   3600  IN  NS  ns03.fedcloud.eu.
services.eden-fidelis.eu.   3600  IN  NS  ns04.fedcloud.eu.
```

### 2. EGI Helpdesk ticket

At https://helpdesk.egi.eu/, support unit **Dynamic DNS**:

> Subject: Project domain services.eden-fidelis.eu on nsupdate.fedcloud.eu
>
> Hello,
>
> The EDEN project runs services in the EOSC EU Node (PSNC) with hostnames
> under vm.fedcloud.eu. We would like the project domain
> services.eden-fidelis.eu on nsupdate.fedcloud.eu.
>
> Trust-IT, who manage eden-fidelis.eu, have delegated
> services.eden-fidelis.eu to ns02, ns03 and ns04.fedcloud.eu. Those servers
> answer REFUSED for the zone. Could you set up the domain, managed by the
> following EGI Check-in accounts: <list>?
>
> Thank you,
> <name>, on behalf of the EDEN project

### 3. Verify the delegation

```bash
dig NS services.eden-fidelis.eu +short     # ns02, ns03, ns04.fedcloud.eu
dig +trace services.eden-fidelis.eu        # path ends at the EGI name servers
```

The domain is listed under your account in the nsupdate portal.

### 4. Register the hosts

```bash
curl "https://ffis.services.eden-fidelis.eu:<secret>@nsupdate.fedcloud.eu/nic/update?myip=62.3.175.206"
dig +short ffis.services.eden-fidelis.eu   # 62.3.175.206
```

| Service   | Current name               | New name                             |
| --------- | -------------------------- | ------------------------------------ |
| FFIS      | `eden-ffis.vm.fedcloud.eu` | `ffis.services.eden-fidelis.eu`      |
| Harvester | `eden-harvester.vm.fedcloud.eu` | `harvester.services.eden-fidelis.eu` |

The new names drop the `eden-` prefix.

### 5. Serve both names on the entry VM

```bash
sudo sed -i 's/server_name eden-ffis\.vm\.fedcloud\.eu;/server_name eden-ffis.vm.fedcloud.eu ffis.services.eden-fidelis.eu;/' \
  /etc/nginx/sites-available/ffis
sudo nginx -t && sudo systemctl reload nginx

sudo certbot --nginx --cert-name eden-ffis.vm.fedcloud.eu \
  -d eden-ffis.vm.fedcloud.eu -d ffis.services.eden-fidelis.eu

curl -s https://ffis.services.eden-fidelis.eu/health
```

### 6. Switch

1. Announce the new names and update the links in documentation and
   configuration.
2. After a transition period, redirect the old name with
   `return 301 https://ffis.services.eden-fidelis.eu$request_uri;` in a
   server block for the old name.
3. Remove the old name from the certificate and delete the old host in
   nsupdate.

Until step 3, the old names keep working as a rollback.
