# Exercice 3 — DHCP : obtenir une adresse IP automatiquement

**Niveau :** ⭐⭐ · **Durée :** ~1 h
**📖 Fiche théorique à lire avant :** `https://<votre-site>.pythonanywhere.com/dhcp`

## 🎯 Objectifs

- Constater qu'une machine **sans adresse IP** ne peut pas communiquer.
- Capturer les **4 messages DHCP** (Discover, Offer, Request, ACK = « DORA »).
- Retrouver dans ces messages la configuration reçue : IP, masque, passerelle, DNS, durée du bail.

> 🖥️ Préparation : lab démarré. Ouvrez **deux terminaux client** 💻 et gardez un terminal Codespace 🖥️
> (à la racine du dépôt).

---

## Partie A — Un client sans adresse

### Étape 1 — État de départ

💻 **Terminal client n°1** — notez l'adresse actuelle du client :

```bash
ip -br addr show eth0
```

👀 Vous voyez `172.20.1.50/24`. Cette adresse a été fixée **à la main** (par la configuration
du lab) : c'est une adresse **statique**.

### Étape 2 — Supprimer l'adresse

💻 **Terminal client n°1** — retirez l'adresse IP et faites oublier au client tout bail DHCP précédent :

```bash
ip addr flush dev eth0
rm -f /var/lib/dhcp/dhclient.leases
ip -br addr show eth0
```

💻 Essayez de joindre le routeur :

```bash
ping -c 2 172.20.1.254
```

**Question A.1.** Quel message obtenez-vous ? Pourquoi le client ne peut-il plus rien envoyer ?

> 💬 **Votre réponse :**
>
> _Network is unreachable.le noyau ne trouve aucune route pour atteindre 172.20.1.254. En supprimant l'adresse IP de eth0, on a aussi supprimé la route du réseau directement connecté (172.20.1.0/24) qui en dépendait. Sans adresse source ni route, le client ne peut même pas émettre de paquet, ni résoudre l'adresse MAC du routeur via ARP._

---

## Partie B — Capturer l'échange DORA

### Étape 3 — Lancer la capture

💻 **Terminal client n°1** — enregistrez les paquets DHCP (ports UDP 67 et 68) :

```bash
rm -f /tmp/dhcp.pcap
tcpdump -i eth0 -U -nn -c 4 -w /tmp/dhcp.pcap port 67 or port 68
```

👀 `listening on eth0…` puis attente.

### Étape 4 — Demander une adresse

💻 **Terminal client n°2** — lancez le client DHCP en mode bavard (`-v`) :

```bash
dhclient -v eth0
```

👀 Vous devez voir les 4 étapes, dans cet ordre :

```
DHCPDISCOVER on eth0 to 255.255.255.255 port 67 ...
DHCPOFFER of 172.20.1.1xx from 172.20.1.2
DHCPREQUEST for 172.20.1.1xx on eth0 to 255.255.255.255 port 67
DHCPACK of 172.20.1.1xx from 172.20.1.2
bound to 172.20.1.1xx -- renewal in ... seconds.
```

Et dans le terminal n°1, tcpdump s'arrête : `4 packets captured`.

**Question B.1.** Recopiez les 5 lignes affichées par `dhclient`. Quelle adresse IP avez-vous obtenue ?
Qui vous l'a donnée ?

> 💬 **Votre réponse :**
>
> _DHCPDISCOVER on eth0 to 255.255.255.255 port 67 interval 3
DHCPOFFER of 172.20.1.155 from 172.20.1.2
DHCPREQUEST for 172.20.1.155 on eth0 to 255.255.255.255 port 67
DHCPACK of 172.20.1.155 from 172.20.1.2
le client 2 a obtenue l'ip 172.20.1.55 de 172.20.1.2_

### Étape 5 — Analyser les 4 messages

🖥️ **Terminal Codespace** — listez les paquets capturés :

```bash
./lab/exercises/osi_inspect.py --pcap /tmp/dhcp.pcap
```

🖥️ Puis détaillez chacun d'eux (paquets 1 à 4) :

```bash
./lab/exercises/osi_inspect.py 1 --pcap /tmp/dhcp.pcap
./lab/exercises/osi_inspect.py 2 --pcap /tmp/dhcp.pcap
./lab/exercises/osi_inspect.py 3 --pcap /tmp/dhcp.pcap
./lab/exercises/osi_inspect.py 4 --pcap /tmp/dhcp.pcap
```

**Question B.2.** Complétez le tableau DORA avec **votre** capture :

