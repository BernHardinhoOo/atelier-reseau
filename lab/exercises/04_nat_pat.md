# Exercice 4 — NAT / PAT : comment le LAN sort sur Internet

**Niveau :** ⭐⭐⭐ · **Durée :** ~1 h
**📖 Fiche théorique à lire avant :** `https://<votre-site>.pythonanywhere.com/nat`

## 🎯 Objectifs

- Constater que le serveur web ne voit **jamais** l'adresse privée du client.
- Lire la **table de traduction** du routeur (`conntrack`).
- **Casser** le NAT, observer la panne, puis le **réparer**.

> 🖥️ Préparation : lab démarré. Ouvrez un **terminal routeur** 🔀 (`docker exec -it lab_nat_router bash`),
> un **terminal client** 💻 (`docker exec -it lab_client bash`) et gardez un terminal Codespace 🖥️.

> 💡 L'adresse du client est `172.20.1.50`, ou l'adresse obtenue par DHCP à l'exercice 3
> (`172.20.1.1xx`). Utilisez **la vôtre** dans les réponses (`ip -br addr show eth0`).

---

## Partie A — Le routeur et sa règle NAT

### Étape 1 — Les deux pattes du routeur

🔀 **Terminal routeur** :

```bash
ip -br addr
```

👀 Le routeur a deux interfaces : **`lan`** (`172.20.1.254`, côté réseau local) et **`wan`**
(`172.20.0.254`, côté « Internet »).

### Étape 2 — La règle MASQUERADE

🔀 Affichez les règles de traduction appliquées aux paquets qui **sortent** du routeur :

```bash
iptables -t nat -L POSTROUTING -n -v --line-numbers
```

👀 Une règle n°1 : `MASQUERADE … out wan`. En français : « tout paquet qui sort par `wan`
prend comme adresse source **celle du routeur** ».

### Étape 3 — Ce que voit le serveur web

💻 **Terminal client** :

```bash
curl http://172.20.0.10/whoami
```

🖥️ **Terminal Codespace** — le journal du serveur web :

```bash
docker logs --tail 3 lab_internet
```

👀 Les lignes du journal contiennent `FROM=…` : l'adresse que le serveur a vue.

**Question A.1.** Quelle adresse le serveur web voit-il ? Est-ce l'adresse du client ? De quelle machine s'agit-il ?

> 💬 **Votre réponse :**
>
> _172.20.0.254 cest l'ip du routeur coté wan l'ip du cliant est 172.20.1.155_

**Question A.2.** Les adresses `172.20.x.x`, `192.168.x.x` et `10.x.x.x` sont des **adresses privées**
(voir la fiche). Pourquoi un vrai serveur sur Internet ne pourrait-il pas répondre à une adresse privée ?

> 💬 **Votre réponse :**
>
> _Les adresses privées ne sont pas routées sur Internet : des millions de réseaux utilisent les mêmes plages (192.168.x.x, etc.), et les routeurs d'Internet les rejettent. Une adresse privée n'identifie donc aucune machine unique dans le monde, et la réponse du serveur n'aurait aucun chemin pour revenir._

---

## Partie B — La table de traduction

### Étape 4 — Vider la table

🔀 **Terminal routeur** — effacez la table pour partir de zéro :

```bash
conntrack -F
```

### Étape 5 — Envoyer trois requêtes

💻 **Terminal client** :

```bash
curl -s http://172.20.0.10/whoami
curl -s http://172.20.0.10/whoami
curl -s http://172.20.0.10/whoami
```

### Étape 6 — Lire la table

🔀 **Terminal routeur** :

```bash
conntrack -L -p tcp
```

👀 Une ligne par connexion. Chaque ligne contient **deux groupes** `src= dst= sport= dport=` :

```
tcp 6 119 TIME_WAIT src=172.20.1.50 dst=172.20.0.10 sport=52942 dport=80 src=172.20.0.10 dst=172.20.0.254 sport=80 dport=52942 [ASSURED] ...
                    └─────────────── ALLER (vu par le client) ───────────┘ └─────────────── RETOUR (vu par le serveur) ────────────┘
```

