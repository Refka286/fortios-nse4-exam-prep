# FortiOS 7.6 — Guide de révision condensé
*Basé sur les Quiz partagés*

---

## 1. NAT & VIP — le sujet le plus piégeux

- **SNAT / IP Pool** : quand plusieurs IP pools existent, le pool utilisé est celui référencé dans la policy correspondante (source → destination). Attention aux plages qui se chevauchent (ex: overload vs one-to-one).
- **IP Pool "one-to-one"** : mapping 1:1 → si le pool a moins d'IP que d'utilisateurs internes, les derniers utilisateurs n'ont plus d'IP dispo → pas de connectivité. Solutions : passer en `overload`, ou agrandir le pool (`end ip`).
- **VIP + policy Deny** : pour qu'une policy Deny bloque le trafic vers un VIP, il faut **enable match-vip** dans la policy Deny (sinon elle ne "voit" pas le VIP et le trafic passe par la policy Allow suivante).
- **Port forwarding VIP** : le FortiGate traduit source ET destination au moment du forward. Le client externe garde sa propre IP source (le FortiGate ne fait le SNAT que sur le chemin retour normal) — la destination devient l'IP interne réelle mappée, et le port devient le "Map to IPv4 port" (pas le port externe).
- **Firewall address avec Routing configuration désactivé** : une adresse ne peut être utilisée comme destination de route statique que si "Routing configuration" est activé dans l'objet adresse.
- **DNAT** peut s'appliquer automatiquement à plusieurs policies via les règles DNAT (VIP) — contrairement au SNAT qui est policy-par-policy.

## 2. Firewall Policy — bases qui reviennent souvent

- **Policy ID** : ne peut PAS être modifié une fois la policy créée (mais on peut créer une policy avec ID=0 en CLI).
- **Interface entrante** obligatoire, sortante aussi en pratique — mais une **zone** peut être choisie comme interface sortante, et plusieurs interfaces peuvent être sélectionnées comme entrée/sortie.
- **Consolider deux policies (même profils, interfaces différentes)** → créer une interface **Aggregate** regroupant les deux ports, puis une seule policy dessus (pas "Multiple Interface Policies", qui n'existe pas comme option réelle — piège classique).
- **AV ne bloque pas EICAR** → cause typique : mode Flow-based sans inspection SSL suffisante, ou le profil AV n'est simplement pas appliqué/activé correctement sur le trafic testé — vérifier "deep content inspection" et le mode d'inspection.

## 3. Routing & RPF (reverse path forwarding)

- **Route la plus spécifique / distance la plus basse gagne**. Pour forcer un trafic via port2 uniquement : donner à la nouvelle route une **distance plus basse** (ex: 9) que la route concurrente, PAS la priorité seule (priority ne départage qu'à distance égale).
- **RPF strict** : un paquet est accepté seulement si l'interface d'arrivée correspond à l'interface utilisée pour atteindre la source dans la table de routage (best route vers la source). RPF **désactivé** = plus permissif (accepte même si ça n'arrive pas par la "bonne" interface).
- Table `get router info routing-table database` : contient TOUTES les routes apprises, mais seules certaines sont **installées (FIB, marquées `*` ou `>`)** dans la table de routage active — donc "toutes les entrées sont installées" est FAUX en général.
- Deux default routes avec des **distances différentes** = un seul est actif (le plus bas), l'autre est standby/backup.

## 4. SD-WAN

- **Trafic ne matchant aucune règle SD-WAN** → suit la **règle implicite SD-WAN** (basée sur "source-destination IP", donc load-balancing par session/IP, pas par round-robin pur).
- **Zones** : `virtual-wan-link` et `overlay` sont des zones système par défaut — elles **ne peuvent pas être supprimées**. Underlay peut être vide de membres.
- **SD-WAN rule name absent dans les logs** → typiquement parce que le trafic est passé par la règle **implicite** (pas une règle nommée), donc pas de nom à afficher.
- **Performance SLA** : mesurent packet loss / latency / jitter, peuvent être actifs ou passifs, et les cibles/objectifs SLA sont configurables.
- **ECMP** : si SD-WAN est désactivé, `v4-ecmp-mode` se configure globalement (`system global`) ; si SD-WAN est activé, c'est le paramètre `load-balance-mode` dans la règle SD-WAN qui contrôle l'algorithme.

## 5. IPsec VPN

- **Main mode** vs **Aggressive mode** :
  - Main mode = 6 paquets échangés, ID du peer protégé (chiffré) ; Aggressive = 3 paquets, l'ID du peer est visible dans le 1er paquet (d'où le nom "aggressive" = moins sécurisé mais permet du matching par Peer ID / pré-partage dynamique).
  - **Peer ID** est le paramètre phase 1 utilisé pour faire correspondre un utilisateur nomade (dial-up) à un tunnel spécifique en mode aggressive.