| Étape       | IP source | IP destination    | MAC destination   | Type DHCP (opt 53) | Transaction ID |
| ----------- | --------- | ----------------- | ----------------- | ------------------ | -------------- |
| 1. Discover | `0.0.0.0` | `255.255.255.255` | ff:ff:ff:ff:ff:ff | 1                  | 0x0821763b     |
| 2. Offer    | 172.20.1.2| 172.20.1.155      | fa:77:40:80:d4:5a | 2                  | 0x0821763b     |
| 3. Request  | 0.0.0.0   | 255.255.255.255   | ff:ff:ff:ff:ff:ff | 3                  | 0x0821763b     |
| 4. ACK      |172.20.1.2 | 172.20.1.155      | fa:77:40:80:d4:5a | 5                  | 0x0821763b     |

**Question B.3.** Dans le paquet **Offer** (n°2), relevez la configuration proposée par le serveur :

| Information              | Option DHCP | Valeur |
| ------------------------ | ----------- | ------ |
| Adresse IP proposée      | (yiaddr)    | 172.20.1.155  |
| Masque de sous-réseau    | 1           | 255.255.255.0 |
| Passerelle (routeur)     | 3           |172.20.1.254 |
| Serveurs DNS             | 6           |1.1.1.1, 8.8.8.8  |
| Durée du bail (secondes) | 51          | 43200  |

Convertissez la durée du bail en heures.

> 💬 **Votre réponse (conversion) :**
>
> _43200 sec = 12heures_

---

## Partie C — Vérifier la configuration reçue

### Étape 6 — Le client est configuré

💻 **Terminal client n°2** :

```bash
ip -br addr show eth0
ip route
cat /etc/resolv.conf
ping -c 2 172.20.1.254
```

**Question C.1.** Retrouvez-vous dans `ip addr`, `ip route` et `resolv.conf` les valeurs de la
question B.3 ? Le ping vers le routeur fonctionne-t-il à nouveau ?

> 💬 **Votre réponse :**
>
> _oui c'est le smeme valeurs que dans B.3. oui le ping refonctionne_

### Étape 7 — Le point de vue du serveur

🖥️ **Terminal Codespace** — lisez le journal du serveur DHCP, en ne gardant que les lignes des 4 étapes :

```bash
docker logs lab_dhcp_server 2>&1 | grep -E "DHCP(DISCOVER|OFFER|REQUEST|ACK)"
```

👀 Chaque ligne contient le *transaction ID* (en décimal), l'étape (`DHCPDISCOVER`, `DHCPOFFER`…),
l'adresse IP et l'adresse MAC concernées.

> 💡 Pour voir **tout** le journal (options envoyées, etc.) : `docker logs --tail 40 lab_dhcp_server`.

**Question C.2.** Retrouvez les 4 étapes DORA dans le journal. Quelle **adresse MAC** le serveur
a-t-il associée à l'IP attribuée ?

> 💬 **Votre réponse :**
>
> _fa:77:40:80:d4:5a_

---

## Partie D — Comprendre

**Question D.1.** Pourquoi le client utilise-t-il `0.0.0.0` comme **IP source** dans le Discover ?

> 💬 **Votre réponse :**
>
> _0.0.0.0 signifie « adresse inconnue / non encore attribuée ». Le client doit pourtant bien mettre quelque chose dans le champ source du paquet IP, donc il utilise cette adresse réservée._

**Question D.2.** Pourquoi le Discover est-il envoyé à `255.255.255.255` et à la MAC `ff:ff:ff:ff:ff:ff` ?
Que signifient ces adresses ?