**Question B.1.** Recopiez **une** ligne de votre table et complétez :

| Sens   | IP source | IP destination | Port source | Port destination |
| ------ | --------- | -------------- | ----------- | ---------------- |
| Aller  |172.20.1.155|172.20.0.10    | 38910       | 80               |
| Retour |172.20.0.10|172.20.0.254    | 80          | 38910            |

Dans le sens **retour**, à quelle adresse le serveur répond-il ? Qui fait la traduction vers le client ?

> 💬 **Votre réponse :**
>
> _172.20.0.254, l'adresse du routeur c'est lui qui fait la traduction_

**Question B.2.** Combien de lignes voyez-vous pour vos 3 requêtes ? Les **ports source** sont-ils
identiques ? Pourquoi est-ce indispensable quand plusieurs PC partagent la même adresse publique ?

> 💬 **Votre réponse :**
>
> _3 lignes pour 3 requetes,les ports sources sont différents car chaque connexion utilise un port éphémère différent. C'est indispensable quand plusieurs PC partagent la même adresse publique : le routeur voit arriver des réponses toutes adressées à sa seule IP, et c'est le port qui lui permet de savoir à quelle connexion, donc à quel PC, elles appartiennent._

**Question B.3.** On parle de **PAT** (*Port Address Translation*) quand plusieurs machines partagent
**une seule** IP publique grâce aux ports. Notre routeur fait-il du NAT simple ou du PAT ? Justifiez.

> 💬 **Votre réponse :**
>
> _Du PAT Plusieurs machines du LAN peuvent sortir avec une seule adresse, et les connexions sont distinguées par leurs ports_

---

## Partie C — Casser le NAT… puis le réparer

### Étape 7 — Supprimer la règle

🔀 **Terminal routeur** :

```bash
iptables -t nat -D POSTROUTING 1
iptables -t nat -L POSTROUTING -n -v --line-numbers
```

👀 La règle MASQUERADE a disparu.

### Étape 8 — Observer la panne

🔀 **Terminal routeur** — écoutez ce qui sort côté WAN :

```bash
tcpdump -i wan -nn -c 4 host 172.20.0.10
```

💻 **Terminal client** — tentez une requête (abandon au bout de 5 secondes) :

```bash
curl -m 5 http://172.20.0.10/whoami
```

🖥️ **Terminal Codespace** :

```bash
docker logs --tail 3 lab_internet
```

**Question C.1.** Que se passe-t-il côté client ? Une nouvelle ligne apparaît-elle dans le journal du serveur web ?

> 💬 **Votre réponse :**
>
> _Le client n'obtient aucune réponse : curl attend puis abandonne au bout de 5 secondes avec Connection timed out (« Operation timed out »). Le journal du serveur web ne montre aucune nouvelle ligne : il n'a jamais reçu de requête exploitable._

**Question C.2.** Dans la capture côté WAN, quelle **IP source** porte le paquet `[S]` (SYN) qui sort ?
Le paquet **aller** est bien parti : pourquoi la **réponse** ne revient-elle jamais au client ?

> 💬 **Votre réponse :**
>
> _Le paquet [S] sort avec l'IP source 172.20.1.155, l'adresse privée du client, car plus aucune règle ne la remplace. Le paquet aller part donc bien, mais le serveur (ou, dans la réalité, le réseau) ne sait pas renvoyer la réponse vers une adresse privée : elle n'est pas routable depuis le côté WAN. Le SYN-ACK n'arrive jamais au client, la connexion TCP ne s'établit pas et curl finit par expirer._

### Étape 9 — Réparer

