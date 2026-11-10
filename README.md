# htaccess-kill-switch

[![Licence MIT](https://img.shields.io/badge/licence-MIT-5A4095?style=flat-square)](LICENSE)
[![Apache](https://img.shields.io/badge/Apache-2.4%2B%20.htaccess-D22128?style=flat-square&logo=apache&logoColor=white)](htaccess-kill-switch.conf)
[![nginx](https://img.shields.io/badge/nginx-vhost%20fourni-009639?style=flat-square&logo=nginx&logoColor=white)](vhost-nginx-kill-switch.conf)
[![WordPress](https://img.shields.io/badge/WordPress-sans%20plugin-21759B?style=flat-square&logo=wordpress&logoColor=white)](#le-probl%C3%A8me)
[![HTTP 503](https://img.shields.io/badge/HTTP-503%20SEO--safe-FF6F00?style=flat-square)](#le-principe)
![Dépendances : zéro](https://img.shields.io/badge/d%C3%A9pendances-0-brightgreen?style=flat-square)
[![Article](https://img.shields.io/badge/article-julienweb.fr-1E91D4?style=flat-square)](https://julienweb.fr/blog/mettre-site-wordpress-en-pause-htaccess/11426/)

**[Français](#fr) · [English](#en)**

Un interrupteur marche/arrêt pour un site WordPress en mutualisé, actionnable par le client seul,
sans accès au tableau de bord, sans plugin, sans ligne de commande : il renomme un fichier.

`PAUSE-OFF.txt` → `PAUSE.txt` : le site répond **HTTP 503** et sert une page de maintenance.
`PAUSE.txt` → `PAUSE-OFF.txt` : le site revient. Effet immédiat, aucun cache à vider.

---

## FR

### Le problème

Aucun hébergeur mutualisé ne fournit de bouton « mettre mon site en pause ». Ni IONOS, ni OVHcloud,
ni o2switch, ni Infomaniak. Le client qui veut couper son site n'a que des options destructrices :
supprimer les fichiers, changer les DNS, résilier. Toutes sont difficiles à annuler et toutes
coûtent du SEO.

Le plugin WordPress de maintenance ne répond pas au cas qui compte. Un plugin s'exécute *dans*
WordPress : il suppose que WordPress démarre, que la base répond, que l'admin est accessible, et que
personne d'autre ne contrôle le site. Quand le site est piraté, ces quatre hypothèses tombent
ensemble. Un site avec un webshell dans `wp-content/uploads/` continue de servir l'attaquant même
avec le mode maintenance activé, parce que le webshell n'a pas besoin de WordPress pour tourner. Et
un site en erreur fatale PHP ne charge aucun plugin du tout.

L'interrupteur décrit ici vit dans `.htaccess`, donc **au-dessus** de PHP et de WordPress. Il coupe
avant que quoi que ce soit ne démarre. Il fonctionne sur un site cassé, sur un site infecté, sur un
site dont personne n'a plus le mot de passe admin.

Origine : intervention de remise en ligne après piratage d'une boutique WooCommerce, où le client
avait besoin de pouvoir refermer son site tout seul, à n'importe quelle heure, pendant que le
nettoyage se poursuivait.

### Le principe

Deux pièces, rien d'autre.

1. **Un fichier témoin** à la racine du site. Son *nom* porte l'état : `PAUSE-OFF.txt` (en ligne) ou
   `PAUSE.txt` (en pause). Aucun contenu n'est lu, seule l'existence du nom compte.
2. **Un bloc `.htaccess`** qui teste la présence de ce fichier à chaque requête et, s'il le trouve,
   renvoie 503 + `maintenance.html` + `Retry-After: 3600`.

Pourquoi 503 et pas 403 ou 404 : le 503 est le seul code que Google traite comme « reviens plus
tard ». L'exploration est suspendue, le classement est gelé, rien n'est désindexé. Un 404 sur toutes
les URL d'un site, tenu quelques jours, coûte des mois de reprise. Le `Retry-After: 3600` annonce la
durée estimée de l'interruption.

`robots.txt` reste servi pendant la pause, sinon les robots repartent sur un cache et interprètent
mal la coupure. La page `maintenance.html` est autonome : pas de police Google, pas de CDN, pas de
script, 1,7 ko environ, `noindex, follow`. Elle s'affiche même si tout le reste du site est mort.

Les fichiers témoins répondent 403 en HTTP, dans les deux états : ils n'ont rien à faire sur le Web.

### Ce que contient le dépôt

| Fichier | Rôle |
|---|---|
| [`htaccess-kill-switch.conf`](htaccess-kill-switch.conf) | le bloc à coller en haut du `.htaccess` existant |
| [`htaccess-complet-exemple.conf`](htaccess-complet-exemple.conf) | un `.htaccess` complet et durci, interrupteur inclus |
| [`vhost-apache-kill-switch.conf`](vhost-apache-kill-switch.conf) | la même règle en configuration serveur Apache |
| [`vhost-nginx-kill-switch.conf`](vhost-nginx-kill-switch.conf) | l'équivalent nginx, audité contre la doc officielle |
| [`maintenance.html`](maintenance.html) | la page servie pendant la pause, autonome et `noindex` |
| [`PAUSE-OFF.txt`](PAUSE-OFF.txt) | le fichier témoin, avec le mode d'emploi écrit dedans |
| [`tutoriel-client/`](tutoriel-client/) | le pas-à-pas à remettre au client, en HTML et en PDF |

### Installation en 3 étapes

**1. Trouver le chemin absolu du docroot.**
Il ressemble à `/kunden/homepages/NN/dXXXXXXXXX/htdocs` chez IONOS, `/home/COMPTE/www` chez
OVHcloud, `/home/COMPTE/public_html` en cPanel. Le gestionnaire de fichiers de l'hébergeur l'affiche
en général en haut de page. Sinon, poser trente secondes un fichier `chemin.php` à la racine :

```php
<?php echo __DIR__;
```

Appeler `https://votre-site.fr/chemin.php`, noter le résultat, **supprimer le fichier**.

**2. Copier les trois fichiers à la racine du site.**

| Fichier du dépôt | Destination |
|---|---|
| `htaccess-kill-switch.conf` | à coller **en haut** du `.htaccess` existant |
| `maintenance.html` | `/maintenance.html` |
| `PAUSE-OFF.txt` | `/PAUSE-OFF.txt` |

Sauvegarder le `.htaccess` existant avant de le modifier. Pousser en **LF**, jamais en CRLF : un
`.htaccess` en fins de ligne Windows produit des 500 sur certains serveurs.

Dans le bloc collé, remplacer `votre-site.fr` par le domaine et
`/kunden/homepages/NN/dXXXXXXXXX/htdocs/` par le chemin trouvé à l'étape 1. Le chemin garde son
slash final avant `PAUSE.txt`.

**3. Recetter les deux sens, avant de confier l'interrupteur au client.**

```bash
curl -sI https://votre-site.fr/            # attendu : HTTP/2 200
# renommer PAUSE-OFF.txt en PAUSE.txt
curl -sI https://votre-site.fr/            # attendu : HTTP/2 503 + retry-after: 3600
curl -s  https://votre-site.fr/ | head -5  # attendu : la page de maintenance
curl -sI https://votre-site.fr/robots.txt  # attendu : HTTP/2 200, toujours servi
curl -sI https://votre-site.fr/PAUSE.txt   # attendu : HTTP/2 403
# renommer PAUSE.txt en PAUSE-OFF.txt
curl -sI https://votre-site.fr/            # attendu : HTTP/2 200
```

Tester en `http://` et en `https://`, avec et sans `www`, et sur chaque alias technique du domaine.
Si l'hébergement porte plusieurs sites, vérifier surtout que les **voisins répondent toujours 200**
pendant que celui-ci est en pause.

### Renommer un fichier sans FTP : qui a un gestionnaire de fichiers web

C'est la question qui décide si le client peut se servir de l'interrupteur tout seul. État au
premier semestre 2026.

| Hébergeur | Gestionnaire de fichiers dans le navigateur | Nom de l'outil |
|---|---|---|
| IONOS | oui | WebTransfert (`login.ionos.fr`) |
| OVHcloud | ⚠️ **non depuis 2026** | FTP Explorer et Net2FTP désactivés, client FTP obligatoire |
| cPanel (tout revendeur) | oui | File Manager |
| Plesk (tout revendeur) | oui | Gestionnaire de fichiers |
| o2switch | oui | cPanel File Manager |
| Infomaniak | oui | Web FTP |
| Hostinger | oui | hPanel File Manager |
| LWS | oui | Gestionnaire de fichiers |
| PlanetHoster | oui | cPanel / HCloud File Manager |
| Gandi | non | SSH ou FTP uniquement |

Chez ceux qui n'en ont pas, prévoir un client FTP installé et configuré sur le poste du client
(FileZilla, WinSCP, Cyberduck) avant de lui remettre la procédure. Renommer un fichier reste une
opération triviale en FTP, mais elle suppose un logiciel déjà prêt.

### Garder la main pendant la pause

Le bloc porte une ligne commentée pour laisser passer votre adresse IP : vous voyez le site, les
visiteurs voient la page de maintenance.

```apache
# RewriteCond %{REMOTE_ADDR} !^203\.0\.113\.42$
```

Décommentez-la, mettez votre adresse IP publique avec les points échappés, une ligne par adresse.
Derrière Cloudflare ou un autre proxy, `REMOTE_ADDR` porte l'adresse du proxy, pas la vôtre : la
ligne ne marche pas telle quelle. Une box change d'adresse sans prévenir : vérifiez la vôtre le jour
même.

### Les 3 pièges documentés

**Piège 1 : `%{DOCUMENT_ROOT}` ment, et il ment sans erreur.**
Chez IONOS, mesuré sur deux vhosts distincts du même espace, `%{DOCUMENT_ROOT}` vaut `/var/www/html`
pour tous. Le test `%{DOCUMENT_ROOT}/PAUSE.txt -f` est donc toujours faux. Pire : `%{DOCUMENT_ROOT}
-d` répond **vrai**, parce que le chemin existe dans le conteneur. Une règle écrite avec cette
variable a l'air de fonctionner et branche en réalité **un seul interrupteur sur tous les sites de
l'espace**. Les chemins relatifs (`PAUSE.txt`, `sous-dossier/PAUSE.txt`) échouent également. Seul le
chemin **absolu** est fiable. `%{REQUEST_FILENAME}` est la seule autre variable correctement
positionnée par vhost.

**Piège 2 : la cascade sur les voisins du même docroot.**
Sur un mutualisé, un second site vit souvent dans un sous-dossier du docroot du premier, sans
`.htaccess` propre. Il hérite alors de toutes les directives du parent. Une règle de maintenance
sans garde d'hôte coupe donc les voisins en même temps. Incident constaté : trois sites éteints d'un
coup par une règle destinée à un seul. Le `RewriteCond %{HTTP_HOST} ^(www\.)?votre-site\.fr$ [NC]`
n'est pas cosmétique, c'est ce qui rend l'interrupteur unitaire. Un site qui a son propre
`.htaccess` avec `RewriteEngine On` coupe l'héritage des règles de réécriture du parent, mais pas
celui des `Header`, `ErrorDocument` ou `FilesMatch`.

**Piège 3 : la redirection canonique mange le 503.**
Si le `.htaccess` force HTTPS ou `www` par un 301, cette redirection s'applique aussi à la
sous-requête interne que Apache émet pour rendre `ErrorDocument`. Résultat : le visiteur reçoit un
301, puis la page de maintenance en **200**, et Google enregistre un site normal avec une page
bizarre. Correctif : exclure la page de maintenance du bloc canonique.

```apache
RewriteCond %{REQUEST_URI} !^/maintenance[.]html$
```

Corollaire du même mécanisme : Apache préfixe les variables d'environnement par `REDIRECT_` dans la
sous-requête. D'où les **deux** lignes `Retry-After`, `env=JW_MAINT` et `env=REDIRECT_JW_MAINT`.
Avec une seule, l'en-tête manque sur la réponse réellement envoyée.

### Si vous administrez le serveur

Le `.htaccess` est fait pour le cas où l'on n'a pas la main sur la machine. Si vous l'avez, mettez la
règle dans la configuration du serveur : elle y est plus rapide (aucun `.htaccess` à chercher et
relire à chaque requête) et plus difficile à neutraliser par un fichier déposé dans un répertoire
d'upload.

| Fichier | Pour |
|---|---|
| [`htaccess-kill-switch.conf`](htaccess-kill-switch.conf) | mutualisé, aucun accès serveur — le cas par défaut |
| [`vhost-apache-kill-switch.conf`](vhost-apache-kill-switch.conf) | VPS/dédié sous Apache |
| [`vhost-nginx-kill-switch.conf`](vhost-nginx-kill-switch.conf) | VPS/dédié sous nginx |

Quatre pièges spécifiques à ces versions serveur, tous documentés dans les fichiers :

1. **`AllowOverride None` coupe TOUS les `.htaccess`**, dont celui de WordPress. Ses règles de
   permaliens doivent être remontées dans le vhost, sinon toutes les URLs tombent en 404. Le vhost
   Apache fourni contient déjà ce bloc. Dans le doute, gardez `AllowOverride All` : vous perdez le
   gain de vitesse, l'interrupteur fonctionne quand même.
2. **Sous nginx, `error_page 503 @maintenance;` s'écrit SANS signe égal.** La doc officielle illustre
   pourtant les locations nommées avec `error_page 404 = @fallback;`. Recopier ce `=` fait *adopter*
   le code du gestionnaire : le 503 devient un 200, et le site se fait désindexer par la manœuvre
   censée l'en protéger.
3. **`rewrite ^ /maintenance.html break;` dégrade en 404** si le fichier manque, parce que le statut
   de la dernière erreur l'emporte. `try_files /maintenance.html =503;` est la seule forme qui ne
   peut pas dégrader le code.
4. **Sous nginx, tester `$uri` et non `$request_uri`** pour les exemptions : `$request_uri` inclut la
   query string, donc `/robots.txt?v=2` ne matche pas l'ancre de fin et part en 503 — exactement le
   cas que Google pénalise.

Le `if` au niveau `server` n'est pas un problème, contrairement à ce qu'on lit partout : la page qui
a fait sa réputation s'intitule *« If is Evil… when used in location context »* et indique
explicitement qu'on peut remonter les `if` au niveau `server`, « où c'est sûr ».

### Ce que ça ne fait pas

Ça coupe. Ça ne répare pas, ça ne désinfecte pas.

- **Ce n'est pas une remédiation.** Un site piraté reste piraté derrière son 503. Les portes
  d'entrée (identifiants volés, extension vulnérable, poste du client infecté) restent ouvertes.
  L'interrupteur achète du temps propre pour travailler, il ne remplace aucune étape du nettoyage.
- **Ça ne bloque pas l'attaquant déjà dedans.** Un accès SFTP ou une session admin volée continue de
  fonctionner : le 503 concerne le trafic HTTP public, pas les autres canaux. La règle anti-reshell
  du fichier complet limite l'exécution de PHP dans `uploads/`, ce qui est un durcissement, pas une
  garantie.
- **Ça ne protège pas le back-office.** `/wp-admin` répond 503 comme le reste. Utile pour couper les
  bots, inutile comme contrôle d'accès. Un vrai verrou, c'est un `.htpasswd` ou un filtre IP.
- **Ça ne surveille rien.** Personne n'est prévenu quand le site tombe pour de vrai. Brancher une
  sonde de disponibilité, et la configurer pour ne pas alerter sur les 503 volontaires.
- **Ça dépend d'Apache**, du moins dans sa version `.htaccess`. Nginx ne lit pas de `.htaccess` : sa
  configuration est centralisée, donc il faut un accès à la machine. L'équivalent exact est fourni
  dans [`vhost-nginx-kill-switch.conf`](vhost-nginx-kill-switch.conf), audité contre la
  documentation officielle. Sur un mutualisé nginx, vous n'avez pas cet accès : demandez à
  l'hébergeur.
- **Ça laisse le cache tiers en travers.** Un CDN ou un cache de page peut continuer à servir des
  copies HTML pendant plusieurs minutes. Purger après avoir basculé l'interrupteur.

### Licence et références

MIT, voir [LICENSE](LICENSE).

- Article détaillé : <https://julienweb.fr/blog/mettre-site-wordpress-en-pause-htaccess/11426/>
- Site piraté, plainte, prestataires agréés : <https://www.cybermalveillance.gouv.fr/>
- Auteur : Julien Guézennec, [julienweb.fr](https://julienweb.fr), contact@julienweb.fr

---

## EN

### The problem

No shared host ships a "pause my site" button. Not IONOS, not OVHcloud, not any of the others. The
only options a customer has are destructive: delete the files, change DNS, cancel the plan. All are
hard to undo and all cost SEO.

A WordPress maintenance plugin does not cover the case that matters. A plugin runs *inside*
WordPress: it assumes WordPress boots, the database answers, wp-admin is reachable, and nobody else
controls the site. When the site is hacked, those four assumptions fail together. A webshell sitting
in `wp-content/uploads/` keeps serving the attacker with maintenance mode on, because it does not
need WordPress to run. A site in PHP fatal error loads no plugin at all.

This switch lives in `.htaccess`, above PHP and above WordPress. It cuts before anything boots. It
works on a broken site, an infected site, and a site whose admin password nobody has anymore.

### How it works

Two pieces. A **flag file** at the site root whose *name* carries the state (`PAUSE-OFF.txt` for
live, `PAUSE.txt` for paused), and an **`.htaccess` block** that checks for it on every request and
answers 503 + `maintenance.html` + `Retry-After: 3600`.

503 is the only status Google reads as "come back later": crawling pauses, rankings freeze,
nothing gets deindexed. `robots.txt` keeps being served. `maintenance.html` is self-contained
(no fonts, no CDN, no script, about 1.7 kB, `noindex, follow`) so it renders even when everything
else is dead. Both flag files return 403 over HTTP.

### What's in the repo

| File | Purpose |
|---|---|
| [`htaccess-kill-switch.conf`](htaccess-kill-switch.conf) | the block to paste at the top of an existing `.htaccess` |
| [`htaccess-complet-exemple.conf`](htaccess-complet-exemple.conf) | a full hardened `.htaccess`, switch included |
| [`vhost-apache-kill-switch.conf`](vhost-apache-kill-switch.conf) | the same rule as Apache server config |
| [`vhost-nginx-kill-switch.conf`](vhost-nginx-kill-switch.conf) | the nginx equivalent, audited against the official docs |
| [`maintenance.html`](maintenance.html) | the page served while paused, self-contained and `noindex` |
| [`PAUSE-OFF.txt`](PAUSE-OFF.txt) | the flag file, with its instructions written inside |
| [`tutoriel-client/`](tutoriel-client/) | the walkthrough to hand over to the customer (HTML and PDF, in French) |

### Install in 3 steps

1. **Find the absolute docroot path.** Look at your host's file manager, or drop `<?php echo
   __DIR__;` at the root for thirty seconds, call it, then delete it.
2. **Copy the files.** Paste `htaccess-kill-switch.conf` at the **top** of the existing `.htaccess`
   (back it up first, upload in **LF**), then `maintenance.html` and `PAUSE-OFF.txt` at the root.
   Replace `votre-site.fr` with your domain and the placeholder path with the real one.
3. **Test both directions** with `curl -sI` before handing the switch over: 200, rename to
   `PAUSE.txt`, 503 with `retry-after`, `robots.txt` still 200, flag file 403, rename back, 200.
   Test http and https, www and non-www, and check that neighbouring sites still answer 200.

### Web file manager, by host (2026)

| Host | In-browser file manager | Tool |
|---|---|---|
| IONOS | yes | WebTransfert (`login.ionos.fr`) |
| OVHcloud | ⚠️ **no, as of 2026** | FTP Explorer and Net2FTP retired, FTP client required |
| cPanel (any reseller) | yes | File Manager |
| Plesk (any reseller) | yes | File Manager |
| o2switch | yes | cPanel File Manager |
| Infomaniak | yes | Web FTP |
| Hostinger | yes | hPanel File Manager |
| LWS | yes | File manager |
| PlanetHoster | yes | cPanel / HCloud File Manager |
| Gandi | no | SSH or FTP only |

Where there is none, install and configure an FTP client on the customer's machine first.

### Keep access during the pause

The block ships a commented line that lets your own IP address through: you see the site, visitors
get the maintenance page.

```apache
# RewriteCond %{REMOTE_ADDR} !^203\.0\.113\.42$
```

Uncomment it, put in your public IP with escaped dots, one line per address. Behind Cloudflare or
any other proxy, `REMOTE_ADDR` holds the proxy's address, not yours: the line will not work as is.
Home connections change address without notice: check yours on the day.

### The 3 documented traps

1. **`%{DOCUMENT_ROOT}` lies, silently.** On IONOS it reads `/var/www/html` for every vhost of the
   space. The `-f` test always fails, while `%{DOCUMENT_ROOT} -d` returns **true**, so a rule
   written with it looks correct and actually wires **one switch across every site**. Relative paths
   fail too. Use the absolute path.
2. **Cascade onto neighbours.** On shared hosting a second site often lives in a subfolder of the
   first site's docroot with no `.htaccess` of its own, inheriting everything. Without the
   `RewriteCond %{HTTP_HOST}` guard, pausing one site pauses them all (observed: three sites down at
   once from a rule meant for one).
3. **The canonical 301 eats the 503.** An HTTPS or www redirect also applies to the internal
   subrequest Apache issues for `ErrorDocument`, turning the answer into a 301 then a **200**. Fix:
   `RewriteCond %{REQUEST_URI} !^/maintenance[.]html$` inside the canonical block. Same mechanism
   explains the **two** `Retry-After` lines: Apache prefixes env vars with `REDIRECT_` in the
   subrequest.

### If you run the server yourself

The `.htaccess` version exists for the case where you have no access to the machine. If you do have
it, put the rule in the server config instead: it is faster (no `.htaccess` lookup on every request)
and it cannot be neutralised by a file dropped into an upload directory.

| File | For |
|---|---|
| [`htaccess-kill-switch.conf`](htaccess-kill-switch.conf) | shared hosting, no server access — the default case |
| [`vhost-apache-kill-switch.conf`](vhost-apache-kill-switch.conf) | Apache on a VPS or dedicated box |
| [`vhost-nginx-kill-switch.conf`](vhost-nginx-kill-switch.conf) | nginx on a VPS or dedicated box |

Four traps specific to the server-side versions, all documented inside the files:

1. **`AllowOverride None` disables every `.htaccess`**, WordPress's included. Its permalink rules must
   be moved into the vhost or every URL 404s on reload. The Apache vhost here already carries that
   block. When in doubt keep `AllowOverride All`: you lose the speed gain, the switch still works.
2. **On nginx, write `error_page 503 @maintenance;` with NO equals sign.** The official docs show
   named locations as `error_page 404 = @fallback;`. Copying that `=` makes nginx *adopt* the
   handler's status code: your 503 becomes a 200, and the site gets deindexed by the very manoeuvre
   meant to protect it.
3. **`rewrite ^ /maintenance.html break;` degrades to 404** when the file is missing, because the
   status of the last error wins. `try_files /maintenance.html =503;` is the only form that cannot
   downgrade the status code.
4. **On nginx, test `$uri`, not `$request_uri`**, for the exemptions: `$request_uri` keeps the query
   string, so `/robots.txt?v=2` fails the end anchor and gets a 503 — precisely the case Google
   penalises.

The `if` at `server` level is fine, contrary to what you will read everywhere: the page that made its
reputation is titled *"If is Evil… when used in location context"* and explicitly says you may move
`if` up to server level, "where it's safe".

### What it does not do

It cuts. It does not fix, and it does not clean.

- Not a remediation: a hacked site stays hacked behind its 503, and the entry point stays open.
- Does not lock out an attacker already inside: stolen SFTP or admin sessions are unaffected.
- Not access control: `/wp-admin` returns 503 like everything else. Use `.htpasswd` or an IP filter.
- No monitoring: add an uptime probe, and configure it to ignore intentional 503s.
- Apache only, in its `.htaccess` form. Nginx reads no `.htaccess`: its config is centralised, so you
  need access to the machine. The exact equivalent ships as
  [`vhost-nginx-kill-switch.conf`](vhost-nginx-kill-switch.conf), audited against the official docs.
  On shared nginx hosting you do not have that access: ask the host.
- Third-party caches and CDNs may keep serving HTML copies. Purge after flipping the switch.

### License

MIT, see [LICENSE](LICENSE). Written by Julien Guézennec, [julienweb.fr](https://julienweb.fr).
