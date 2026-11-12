# eden-infra

Shared infrastructure for EDEN services in the EOSC EU Node (PSNC OpenStack).

All EDEN service VMs sit on a private network (`eden-net`) behind one entry
VM, `eden-entry` (`62.3.175.206`). It is the only VM with a public IP. It runs
nginx and certbot for every service hostname and a forwarding-only `jump`
account for SSH.

```
Internet --80/443--> eden-entry (62.3.175.206) --HTTP--> service VMs (192.168.0.x)
Admin SSH:  laptop --> jump@62.3.175.206 --> ubuntu@<service-private-IP>
```

## Where to start

- **Deploying a new service behind the entry VM:**
  [docs/entry-vm.md, Part G](docs/entry-vm.md#part-g---deploying-a-new-service)
- **Publishing a service (nginx site file, certificate):**
  [docs/entry-vm.md, Part D](docs/entry-vm.md#part-d---onboarding-a-service)
- **Moving hostnames to the project domain:** [docs/project-domain.md](docs/project-domain.md)

## Contents

| Path                       | What                                                        |
| -------------------------- | ----------------------------------------------------------- |
| `docs/entry-vm.md`         | Entry VM runbook: OpenStack, jump host, nginx, onboarding, operations |
| `docs/project-domain.md`   | Plan for `<service>.services.eden-fidelis.eu` hostnames     |
| `nginx/entry-default.conf` | Catch-all nginx site for the entry VM                       |

Each service keeps its own deployment (Dockerfile, compose file, nginx site
file) in its own repository. For an example, see
[eosc-ffis](https://github.com/EOSC-EDEN/eosc-ffis).