🔀 **Terminal routeur** (<kbd>Ctrl</kbd>+<kbd>C</kbd> pour arrêter tcpdump s'il tourne encore) :

```bash
iptables -t nat -A POSTROUTING -o wan -j MASQUERADE
```

💻 **Terminal client** :

```bash
curl http://172.20.0.10/whoami
```

👀 Le site répond à nouveau avec `172.20.0.254`.

---

## Partie D — Synthèse

**Question D.1.** Expliquez en 3-4 phrases, avec vos mots, le trajet d'une requête du client vers
le serveur web **et** de sa réponse, en indiquant à quel moment l'adresse est traduite.

> 💬 **Votre réponse :**
>
> _Le client envoie un paquet vers 172.20.0.10 avec sa source 172.20.1.155. Il arrive au routeur, qui le fait sortir par wan. C'est à ce moment que l'adresse source est traduite : 172.20.1.155 devient 172.20.0.254, et le routeur enregistre la correspondance (adresse et port) dans sa table conntrack. Le serveur web voit une requête venant du routeur et répond à 172.20.0.254. À l'arrivée de la réponse, le routeur consulte sa table, retraduit l'adresse de destination en 172.20.1.155 et la transmet au client._

**Question D.2.** Chez vous, combien d'adresses IP publiques votre box possède-t-elle en général ?
Combien d'appareils l'utilisent ? Quel mécanisme rend cela possible ?

> 💬 **Votre réponse :**
>
> _une seule grace au NAT/PAT_

**Question D.3.** Pourquoi le NAT a-t-il été inventé ? (Indice : combien existe-t-il d'adresses IPv4 au total ?)

> 💬 **Votre réponse :**
>
> _Il y a 2^32 addresses en IPV4 (4.3 milliards),le NAT a été inventé pour économiser les addresses publiques_

---

## 🚀 Pour aller plus loin (facultatif) — la redirection de port

Sur une box, la « redirection de port » permet de rendre joignable un service situé derrière
le routeur. C'est un **DNAT** (NAT sur l'adresse de **destination**).

🔀 **Terminal routeur** — redirigez le port `8080` du routeur vers le port `80` du serveur web :

```bash
iptables -t nat -A PREROUTING -i lan -p tcp --dport 8080 -j DNAT --to-destination 172.20.0.10:80
```

💻 **Terminal client** — interrogez **le routeur** sur le port 8080 :

```bash
curl http://172.20.1.254:8080/whoami
```

🔀 Regardez la traduction : `conntrack -L -p tcp | grep 8080`

**Question P.1.** Le client a demandé `172.20.1.254:8080`. Qui a réellement répondu ? Dans la ligne
`conntrack`, qu'est-ce qui a été traduit cette fois : la source, la destination, ou les deux ?

> 💬 **Votre réponse :**
>
> _Le client a demandé 172.20.1.254:8080 (le routeur), mais c'est le serveur web (172.20.0.10, port 80) qui a réellement répondu. Dans la ligne conntrack, les deux ont été traduites. La destination l'a été par le DNAT : 172.20.1.254:8080 devient 172.20.0.10:80. La source l'a été par le MASQUERADE : 172.20.1.155 devient 172.20.0.254 à la sortie par wan. On le voit dans le groupe retour : le serveur répond à 172.20.0.254 et non au client._

**Question P.2.** Donnez un exemple de situation où vous auriez besoin d'une redirection de port chez vous.

> 💬 **Votre réponse :**
>
> _Pour héberger un serveur minecraft_

🔀 Nettoyage : `iptables -t nat -D PREROUTING 1`

---

## ✅ Ce qu'il faut retenir

- Les adresses privées ne circulent pas sur Internet : le routeur les **remplace** par son adresse publique.
- La table `conntrack` mémorise chaque traduction pour renvoyer les réponses au bon client.
- Le **PAT** utilise les **ports** pour que plusieurs machines partagent une seule IP publique.
- Sans NAT, le paquet aller part, mais la **réponse** ne trouve pas le chemin du retour.

## 🆘 En cas de problème

| Problème | Solution |
| -------- | -------- |
| `# Warning: iptables-legacy tables present…` | Simple avertissement, ignorez-le |
| `curl` ne répond plus après la partie C | La règle n'a pas été remise : refaites l'étape 9 |
| `conntrack -L` est vide | Les entrées disparaissent après 2 min : relancez les `curl` de l'étape 5 |
| Tout est cassé | 🖥️ `docker compose restart nat-router` recrée les règles d'origine |
