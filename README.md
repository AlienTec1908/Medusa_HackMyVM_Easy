# Medusa - HackMyVM (Easy)

![Medusa.png](Medusa.png)

## Übersicht

*   **VM:** Medusa
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Medusa)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 2023-04-05
*   **Original-Writeup:** https://alientec1908.github.io/Medusa_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser Challenge war es, Root-Rechte auf der Maschine "Medusa" zu erlangen. Der initiale Zugriff erfolgte durch Ausnutzung einer Local File Inclusion (LFI)-Schwachstelle auf einer Webanwendung unter der Subdomain `dev.medusa.hmv`. Diese LFI wurde genutzt, um durch SMTP Log Poisoning eine PHP-Reverse-Shell in die FTP-Logdatei (`vsftpd.log`) zu schreiben und diese dann über die LFI auszuführen, was zu einer Shell als `www-data` führte. Lateral Movement zum Benutzer `spectre` gelang durch das Entpacken eines passwortgeschützten ZIP-Archivs (`old_files.zip`), das einen LSASS-Dump (`lsass.DMP`) enthielt. Mit `pypykatz` wurden aus diesem Dump Klartext-Credentials für `spectre` extrahiert. Die finale Eskalation zu Root erfolgte durch Ausnutzung der Mitgliedschaft des Benutzers `spectre` in der Gruppe `disk`, was den direkten Zugriff auf das Blockgerät (`/dev/sda1`) mit `debugfs` ermöglichte, um die `/etc/shadow`-Datei zu lesen. Der Root-Passwort-Hash konnte anschließend geknackt werden.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `nmap`
*   `lftp`
*   `ftp`
*   `gobuster`
*   `nikto`
*   `showmount`
*   `curl`
*   `hydra`
*   `ssh-keyscan`
*   `wfuzz`
*   `stegsnow`
*   `stegseek`
*   `steghide`
*   `nc` (netcat)
*   Python3 (`pty` Modul)
*   `stty`
*   `find`
*   `wget`
*   `zip2john`
*   `john`
*   `7z`
*   `pypykatz`
*   `ssh`
*   `passwd`
*   `df`
*   `debugfs`
*   Standard Linux-Befehle (`vi`, `echo`, `grep`, `cat`, `su`, `id`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Medusa" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Web Enumeration:**
    *   IP-Adresse des Ziels (192.168.2.118) identifiziert.
    *   `nmap`-Scan offenbarte Port 21 (FTP, vsftpd 3.0.3), 22 (SSH, OpenSSH 8.4p1) und 80 (HTTP, Apache 2.4.54). Anonymer FTP-Login scheiterte.
    *   `gobuster` auf `http://medusa.hmv/` fand u.a. das Verzeichnis `/hades/`.
    *   `wfuzz` zur Subdomain-Enumeration fand `dev.medusa.hmv`.
    *   `gobuster` auf `http://dev.medusa.hmv/` fand u.a. `/files/system.php` und `/files/readme.txt`.
    *   Steganographie-Versuche auf ein Bild (`medusa.jpg` von `dev.medusa.hmv/assets/`) mit `stegsnow` fanden den Hinweis `ssstb`.

2.  **Initial Access (LFI & Log Poisoning als `www-data`):**
    *   `wfuzz` auf `http://dev.medusa.hmv/files/system.php` fand den GET-Parameter `view`.
    *   Die LFI-Schwachstelle wurde mit `curl http://dev.medusa.hmv/files/system.php?view=/etc/passwd` bestätigt, was den Benutzer `spectre` enthüllte.
    *   Mittels FTP (`lftp`) wurde ein fehlgeschlagener Login-Versuch mit einem PHP-Reverse-Shell-Payload im Benutzernamen (z.B. ``) durchgeführt, um diesen in `/var/log/vsftpd.log` zu schreiben.
    *   Die LFI wurde genutzt, um die Logdatei zu inkludieren (`http://dev.medusa.hmv/files/system.php?view=/var/log/vsftpd.log`), was den PHP-Code ausführte.
    *   Eine Reverse Shell als `www-data` wurde auf einem Netcat-Listener empfangen und stabilisiert.

3.  **Lateral Movement (von `www-data` zu `spectre`):**
    *   Die Suche nach beschreibbaren Dateien (`find / -writable ...`) fand u.a. `/.../old_files.zip`.
    *   `old_files.zip` wurde heruntergeladen. Sie war passwortgeschützt.
    *   `zip2john old_files.zip > hash` und `john --wordlist=rockyou.txt hash` knackten das ZIP-Passwort: `medusa666`.
    *   Die ZIP-Datei enthielt `lsass.DMP` (LSASS Speicherabbild).
    *   `pypykatz lsa minidump lsass.DMP` extrahierte Klartext-Credentials für den Benutzer `spectre`: `5p3ctr3_p0is0n_xX`.
    *   Erfolgreicher SSH-Login als `spectre` mit den gefundenen Credentials.

4.  **Privilege Escalation (von `spectre` zu `root` via `debugfs`):**
    *   Als `spectre` zeigte `id`, dass der Benutzer Mitglied der Gruppe `disk` ist.
    *   Mittels `/sbin/debugfs -w /dev/sda1` wurde direkter Zugriff auf das Dateisystem erlangt.
    *   Innerhalb von `debugfs` wurde mit `cat /etc/shadow` der Inhalt der Shadow-Datei ausgelesen.
    *   Der `yescrypt`-Hash (`$y$j9T$...`) für den `root`-Benutzer wurde extrahiert.
    *   `john --wordlist=rockyou.txt --format=crypt <hashdatei>` knackte den Root-Hash. Das Passwort war `andromeda`.
    *   Mit `su root` und dem Passwort `andromeda` wurde Root-Zugriff erlangt.
    *   Die User-Flag (`487a5d1ce02c53fbf60c3abd300d9ff5`) und Root-Flag (`34b1e6fc5e7fe0bfd56ed4b8776c9f5b`) wurden gefunden.

## Wichtige Schwachstellen und Konzepte

*   **Local File Inclusion (LFI):** Eine Webanwendung erlaubte das Einbinden lokaler Dateien über einen GET-Parameter.
*   **Log Poisoning:** Die LFI wurde genutzt, um eine Logdatei (hier `vsftpd.log`) zu inkludieren, in die zuvor bösartiger PHP-Code über einen fehlgeschlagenen FTP-Login geschrieben wurde, was zu RCE führte.
*   **Credential Dumping (LSASS):** Ein LSASS-Speicherabbild (`lsass.DMP`) wurde in einem ZIP-Archiv gefunden und enthielt Klartext-Zugangsdaten.
*   **Unsichere Gruppenmitgliedschaft (Gruppe `disk`):** Die Mitgliedschaft eines normalen Benutzers in der Gruppe `disk` ermöglichte direkten Lese- und Schreibzugriff auf Blockgeräte, was zum Auslesen der `/etc/shadow`-Datei mittels `debugfs` missbraucht wurde.
*   **Passwort-Cracking:** Knacken eines ZIP-Archiv-Passworts und eines `yescrypt`-Passwort-Hashes aus `/etc/shadow`.
*   **Subdomain Enumeration:** Auffinden einer `dev`-Subdomain, die die kritische LFI-Schwachstelle enthielt.

## Flags

*   **User Flag (`/home/spectre/user.txt`):** `487a5d1ce02c53fbf60c3abd300d9ff5`
*   **Root Flag (`/root/.r0t.txt`):** `34b1e6fc5e7fe0bfd56ed4b8776c9f5b`

## Tags

`HackMyVM`, `Medusa`, `Easy`, `LFI`, `Log Poisoning`, `FTP Log Poisoning`, `LSASS Dump`, `pypykatz`, `debugfs`, `disk group exploit`, `Password Cracking`, `yescrypt`, `Linux`, `Web`, `Privilege Escalation`, `Apache`, `vsftpd`
