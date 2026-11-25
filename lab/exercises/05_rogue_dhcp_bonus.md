# Exercice 5 (Bonus) — Un faux serveur DHCP sur le réseau

**Niveau :** ⭐⭐⭐ · **Durée :** ~45 min
**📖 Fiche théorique à relire :** `https://<votre-site>.pythonanywhere.com/dhcp` (section *Points de vigilance*)

## 🎯 Objectifs

- Comprendre pourquoi DHCP est **vulnérable** : le client fait confiance au premier serveur qui répond.
- Observer un faux serveur (*rogue DHCP*) détourner un client.
- Utiliser un petit script de **détection**.

> ⚠️ Cette attaque se fait **uniquement dans le lab**. Lancer un faux serveur DHCP sur un vrai
> réseau (école, entreprise, Wi-Fi public) est **interdit** et perturbe tous les utilisateurs.

> 🖥️ Préparation : avoir terminé l'exercice 3. Ouvrez **deux terminaux client** 💻 et gardez un terminal Codespace 🖥️.

---

## Partie A — L'attaquant arrive

Un cinquième conteneur, `rogue-dhcp` (`172.20.1.99`), est déjà préparé dans le lab mais ne démarre
pas par défaut. Sa configuration ([`lab/rogue/dnsmasq.conf`](../rogue/dnsmasq.conf)) distribue des
adresses avec **lui-même comme passerelle et comme DNS**.

🖥️ **Terminal Codespace** (dans le dossier `lab`) :

```bash
docker compose --profile bonus up -d rogue-dhcp
docker compose --profile bonus ps
```

👀 Il y a maintenant **5** machines, dont `lab_rogue_dhcp`.

**Question A.1.** Ouvrez le fichier `lab/rogue/dnsmasq.conf` dans Codespace. Quelle passerelle et quel
DNS ce serveur donne-t-il aux clients ? Comparez avec `lab/dhcp-server/dnsmasq.conf`.

> 💬 **Votre réponse :**
>
> _passerelle:172.20.1.99 et dns 172.20.1.99 au lieu de 172.20.1.254_

---

## Partie B — La course aux offres

### Étape 1 — Écouter les offres

💻 **Terminal client n°1** — affichez chaque **Offer** reçue (serveur, IP proposée, passerelle proposée) :

```bash
tshark -i eth0 -l -Y "dhcp.option.dhcp == 2" -T fields \
  -e dhcp.option.dhcp_server_id -e dhcp.ip.your -e dhcp.option.router 2>/dev/null
```

### Étape 2 — Demander une adresse

💻 **Terminal client n°2** — demandez une nouvelle adresse, puis regardez votre passerelle :

```bash
dhclient -r eth0; rm -f /var/lib/dhcp/dhclient.leases
dhclient -v eth0
ip route
```

Répétez l'étape 2 **trois fois**.

**Question B.1.** Combien d'**Offers** voyez-vous dans le terminal n°1 pour chaque demande ?
De quels serveurs viennent-elles ?

> 💬 **Votre réponse :**
>
> _Je vois deux Offers pour chaque demande : l'une du vrai serveur DHCP (172.20.1.2), qui propose 172.20.1.155 avec la passerelle légitime 172.20.1.254, l'autre du faux serveur (172.20.1.99), qui propose 172.20.1.157 avec lui-même comme passerelle. Les deux répondent au même Discover, car il est envoyé en broadcast_

**Question B.2.** Quelle offre le client a-t-il acceptée (ligne `DHCPOFFER of … from …` et `ip route`) ?
D'après vous, selon quel critère le client choisit-il ?

> 💬 **Votre réponse :**
>
> _Aux trois essais, le client a accepté l'offre du vrai serveur (DHCPOFFER of 172.20.1.155 from 172.20.1.2), et ip route donne chaque fois default via 172.20.1.254. Le faux serveur (172.20.1.99) répond aussi, mais sa réponse arrive après. Le client n'applique aucun critère de confiance : il retient la première offre reçue, donc celle du serveur le plus rapide. Ici, le vrai serveur a été plus rapide à chaque essai. Avec davantage d'essais, le faux serveur finirait peut-être par gagner_

---

## Partie C — Les conséquences

💻 **Terminal client n°2** — essayez d'accéder au site web :

```bash
ip route
curl -m 5 http://172.20.0.10/whoami
```

**Question C.1.** Le site répond-il ? Expliquez pourquoi, en regardant la passerelle du client.

