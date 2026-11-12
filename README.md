# eden-infra

Shared infrastructure for EDEN services in the EOSC EU Node (PSNC OpenStack).

All EDEN service VMs are on the private network `eden-net`, behind the entry
VM `eden-entry` (`62.3.175.206`). nginx and certbot on `eden-entry` serve
every service hostname, and the `jump` account forwards SSH connections to the
service VMs.

| Task                                | Document                                                   |
| ----------------------------------- | ---------------------------------------------------------- |
| Deploy a new service                | [entry-vm.md, Part A](docs/entry-vm.md#part-a---deploying-a-new-service) |
| Publish a service on the entry VM   | [entry-vm.md, Part B](docs/entry-vm.md#part-b---publishing-a-service) |
| Move to `services.eden-fidelis.eu`  | [project-domain.md](docs/project-domain.md)                |

| Path                       | Content                         |
| -------------------------- | ------------------------------- |
| `docs/entry-vm.md`         | Entry VM runbook                |
| `docs/project-domain.md`   | Project domain plan             |
| `nginx/entry-default.conf` | Catch-all nginx site            |

Every service keeps its own deployment files (Dockerfile, compose file, nginx
site file) in its own repository, for example
[eden-ffis](https://github.com/EOSC-EDEN/eden-ffis).
