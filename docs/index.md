Wichtig ist:

- Der Proxmox-Host muss IP-Forwarding aktivieren.
- Es muss eine NAT-Weiterleitung (DNAT) erfolgen.
- Die Rückantworten der VM müssen über den Proxmox-Host zurückgehen (deshalb brauchst du meist MASQUERADE oder SNAT).

---

## 🔧 Szenario:

| Rolle        | IP-Adresse     | Netzmaske | Port     |
|--------------|----------------|-----------|----------|
| Proxmox-Host | 192.168.70.222 | /22       | 8100     |
| VM           | 172.31.98.24   | /24       | z. B. 80 |

Ziel: Verbindungen an 192.168.70.222:8100 sollen an 172.31.98.24:8100 weitergeleitet werden.

---

## 🛠 Schritt-für-Schritt-Anleitung mit iptables

### 1. IP-Forwarding aktivieren

::: success
Eventuell besser Punkt 6 anwenden, falls das hier nicht geht.

:::

Temporär:

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```

Dauerhaft in /etc/sysctl.conf eintragen:

```conf
net.ipv4.ip_forward=1
```

Dann übernehmen mit:

```bash
sysctl -p
```

---

### 2. Portweiterleitung einrichten (iptables NAT-Regeln)

```bash
iptables -t nat -A PREROUTING -p tcp -s 192.168.68.7 -d 192.168.70.222 --dport 8100 -j DNAT --to-destination 172.31.98.24:8100
iptables -t nat -A POSTROUTING -d 172.31.98.24 -p tcp --dport 8100 -j MASQUERADE
```

---

### 3. Regeln testen

Du kannst z. B. von einem anderen Rechner im 192.168.70.0/22-Netz prüfen:

```bash
curl http://192.168.70.222:8100
```

Das sollte (sofern der Dienst läuft) mit der VM sprechen.

---

### 4. Regeln dauerhaft speichern

Installiere iptables-persistent, falls noch nicht installiert:

```bash
apt install iptables-persistent
```

Dann Regeln speichern:

```bash
iptables-save > /etc/iptables/rules.v4
```

---

## 🧠 Wichtig:

- Die VM darf ihre Antwort nicht direkt an den Client schicken, sondern muss durch den Proxmox-Host antworten. Darum ist MASQUERADE nötig.
- Falls du Bridge- oder NAT-Networking verwendest: Die Regeln können sich leicht unterscheiden, je nachdem, wie die VM verbunden ist. Bei Proxmox ist Bridge-Modus (vmbrX) üblich.
- Wenn die VM den Proxmox-Host nicht als Gateway nutzt, ist MASQUERADE besonders wichtig, weil sonst die Antworten ins Leere gehen.

---

### 5. Bash-Skript

Ein einfaches Bash-Skript, das deine gewünschte Portweiterleitung mit iptables auf einem Proxmox-Host einrichtet. Es prüft auch, ob IP-Forwarding aktiv ist, und speichert die Regeln optional dauerhaft mit iptables-persistent.

---

### 📝 Script: port_forwarding.sh

```bash
#!/bin/bash

# Konfiguration
PROXMOX_IP="192.168.70.222"
VM_IP="172.31.98.24"
PORT_HOST=8100
PORT_VM=8100
SOURCE_IP="192.168.68.7" 

echo "🔧 Starte Einrichtung der Portweiterleitung $PROXMOX_IP:$PORT_HOST → $VM_IP:$PORT_VM"

# 1. IP-Forwarding aktivieren (sofort)
echo "✅ Aktiviere IP-Forwarding..."
sysctl -w net.ipv4.ip_forward=1

# 2. Regeln einfügen
echo "✅ Füge iptables-Regeln hinzu..."
iptables -t nat -A PREROUTING -p tcp -s $SOURCE_IP -d $PROXMOX_IP --dport $PORT_HOST -j DNAT --to-destination $VM_IP:$PORT_VM
iptables -t nat -A POSTROUTING -d $VM_IP -p tcp --dport $PORT_VM -j MASQUERADE

# 3. Optionale dauerhafte Speicherung
read -p "💾 Möchtest du die Regeln dauerhaft speichern (iptables-persistent verwenden)? [y/N] " save
if [[ "$save" == "y" || "$save" == "Y" ]]; then
    echo "✅ Speichere Regeln dauerhaft..."
    apt-get install -y iptables-persistent
    iptables-save > /etc/iptables/rules.v4
else
    echo "ℹ️ Regeln bleiben bis zum nächsten Neustart aktiv."