> 💬 **Votre réponse :**
>
> _Le client ne connaît ni l'adresse IP ni l'adresse MAC du serveur DHCP (il ne sait même pas s'il y en a un). Il envoie donc un broadcast.255.255.255.255 est l'adresse de broadcast IP limité : « tous les hôtes du réseau local », sans passer par un routeur et ff:ff:ff:ff:ff:ff est l'adresse de broadcast Ethernet : la trame est remise à toutes les machines du segment local_

**Question D.3.** À quoi sert le **Transaction ID**, identique dans les 4 messages ?
(Imaginez 30 PC qui démarrent en même temps sur le même réseau.)

> 💬 **Votre réponse :**
>
> _Le Transaction ID (xid) est un identifiant aléatoire choisi par le client, qui permet d'associer chaque réponse à la bonne demande. Comme les messages du serveur (Offer, Ack) sont souvent diffusés en broadcast, les 30 PC les reçoivent tous : chaque client compare le xid avec le sien et ignore les messages qui ne correspondent pas. Il garantit que les quatre messages Discover, Offer, Request, Ack appartiennent bien à la même négociation._

**Question D.4.** Que se passe-t-il quand le **bail expire** ? Pourquoi les baux ont-ils une durée
limitée plutôt qu'infinie ?

> 💬 **Votre réponse :**
>
> _le client ne attend pas la fin du bail. À 50 % de la durée (T1), il tente de le renouveler auprès du serveur (Request en unicast). Sans réponse, à 87,5 % (T2) il retente en broadcast. Si le bail arrive à terme sans renouvellement, le client doit abandonner son adresse IP et recommencer la procédure complète (Discover...). Le serveur, lui, remet l'adresse dans le pool.pour recycler les adresses. Un pool d'adresses est fini : des machines éteintes, déconnectées ou parties (portables, smartphones, invités) monopoliseraient indéfiniment leur adresse avec un bail infini, et le pool finirait par être épuisé._

---

## 🚀 Pour aller plus loin (facultatif)

**P.1 — Rendre son adresse.** 💻 Libérez le bail puis redemandez-en un :

```bash
dhclient -r -v eth0
ip -br addr show eth0
dhclient -v eth0
```

Quel message est envoyé par `dhclient -r` ? Obtenez-vous la même adresse ensuite ? Pourquoi, d'après vous ?

> 💬 **Votre réponse :**
>
> _dhclient -r envoie un DHCPRELEASE au serveur pour lui rendre l'adresse (message affiché : Killed old client process, puis le client s'arrête). L'adresse disparaît de eth0, et le serveur peut la remettre dans son pool. Après dhclient -v eth0, le client suit à nouveau la séquence DORA (Discover, Offer, Request, Ack). Il obtient en général la même adresse : le serveur garde la trace du bail précédent associé à l'adresse MAC du client, et la plupart des serveurs proposent de préférence l'adresse déjà utilisée par ce client. Le client peut aussi la redemander explicitement dans son Discover. Si l'adresse avait été attribuée à un autre client entre-temps, ou si le serveur ne conserve pas l'historique, l'adresse pourrait être différente.._

**P.2 — Panne du serveur DHCP.** 🖥️ Arrêtez le serveur, puis 💻 redemandez une adresse :

```bash
# 🖥️ Terminal Codespace (dans le dossier lab)
docker compose stop dhcp-server
```

```bash
# 💻 Terminal client
dhclient -r eth0
rm -f /var/lib/dhcp/dhclient.leases
timeout 30 dhclient -v -1 eth0
```

Que se passe-t-il ? Quelles conséquences dans une entreprise si le serveur DHCP tombe en panne ?
**Pensez à le redémarrer** : 🖥️ `docker compose start dhcp-server`.

> 💬 **Votre réponse :**
>
> _dhclient envoie des DHCPDISCOVER en broadcast, mais personne ne répond. Il les réémet à intervalles croissants (environ 3, 4, 8, 16 secondes, avec un peu d'aléa), puis le timeout 30 interrompt la commande, ou dhclient -1 abandonne avec le message No DHCPOFFERS received / No working leases in persistent database - sleeping. L'interface eth0 reste sans adresse IP (ip -br addr show eth0 n'affiche aucune adresse IPv4) et le client n'a donc ni IP, ni route, ni passerelle, ni DNS._

---

## ✅ Ce qu'il faut retenir

- Sans adresse IP, une machine ne peut pas communiquer : DHCP la configure **automatiquement**.
- L'échange se fait en 4 messages : **D**iscover, **O**ffer, **R**equest, **A**CK.
- Le client commence en **broadcast** (il ne connaît encore personne) depuis `0.0.0.0`.
- Le serveur fournit IP, masque, passerelle, DNS, et une **durée de bail**.

## 🆘 En cas de problème

| Problème | Solution |
| -------- | -------- |
| Seulement 2 paquets (REQUEST/ACK), pas de Discover | Le client se souvenait d'un ancien bail. 💻 `dhclient -r eth0; rm -f /var/lib/dhcp/dhclient.leases` puis recommencez l'étape 3 |
| `hostname: you must be root to change the host name` | Message sans importance, ignorez-le |
| `dhclient` attend sans fin | Le serveur DHCP est-il démarré ? 🖥️ `docker compose ps` |
| Remettre le client dans son état initial (`172.20.1.50`) | 🖥️ `docker compose restart client` |
