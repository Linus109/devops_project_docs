# Konzept

## Grundlegende Idee

Die grundlegende Idee ist eine CI/CD Pipeline + Workflow für mein "E-Business und Entrepreneurship" Projekt. Dabei handelt es sich eine Art Kalender und Note-Taking App mit KI-Chatbot. Ein Teil des Tech-Stacks für diese App sieht folgendermaßen aus:

- Frontend: React Native
- Backend: Express.js
- Datenbank: MongoDB

Das Projekt liegt in einer [Gitea](https://about.gitea.com/)-Instanz auf einem VPS (Ubuntu 24.04). Für die CI/CD Pipeline wird in Kombination mit Gitea [Drone](https://docs.drone.io/) verwendet.
Grund für die Verwendung von Open-Source Software ist, dass das ganze einfach und kostenlos gehostet werden kann.

> **Hinweis:** Gitea und Drone liefen ursprünglich auf einem Banana Pi BPI-M4 Zero. Aufgrund unzuverlässiger Netzwerkverbindungen (WireGuard-Tunnel brach regelmäßig ab, sehr langsamer upload) wurden sie auf den VPS umgezogen.

## Repositories

| Repository           | URL                                                     | Beschreibung                       |
| -------------------- | ------------------------------------------------------- | ---------------------------------- |
| **CalChat**          | https://gitea.gilmour109.de/Gilmour109/calchat          | Monorepo (Client, Server, Shared)  |
| **CalChat Pipeline** | https://gitea.gilmour109.de/Gilmour109/calchat-pipeline | IaC (OpenTofu, Ansible), Garage S3 |

## Docker, Docker Compose und K3s

Für den Server der Smartphone-App sollen automatisch Docker Images erstellt werden. Die Produktion auf dem VPS wird per Docker Compose deployed, da dort immer nur eine Version läuft. Für Staging auf Proxmox wird [K3s](https://k3s.io/) verwendet, da dort mehrere Versionen parallel laufen können.

## Infrastruktur (IaC)

Die Infrastruktur wird per [OpenTofu](https://opentofu.org/) provisioniert und per [Ansible](https://github.com/ansible/ansible) konfiguriert. Alle lokalen Systeme (Proxmox) sind hinter CGNAT und nur über WireGuard-Tunnel via VPS erreichbar.

### State Management

Der OpenTofu State wird nicht lokal gespeichert, sondern remote in einem selbst gehosteten [Garage](https://garagehq.deuxfleurs.fr/) S3-Backend auf dem VPS (`garage.gilmour109.de`). Garage ist ein leichtgewichtiger, S3-kompatibler Object Store, der als Docker Container läuft. So kann der State von verschiedenen Rechnern aus genutzt werden und geht bei einem lokalen Datenverlust nicht verloren.

## Testserver

Ein Desktop-PC mit [Proxmox](https://proxmox.com/en/) (64GB RAM) dient als Build- und Testserver. Der Drone Runner läuft in einem privilegierten LXC-Container (Debian 13, VMID 200) mit Docker auf Proxmox. Dieser führt die CI/CD-Jobs aus. Zusätzlich läuft eine K3s-VM (Debian 13, VMID 201) für Staging/Preview-Deployments.

### E2E-Tests

E2E-Tests laufen automatisch bei Pushes auf `main` und bei Tags. Pro CI-Run wird per OpenTofu ein Linked Clone aus einem manuell erstellten Debian-Template (mit XFCE-Desktop, Android SDK, Emulator, Appium, Node.js) erzeugt. Auf diesem Clone wird ein Android Emulator gestartet, die App per Expo Go geladen und die Tests mit [Appium](https://appium.io/docs/en/latest/) + WebDriverIO ausgefuehrt. Nach dem Testlauf wird die VM automatisch wieder zerstoert.

- **main-Branch:** Ein temporaeres Test-Backend (CalChat Server + MongoDB) wird als Pod-Set auf K3s deployed. Die E2E-Tests laufen gegen dieses temporaere Backend. Nach dem Test werden die K3s-Ressourcen per Label-Selektor aufgeraeumt.
- **Tags:** Das Backend existiert bereits als K3s-Deployment (aus dem Tag-Deploy-Step). Die E2E-Tests laufen direkt dagegen.
- **Bei Fehlschlag:** Pipeline stoppt, APK wird nicht gebaut.
- **Bei Erfolg:** APK-Build und Release (Gitea Release bei Tags, S3-Upload bei main).

> **Hinweis:** Die E2E-Tests funktionieren aktuell nicht korrekt. Der Testerfolg wird im Script gefaked (`exit 0`), damit die restliche Pipeline getestet werden kann.

## Releases

Mit Hilfe von Drone sollen automatisch Releases erstellt werden, die dann im Anschluss sowohl in Gitea als auch auf einer eigens dafür hergerichteten Website zum Download zur Verfügung stehen: https://home.gilmour109.de/devops. Der Source-Code für letzteres soll natürlich auch in Gitea gehostet werden.

## Übersicht über den Life-Cycle

1. Development:
	- Code wird auf Gitea (VPS) gepusht
	- Gitea triggert Drone Pipeline über Webhook
2. Build und Unit-Tests:
	- Dependencies installieren
	- Unit Tests (Jest)
	- Formatting-Check (Prettier)
	- Bei Fehler: Pipeline wird gestoppt
3. Container erstellen:
	- Docker Image für das Backend bauen
	- Push zur Container Image Registry von Gitea (bei main und Tags)
4. Deployen:
	- main-Branch → VPS (Docker Compose, via SSH) + temporäres Test-Backend auf K3s
	- Tags → Proxmox K3s (envsubst + kubectl apply, via SSH)
5. E2E-Test:
	- Laufen automatisch bei Pushes auf main und bei Tags
	- OpenTofu erstellt einen Linked Clone aus dem E2E-Template auf Proxmox
	- Auf der VM: Android Emulator starten, App per Expo Go laden, Appium + WebDriverIO Tests ausführen
	- main: Tests laufen gegen temporäres K3s-Backend, danach Cleanup (VM + K3s-Ressourcen)
	- Tags: Tests laufen gegen bestehendes K3s-Deployment, danach VM-Cleanup
	- Bei Fehler: Pipeline stoppt (kein APK-Build)
6. Release:
	- Nur bei erfolgreichen E2E-Tests
	- main: APK-Build + Upload zu S3 (Download-Portal)
	- Tags: APK-Build + Gitea Release