fi

echo "✅ Weiterleitung erfolgreich eingerichtet!"
```

---

### 💡 Verwendung

1. Skript speichern:

   ```bash
   nano port_forwarding.sh
   ```
2. Inhalt einfügen (s. oben), speichern und schließen mit Ctrl+O, Enter, Ctrl+X.
3. Ausführbar machen:

   ```bash
   chmod +x port_forwarding.sh
   ```
4. Skript ausführen mit:

   ```bash
   sudo ./port_forwarding.sh
   ```

---

Wenn du mehrere Weiterleitungen brauchst, kann ich dir das Skript auch so erweitern, dass es mehrere Regeln auf einmal verarbeitet (z. B. aus einer Datei oder Liste). Sag einfach Bescheid!

Alles klar – das bedeutet, dass die Zeile net.ipv4.ip_forward=1 in deiner Datei /etc/sysctl.conf noch nicht vorhanden ist. Kein Problem, du kannst sie einfach hinzufügen. Hier sind die Schritte im Detail:

---

## 6. IP-Forwarding dauerhaft aktivieren

### 1. Öffne die Datei /etc/sysctl.conf mit einem Editor:

```bash
sudo nano /etc/sysctl.conf
```

### 2. Füge am Ende die folgende Zeile hinzu:

```conf
net.ipv4.ip_forward=1
```

### 3. Speichern und schließen (bei nano):

- Ctrl+O (zum Speichern)
- Enter (bestätigen)
- Ctrl+X (Editor schließen)

---

### 4. Neue Einstellung aktivieren:

```bash
sudo sysctl -p
```

Das sollte dann so etwas wie das hier ausgeben:

```
net.ipv4.ip_forward = 1
```

---

Jetzt ist IP-Forwarding dauerhaft aktiviert, auch nach einem Neustart.

### 6.  Löschen der Regeln

Gute Frage! Du kannst iptables-Regeln gezielt löschen, wenn du weißt, wie sie aussehen, oder du kannst sie alle zurücksetzen. Hier zeige ich dir beide Wege – speziell für NAT-Regeln.

---

#### 🧽 Variante 1: Eine spezifische Regel löschen

Wenn du genau weißt, wie die Regel lautet, kannst du sie mit -D (delete) und denselben Parametern wie beim Hinzufügen löschen:

##### Beispiel – PREROUTING-Regel löschen:

```bash
iptables -t nat -D PREROUTING -p tcp -s 192.168.68.7 -d 192.168.70.222 --dport 8100 -j DNAT --to-destination 172.31.98.24:80
```

#### Beispiel – POSTROUTING-Regel löschen:

```bash
iptables -t nat -D POSTROUTING -d 172.31.98.24 -p tcp --dport 80 -j MASQUERADE
```

---

## 📋 Variante 2: Nachsehen, welche Regeln gesetzt sind

Wenn du dir nicht sicher bist, welche Regeln aktiv sind, schau dir die aktuelle NAT-Tabelle an:

```bash
iptables -t nat -L -n -v --line-numbers
```

Beispiel-Ausgabe (gekürzt):

```
Chain PREROUTING (policy ACCEPT)
num  target  prot opt source           destination
1    DNAT    tcp  --  192.168.68.7     192.168.70.222  tcp dpt:8100 to:172.31.98.24:80

Chain POSTROUTING (policy ACCEPT)
num  target     prot opt source        destination
1    MASQUERADE tcp  --  0.0.0.0/0     172.31.98.24    tcp dpt:80
```

---

🎯 Variante 3: Regel per Zeilennummer löschen

Du kannst anhand der Zeilennummer löschen:

```bash
iptables -t nat -D PREROUTING <Nummer>
iptables -t nat -D POSTROUTING <Nummer>
```

Beispiel:

```bash
iptables -t nat -D PREROUTING 1
iptables -t nat -D POSTROUTING 1
```

---

#### 💣 Variante 4: Alle NAT-Regeln löschen (vorsichtig!)

```bash
iptables -t nat -F
```

⚠️ Das löscht alle NAT-Regeln, also auch evtl. andere Weiterleitungen oder SNAT-Regeln.

---

#### 🛠 Regeln dauerhaft löschen (wenn iptables-persistent genutzt)

Wenn du iptables-persistent verwendest, nach dem Löschen speichern:

```bash
iptables-save > /etc/iptables/rules.v4
```

Sonst sind die Regeln beim nächsten Reboot wieder da.

---