> 💬 **Votre réponse :**
>
> _Oui, le site répond (Your perceived source IP (post-NAT) is: 172.20.0.254). La passerelle du client est 172.20.1.254, c'est-à-dire le routeur légitime : le vrai serveur DHCP a gagné la course, donc le client n'a pas été détourné. Si le faux serveur avait gagné, la passerelle aurait été 172.20.1.99. Le client lui aurait envoyé ses paquets, il ne les aurait pas relayés, et curl aurait expiré au bout de 5 secondes._

**Question C.2.** Ici, le faux serveur ne fait pas suivre le trafic : le client perd simplement son accès.
Un vrai attaquant ferait suivre le trafic **tout en le lisant**. Que pourrait-il voir ou modifier ?
(Pensez aussi au **DNS** qu'il a distribué.)

> 💬 **Votre réponse :**
>
> _lire tout ce qui passe en clair (HTTP, identifiants de formulaires, cookies, mails non chiffrés) et modifier les pages ou les fichiers téléchargés et avec son DNS, répondre aux requêtes par de fausses adresses (DNS spoofing) : le client tape banque.fr et arrive sur un site pirate qui imite l'original (hameçonnage), sans rien remarquer._

---

## Partie D — Détecter l'attaque

Un script de détection est fourni : [`detect_rogue_dhcp.sh`](detect_rogue_dhcp.sh). Ouvrez-le et lisez-le.

💻 Arrêtez tshark dans le **terminal client n°1** (<kbd>Ctrl</kbd>+<kbd>C</kbd>) et ressortez du client (`exit`).
Dans ce même terminal, depuis la racine du dépôt 🖥️, lancez le détecteur sur le poste client :

```bash
docker exec -i lab_client bash < lab/exercises/detect_rogue_dhcp.sh
```

💻 **Terminal client n°2** — redemandez une adresse :

```bash
dhclient -r eth0; rm -f /var/lib/dhcp/dhclient.leases
dhclient -v eth0
```

👀 Le détecteur affiche une ligne `OK` pour le vrai serveur et une ligne `ALERTE` pour l'intrus.

**Question D.1.** Recopiez la sortie du détecteur.

> 💬 **Votre réponse :**
>
> _Surveillance des offres DHCP sur eth0 (Ctrl+C pour arrêter)..._
_OK     : offre du serveur autorisé 172.20.1.2_
_ALERTE : serveur DHCP NON autorisé détecté : 172.20.1.99_

**Question D.2.** Expliquez avec vos mots ce que fait le script : quelle information lit-il dans
chaque Offer, et comment décide-t-il de lever une alerte ?

> 💬 **Votre réponse :**
>
> _Le script écoute eth0 avec tshark et ne retient que les messages DHCP Offer. Dans chaque Offer, il lit l'identifiant du serveur (DHCP Server Identifier). Il le compare à l'adresse du serveur autorisé (172.20.1.2). Si elles sont identiques, il affiche OK. Sinon, il affiche une ALERTE : un serveur DHCP inconnu propose des adresses sur le réseau._

**Question D.3.** Ce script **détecte** l'attaque, mais l'**empêche**-t-il ? Sur les vrais réseaux, les
switchs proposent une protection appelée **DHCP Snooping** : seuls les ports déclarés « de confiance »
ont le droit d'envoyer des Offers. Expliquez pourquoi cette solution est plus efficace.

> 💬 **Votre réponse :**
>
> _le script détecte sans empêcher, alors que le DHCP Snooping bloque les Offers venant d'un port non fiable._

---

## Nettoyage

🖥️ Arrêtez le détecteur (<kbd>Ctrl</kbd>+<kbd>C</kbd>), supprimez le faux serveur, puis remettez le client à zéro :

```bash
docker compose --profile bonus stop rogue-dhcp
docker compose --profile bonus rm -f rogue-dhcp
docker compose restart client
```

---

## ✅ Ce qu'il faut retenir

- DHCP n'authentifie pas les serveurs : le client accepte en général **la première offre** reçue.
- Un faux serveur peut imposer sa **passerelle** et son **DNS**, et donc détourner tout le trafic.
- On peut **détecter** les offres illégitimes (surveillance) et les **bloquer** (DHCP Snooping sur les switchs).

## 🆘 En cas de problème

| Problème | Solution |
| -------- | -------- |
| `no such service: rogue-dhcp` | N'oubliez pas `--profile bonus` dans la commande |
| Le vrai serveur gagne à chaque fois | C'est possible ! Répétez la demande plusieurs fois et comparez |
| Le détecteur n'affiche rien | Vérifiez qu'il tourne bien, puis relancez `dhclient` dans l'autre terminal |
| `bash: lab/exercises/detect_rogue_dhcp.sh: No such file` | Placez-vous à la racine du dépôt : `cd /workspaces/*/` |
