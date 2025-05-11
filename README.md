# Doll - HackMyVM (Easy)

![Doll.png](Doll.png)

## Übersicht

*   **VM:** Doll
*   **Plattform:** [HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=Doll)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 28. April 2023
*   **Original-Writeup:** https://alientec1908.github.io/Doll_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser "Easy"-Challenge war es, Root-Zugriff auf der Maschine "Doll" zu erlangen. Die Enumeration deckte eine offene Docker Registry auf Port 1007 auf. Durch Analyse des Manifests des `dolly:latest`-Images wurde ein Passwort (`devilcollectsit`) in der Build-History gefunden. Die Analyse der Image-Layer enthüllte einen privaten SSH-Schlüssel für den Benutzer `bela`. Mit der Kombination aus Schlüssel und Passwort (als Passphrase) wurde initialer Zugriff als `bela` via SSH erlangt. Als `bela` wurde eine `sudo`-Regel entdeckt, die das Ausführen von `/usr/bin/fzf --listen=1337` als Root ohne Passwort erlaubte. Durch SSH Local Port Forwarding wurde dieser `fzf`-Listener vom Angreifer-System aus erreichbar gemacht. Ein `curl`-Befehl an diesen Listener führte dazu, dass `fzf` als Root `chmod +s /bin/bash` ausführte. Anschließend konnte mit `/bin/bash -p` eine Root-Shell erlangt und die Flags gelesen werden.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `nikto`
*   `gobuster`
*   `curl`
*   `wget`
*   `jq` (impliziert für JSON-Verarbeitung)
*   `grep`, `awk`, `tr` (für Textmanipulation)
*   `file`
*   `tar`
*   `ssh`
*   `sudo` (auf Zielsystem)
*   `fzf` (als Exploit-Vektor)
*   Standard Linux-Befehle (`cd`, `ls`, `chmod`, `id`, `cat`, `bash`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Doll" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Docker Registry Enumeration:**
    *   IP-Findung mittels `arp-scan` (Ziel: `192.168.2.119`, Hostname `doll.hmv`).
    *   `nmap`-Scan identifizierte SSH (22/tcp) und eine Docker Registry API (1007/tcp).
    *   `nikto` und `curl` bestätigten die Docker Registry und fanden den Katalog-Endpunkt (`/v2/_catalog`), der das Repository `dolly` enthielt.
    *   Der Tag `latest` für das `dolly`-Repository wurde über `/v2/dolly/tags/list` identifiziert.

2.  **Finding Credentials (Manifest & Layer Analysis):**
    *   Das Manifest für `dolly:latest` wurde via `curl http://doll.hmv:1007/v2/dolly/manifests/latest` heruntergeladen.
    *   Im `history`-Abschnitt des Manifests wurde das Passwort `devilcollectsit` (als `ARG passwd=...`) gefunden.
    *   Die `blobSum`-Hashes der Image-Layer wurden aus dem Manifest extrahiert.
    *   Die Layer wurden mit `wget` heruntergeladen.
    *   Einer der Layer (ein `tar.gz`-Archiv) wurde entpackt und enthielt den privaten SSH-Schlüssel `/home/bela/.ssh/id_rsa`.

3.  **Initial Access (als `bela` via SSH):**
    *   Der extrahierte private SSH-Schlüssel (`id_rsa`) wurde mit `chmod 600` versehen.
    *   Ein SSH-Login als Benutzer `bela` (`ssh bela@doll.hmv -i id_rsa`) war erfolgreich, wobei das zuvor gefundene Passwort `devilcollectsit` als Passphrase für den Schlüssel verwendet wurde.

4.  **Privilege Escalation (von `bela` zu `root` via `sudo fzf`):**
    *   Als `bela` zeigte `sudo -l`, dass `/usr/bin/fzf --listen=1337` als `root` (`ALL`) ohne Passwort (`NOPASSWD:`) ausgeführt werden durfte.
    *   Eine neue SSH-Verbindung wurde mit Local Port Forwarding aufgebaut: `ssh -i id_rsa bela@doll.hmv -L 1337:127.0.0.1:1337`.
    *   In dieser SSH-Sitzung wurde `sudo /usr/bin/fzf --listen=1337` gestartet.
    *   Auf dem Angreifer-System wurde `curl -X POST http://localhost:1337 -d 'execute(chmod +s /bin/bash)'` ausgeführt. Dies wies `fzf` an, `/bin/bash` SUID zu machen.
    *   Zurück in der SSH-Sitzung wurde `/bin/bash -p` ausgeführt, was eine Shell mit Root-Rechten (`euid=0(root)`) lieferte.
    *   Die User- und Root-Flags wurden gelesen.

## Wichtige Schwachstellen und Konzepte

*   **Offene Docker Registry:** Erlaubte die Enumeration von Repositories und das Herunterladen von Images.
*   **Information Disclosure in Docker Manifest/Layern:**
    *   Passwort (`devilcollectsit`) als Build-Argument im Manifest.
    *   Privater SSH-Schlüssel (`id_rsa` für `bela`) in einem Image-Layer.
*   **Unsichere `sudo`-Regel (`fzf --listen`):** Das Erlauben von `fzf` mit der `--listen`-Option als Root ist eine bekannte Methode zur Privilegieneskalation, da es die Ausführung beliebiger Befehle über eine Netzwerkschnittstelle ermöglicht.
*   **SSH Local Port Forwarding:** Genutzt, um den von `fzf` auf dem Zielsystem gestarteten Listener vom Angreifer-System aus erreichbar zu machen.

## Flags

*   **User Flag (`/home/bela/user.txt`):** `juHDnnGMYNIkVgfnMV`
*   **Root Flag (`/root/root.txt`):** `xwHTSMZljFuJERHmMV`

## Tags

`HackMyVM`, `Doll`, `Easy`, `Docker Registry`, `Information Disclosure`, `SSH Private Key`, `Password Leak`, `sudo`, `fzf`, `Privilege Escalation`, `Linux`
