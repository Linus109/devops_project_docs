# Konzept

## Grundlegende Idee

Die grundlegende Idee ist eine vollständige, völlig over-engineerte CI/CD Pipeline + Workflow für mein "E-Business und Entrepreneurship" Projekt. Dabei handelt es sich eine Art Kalender und Note-Taking App mit KI-Chatbot. Ein Teil des Tech-Stacks für diese App sieht folgendermaßen aus:

- Frontend: React Native
- Backend: Express.js
- Datenbank: MongoDB

Das Projekt soll in einer [Gitea](https://about.gitea.com/)-Instanz auf einem [Banana Pi BPI-M4](https://docs.banana-pi.org/en/BPI-M4_Zero/BananaPi_BPI-M4_Zero) Zero liegen. Für die CI/CD Pipeline soll in Kombination mit Gitea [Drone](https://docs.drone.io/) verwendet werden.
Grund für die Verwendung von Open-Source Software ist einfach, dass das ganze einfach und kostenlos zu Hause gehostet werden kann.

## Repositories

| Repository           | URL                                                     | Beschreibung                       |
| -------------------- | ------------------------------------------------------- | ---------------------------------- |
| **CalChat**          | https://gitea.gilmour109.de/Gilmour109/calchat          | Monorepo (Client, Server, Shared)  |
| **CalChat Pipeline** | https://gitea.gilmour109.de/Gilmour109/calchat-pipeline | IaC (OpenTofu, Ansible), Garage S3 |

## Docker, Docker Compose und K3s

Für den Server der Smartphone-App sollen automatisch Docker Images erstellt werden. Die Produktion auf dem VPS wird per Docker Compose deployed — einfach und ausreichend, da dort immer nur eine Version läuft. Für Staging und Preview auf Proxmox wird [K3s](https://k3s.io/) verwendet, da dort mehrere Versionen parallel laufen können (pro Branch ein eigener Namespace).

## Infrastruktur (IaC)

Die Infrastruktur wird per [OpenTofu](https://opentofu.org/) provisioniert und per [Ansible](https://github.com/ansible/ansible) konfiguriert. Der OpenTofu State liegt remote in einem selbst gehosteten [Garage](https://garagehq.deuxfleurs.fr/) S3-Backend auf dem VPS. Alle lokalen Systeme (Pi, Proxmox) sind hinter CGNAT und nur über WireGuard-Tunnel via VPS erreichbar.

## Testserver

Ein Desktop-PC mit [Proxmox](https://proxmox.com/en/) (64GB RAM) dient als Build- und Testserver. Grund für die Aufteilung auf verschiedene Maschinen ist erstmal die Tatsache, dass der Banana Pi nicht so wahnsinnig viel Power hat aber natürlich auch um das ganze ein bisschen spannender zu gestalten.
Der Drone Runner läuft in einem privilegierten LXC-Container (Debian 13) mit Docker auf Proxmox. Dieser führt die CI/CD-Jobs aus. Proxmox wird in Kombination mit Ansible verwendet, um nach dem Infrastructure-as-Code Prinzip Debian-basierte Test-VMs zu starten/erstellen, auf denen E2E-Tests ausgeführt werden.
Mit diesem Ansatz wäre es z.B. möglich mehrere Testserver anzulegen, die dann z.B. verschiedene Versionen gleichzeitig testen.
Die E2E-Tests werden wahrscheinlich entweder [Appium](https://appium.io/docs/en/latest/) oder [Detox](https://github.com/wix/detox/) erstellt und dessen Source-Code ebenfalls auf Gitea gehostet.

## Releases

Mit Hilfe von Drone sollen automatisch Releases erstellt werden, die dann im Anschluss sowohl in Gitea als auch auf einer eigens dafür hergerichteten Website zum Download zur Verfügung stehen. Der Source-Code für letzteres soll natürlich auch in Gitea gehostet werden.

## Übersicht über den Life-Cycle

1. Development:
	- Code wird auf Gitea (Banana Pi) gepushed
	- Gitea triggert Drone Pipeline über Webhook
2. Build und Unit-Tests:
	- Falls notwendig: dependencies installieren
	- Unit Tests (Jest)
	- Bei Fehler: Pipeline wird gestoppt und Entwickler wird benachrichtigt (z.B. per Email)
	- Formatting (Prettier)
3. Container erstellen:
	- Docker Image für das Backend bauen
	- Push zur Container Image Registry von Gitea (zumindest bei den Branches *main* und *tag*)
	- Bei push auf den *dev-Branch* ended die Pipeline in jedem Fall hier
4. Deployen:
	- Server und MongoDB
		- main-Branch → VPS (Docker Compose, via SSH)
		- Feature-Branches → Proxmox (K3s, Namespace `preview-<branch>`)
		- Tags → Proxmox (K3s, Namespace `staging`)
5. E2E-Test:
	- Drone triggert bei Änderungen am *main-Branch* einen E2E-Test
	- Optional kann der Test auch für einen bestimmten Branch oder Tag gestartet werden.
	1. Starten/Erstellen einer VM auf dem Proxmox-Server:
		- Ansible startet oder erstellt Debian-basierte VM
		- VM zieht sich das Apk, als auch den Selenium-Test von Gitea
	2. Durchführung:
		- Android Emulator wird auf der VM gestartet
		- Test wird durchgeführt
		- Bei Fehler: Pipeline wird gestoppt und Entwickler wird benachrichtigt (z.B. per Email)
6. Release:
	- Erstellung von zwei Releases (android, ios) bei Änderungen am *main-Branch* als auch bei der Erstellung eines Tags
	- Releases werden sowohl auf Gitea, also auf einer eigens für die Releases erstellten Website veröffentlicht