- **XAuth** = authentification supplémentaire des UTILISATEURS via username/password (en plus de l'auth IKE des peers).
- **DPD "On Demand"** = envoie des probes seulement quand il n'y a **pas de trafic entrant** — répond à l'exigence "probes seulement si pas de trafic".
- **Tunnel employé nomade à l'étranger** → modèle **Dial-Up User** (pas Site-to-Site, ni Remote Access qui n'est pas un des 4 modèles standards IPsec Wizard : Site-to-Site / Hub-and-Spoke / Remote Access / Dial-Up — attention, sur l'examen "Remote Access" est parfois un distracteur).
- **add-route activé mais route absente** → il faut que la **phase 2 soit montée avec succès** ET que le **réseau distant soit correctement défini dans les sélecteurs phase 2** (sinon pas de route installée).
- **Traversée de devices bloquant IPsec** → utiliser un mode qui évite ESP/UDP bloqués (SSL VPN tunnel mode en fallback) ou activer la fragmentation pour les négociations avec gros certificats.

## 6. Haute disponibilité (HA / FGCP)

- **Override activé + priorité plus haute** → ce membre devient/reste primaire au reboot/resync (mais **override doit être identique sur tous les membres** sinon le cluster part "out of sync").
- **Override désactivé partout** → le membre avec le plus long **uptime** reste/devient primaire, la priorité ne départage qu'en cas d'égalité d'uptime (uptime prioritaire sur priority quand override=disable).
- **Ordre d'élection standard (override activé)** : ports moniteurs connectés → **Priority** → **uptime du système (system uptime)**, pas "HA uptime" → numéro de série FortiGate.
- **FGCP requiert** : même **HA group ID**, même **mode** (a-p/a-a), même **password**, et — détail piégeux — même **configuration de disque dur** (hard disk config) entre les deux membres. Le nombre de VDOM configurés ou le sous-réseau des interfaces "core" ne sont PAS des exigences obligatoires en soi (le cluster gère ça via heartbeat).
- **Heartbeat IP** : adresses 169.254.x.x utilisées pour distinguer les membres du cluster ; elles changent quand un membre rejoint/quitte le cluster.
- **Conserve mode** (mémoire haute) : basé sur `memory-use-threshold-red/green/extreme` — au-dessus du seuil, FortiGate **droppe les nouvelles sessions nécessitant inspection** et **skip les actions de quarantaine**, mais l'admin **peut toujours changer la config** et **n'a pas besoin du port console** (les deux "restrictions totales" sont des pièges).

## 7. Authentification (Firewall Auth / RADIUS / FSSO)

- **Firewall auth sans prompt de login** → cause la plus fréquente : le **groupe d'utilisateurs (Remote-users) n'est pas ajouté comme Destination/Source** de la policy avec authentification (pas un souci de service DNS).
- **"Include in every user group" (RADIUS)** → place le serveur RADIUS et tous les utilisateurs pouvant s'authentifier contre lui dans **chaque groupe FortiGate**, pas dans un groupe RADIUS séparé.
- **Session SSL VPN qui doit tuer la session auth firewall à la déconnexion** → activer le paramétrage qui force la terminaison de la session d'authentification firewall quand la session SSL VPN se termine (lien explicite, pas juste des timeouts différents).
- **Collector Agent (FSSO) qui ne transmet pas les logins** → vérifier que le **port TCP 8000** est ouvert entre le Collector Agent et le FortiGate (port de communication FSSO par défaut).

## 8. Profils de sécurité (Web Filter / App Control / IPS / SSL Inspection)

- **Catégorie web exemptée d'inspection SSL par défaut** (Finance/Banking, Health...) → deux raisons valides : ce sont des sites dans un **allowlist de domaines réputés maintenu par FortiGuard**, et des **régulations légales** protègent ces catégories (vie privée / infos sensibles) — pas un souci de certificat HSTS.
- **Bloquer un site précis dans une catégorie tout en gardant le reste "Allow"** → deux solutions : entrée **URL filter statique** (Wildcard + Block) OU une **policy séparée** avec adresse FQDN dédiée en Deny — PAS "web override rating" vers une catégorie Malicious (ça reclasse tout le domaine, imprécis) ni mettre la catégorie entière en Warning.
- **Autoriser un seul site dans une catégorie bloquée** (ex: Facebook autorisé, reste des réseaux sociaux bloqué) → **Static URL Filter avec Action = Exempt** pour ce site précis (pas "warning" sur toute la catégorie).
- **Application Control "Allow" mais pas de logs de sécurité** → si l'action est **Allow**, FortiGate n'a en général pas besoin de logguer comme un événement de sécurité — vérifier si l'override est bien configuré en Filter vs Application selon le besoin.
- **Google bloqué malgré override "Allow" prioritaire absent** → dans Application/Filter Overrides, la priorité compte : il faut **remonter la règle Google en priorité 1** pour qu'elle soit évaluée avant une règle Filter plus générale (ex: Excessive-Bandwidth qui bloque avant).
- **SNI check "Strict"** → si le SNI ne correspond pas au CN/SAN du certificat serveur, FortiGate **ferme la connexion** (pas juste un warning).
- **Signature IPS avec Severity affichée en couleur / faible sévérité, Action=Pass** → conclusion typique : le trafic matchant est **autorisé et un log est généré** (Pass + logging enabled), ce n'est ni un groupe de signatures ni un rating custom par défaut.
- **IPS fail-open / diagnose test application ipsmonitor** → sortie avec `engine count = 0` typiquement signale que **FortiGate est entré en IPS fail-open state** (le moteur IPS a planté/n'est pas actif, donc pas de blocage).

## 9. Diagnostics fréquents à savoir lire

- **`diagnose debug flow`** : repérer `find a route... via portX` (interface de sortie utilisée), puis `policy-X is matched, act-XXX`. Si "act-accept" match mais ensuite "Denied by forward policy check" → c'est bien la **policy elle-même qui refuse** (pas un problème de route).
- **`diagnose debug rating` (FortiGuard)** : le port par défaut est **HTTPS/8888** pour Web-filter, et FortiGate utilise souvent un **serveur codé en dur / anycast**, pas nécessairement une résolution DNS classique — attention aux affirmations sur "DNS lookup" qui sont parfois vraies selon le contexte (vérifier le "weight" qui **augmente avec les paquets perdus**, donc un poids élevé = mauvaise qualité de lien vers ce serveur).

---

## Erreurs les plus commises dans tes quiz (à revoir en priorité)
D'après les corrigés, les questions les plus "piège" concernent :
1. **HA election process** (ordre exact : ports → priority → uptime → serial) — question 22/23 du Quiz 2 et Quiz 3.
2. **match-vip** dans les policies Deny — logique NAT/VIP.
3. **RPF strict vs désactivé** avec plusieurs sources/interfaces.
4. **Web filter : Exempt vs Warning vs Override rating** — bien distinguer les 3 mécanismes.
5. **SD-WAN : règle implicite vs zones système non supprimables**.


