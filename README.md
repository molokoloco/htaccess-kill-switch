# htaccess-kill-switch

Un interrupteur marche/arret pour un site WordPress en mutualise, actionnable par le client seul,
sans acces au tableau de bord, sans plugin, sans ligne de commande : il renomme un fichier.

`PAUSE-OFF.txt` -> `PAUSE.txt` : le site repond **HTTP 503** et sert une page de maintenance.
`PAUSE.txt` -> `PAUSE-OFF.txt` : le site revient. Effet immediat, aucun cache a vider.

---

## FR

### Le probleme

Aucun hebergeur mutualise ne fournit de bouton « mettre mon site en pause ». Ni IONOS, ni OVHcloud,
ni o2switch, ni Infomaniak. Le client qui veut couper son site n'a que des options destructrices :
supprimer les fichiers, changer les DNS, resilier. Toutes sont difficiles a annuler et toutes
coutent du SEO.

Le plugin WordPress de maintenance ne repond pas au cas qui compte. Un plugin s'execute *dans*
WordPress : il suppose que WordPress demarre, que la base repond, que l'admin est accessible, et que
personne d'autre ne controle le site. Quand le site est pirate, ces quatre hypotheses tombent
ensemble. Un site avec un webshell dans `wp-content/uploads/` continue de servir l'attaquant meme
avec le mode maintenance active, parce que le webshell n'a pas besoin de WordPress pour tourner. Et
un site en erreur fatale PHP ne charge aucun plugin du tout.

L'interrupteur decrit ici vit dans `.htaccess`, donc **au-dessus** de PHP et de WordPress. Il coupe
avant que quoi que ce soit ne demarre. Il fonctionne sur un site casse, sur un site infecte, sur un
site dont personne n'a plus le mot de passe admin.

Origine : intervention de remise en ligne apres piratage d'une boutique WooCommerce, ou le client
avait besoin de pouvoir refermer son site tout seul, a n'importe quelle heure, pendant que le
nettoyage se poursuivait.

### Le principe

Deux pieces, rien d'autre.

1. **Un fichier temoin** a la racine du site. Son *nom* porte l'etat : `PAUSE-OFF.txt` (en ligne) ou
   `PAUSE.txt` (en pause). Aucun contenu n'est lu, seule l'existence du nom compte.
2. **Un bloc `.htaccess`** qui teste la presence de ce fichier a chaque requete et, s'il le trouve,
   renvoie 503 + `maintenance.html` + `Retry-After: 3600`.

Pourquoi 503 et pas 403 ou 404 : le 503 est le seul code que Google traite comme « reviens plus
tard ». L'exploration est suspendue, le classement est gele, rien n'est desindexe. Un 404 sur toutes
les URL d'un site, tenu quelques jours, coute des mois de reprise. Le `Retry-After: 3600` annonce la
duree estimee de l'interruption.

`robots.txt` reste servi pendant la pause, sinon les robots repartent sur un cache et interpretent
mal la coupure. La page `maintenance.html` est autonome : pas de police Google, pas de CDN, pas de
script, 1,7 ko environ, `noindex, follow`. Elle s'affiche meme si tout le reste du site est mort.

Les fichiers temoins repondent 403 en HTTP, dans les deux etats : ils n'ont rien a faire sur le Web.

### Installation en 3 etapes

**1. Trouver le chemin absolu du docroot.**
Il ressemble a `/kunden/homepages/NN/dXXXXXXXXX/htdocs` chez IONOS, `/home/COMPTE/www` chez
OVHcloud, `/home/COMPTE/public_html` en cPanel. Le gestionnaire de fichiers de l'hebergeur l'affiche
en general en haut de page. Sinon, poser trente secondes un fichier `chemin.php` a la racine :

```php
<?php echo __DIR__;
```

Appeler `https://votre-site.fr/chemin.php`, noter le resultat, **supprimer le fichier**.

**2. Copier les trois fichiers a la racine du site.**

| Fichier du depot | Destination |
|---|---|
| `htaccess-kill-switch.conf` | a coller **en haut** du `.htaccess` existant |
| `maintenance.html` | `/maintenance.html` |
| `PAUSE-OFF.txt` | `/PAUSE-OFF.txt` |

Sauvegarder le `.htaccess` existant avant de le modifier. Pousser en **LF**, jamais en CRLF : un
`.htaccess` en fins de ligne Windows produit des 500 sur certains serveurs.

Dans le bloc colle, remplacer `votre-site.fr` par le domaine et
`/kunden/homepages/NN/dXXXXXXXXX/htdocs/` par le chemin trouve a l'etape 1. Le chemin garde son
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
Si l'hebergement porte plusieurs sites, verifier surtout que les **voisins repondent toujours 200**
pendant que celui-ci est en pause.

### Renommer un fichier sans FTP : qui a un gestionnaire de fichiers web

C'est la question qui decide si le client peut se servir de l'interrupteur tout seul. Etat au
premier semestre 2026.

| Hebergeur | Gestionnaire de fichiers dans le navigateur | Nom de l'outil |
|---|---|---|
| IONOS | oui | WebTransfert (`login.ionos.fr`) |
| OVHcloud | ⚠️ **non depuis 2026** | FTP Explorer et Net2FTP desactives, client FTP obligatoire |
| cPanel (tout revendeur) | oui | File Manager |
| Plesk (tout revendeur) | oui | Gestionnaire de fichiers |
| o2switch | oui | cPanel File Manager |
| Infomaniak | oui | Web FTP |
| Hostinger | oui | hPanel File Manager |
| LWS | oui | Gestionnaire de fichiers |
| PlanetHoster | oui | cPanel / HCloud File Manager |
| Gandi | non | SSH ou FTP uniquement |

Chez ceux qui n'en ont pas, prevoir un client FTP installe et configure sur le poste du client
(FileZilla, WinSCP, Cyberduck) avant de lui remettre la procedure. Renommer un fichier reste une
operation triviale en FTP, mais elle suppose un logiciel deja pret.

### Les 3 pieges documentes

**Piege 1 : `%{DOCUMENT_ROOT}` ment, et il ment sans erreur.**
Chez IONOS, mesure sur deux vhosts distincts du meme espace, `%{DOCUMENT_ROOT}` vaut `/var/www/html`
pour tous. Le test `%{DOCUMENT_ROOT}/PAUSE.txt -f` est donc toujours faux. Pire : `%{DOCUMENT_ROOT}
-d` repond **vrai**, parce que le chemin existe dans le conteneur. Une regle ecrite avec cette
variable a l'air de fonctionner et branche en realite **un seul interrupteur sur tous les sites de
l'espace**. Les chemins relatifs (`PAUSE.txt`, `sous-dossier/PAUSE.txt`) echouent egalement. Seul le
chemin **absolu** est fiable. `%{REQUEST_FILENAME}` est la seule autre variable correctement
positionnee par vhost.

**Piege 2 : la cascade sur les voisins du meme docroot.**
Sur un mutualise, un second site vit souvent dans un sous-dossier du docroot du premier, sans
`.htaccess` propre. Il herite alors de toutes les directives du parent. Une regle de maintenance
sans garde d'hote coupe donc les voisins en meme temps. Incident constate : trois sites eteints d'un
coup par une regle destinee a un seul. Le `RewriteCond %{HTTP_HOST} ^(www\.)?votre-site\.fr$ [NC]`
n'est pas cosmetique, c'est ce qui rend l'interrupteur unitaire. Un site qui a son propre
`.htaccess` avec `RewriteEngine On` coupe l'heritage des regles de reecriture du parent, mais pas
celui des `Header`, `ErrorDocument` ou `FilesMatch`.

**Piege 3 : la redirection canonique mange le 503.**
Si le `.htaccess` force HTTPS ou `www` par un 301, cette redirection s'applique aussi a la
sous-requete interne que Apache emet pour rendre `ErrorDocument`. Resultat : le visiteur recoit un
301, puis la page de maintenance en **200**, et Google enregistre un site normal avec une page
bizarre. Correctif : exclure la page de maintenance du bloc canonique.

```apache
RewriteCond %{REQUEST_URI} !^/maintenance[.]html$
```

Corollaire du meme mecanisme : Apache prefixe les variables d'environnement par `REDIRECT_` dans la
sous-requete. D'ou les **deux** lignes `Retry-After`, `env=JW_MAINT` et `env=REDIRECT_JW_MAINT`.
Avec une seule, l'en-tete manque sur la reponse reellement envoyee.

### Ce que ca ne fait pas

Ca coupe. Ca ne repare pas, ca ne desinfecte pas.

- **Ce n'est pas une remediation.** Un site pirate reste pirate derriere son 503. Les portes
  d'entree (identifiants voles, extension vulnerable, poste du client infecte) restent ouvertes.
  L'interrupteur achete du temps propre pour travailler, il ne remplace aucune etape du nettoyage.
- **Ca ne bloque pas l'attaquant deja dedans.** Un acces SFTP ou une session admin volee continue de
  fonctionner : le 503 concerne le trafic HTTP public, pas les autres canaux. La regle anti-reshell
  du fichier complet limite l'execution de PHP dans `uploads/`, ce qui est un durcissement, pas une
  garantie.
- **Ca ne protege pas le back-office.** `/wp-admin` repond 503 comme le reste. Utile pour couper les
  bots, inutile comme controle d'acces. Un vrai verrou, c'est un `.htpasswd` ou un filtre IP.
- **Ca ne surveille rien.** Personne n'est prevenu quand le site tombe pour de vrai. Brancher une
  sonde de disponibilite, et la configurer pour ne pas alerter sur les 503 volontaires.
- **Ca depend d'Apache.** Sous nginx pur, il n'y a pas de `.htaccess` : la logique se transpose en
  `try_files` et `return 503` dans le vhost, ce qui suppose un acces a la configuration serveur.
- **Ca laisse le cache tiers en travers.** Un CDN ou un cache de page peut continuer a servir des
  copies HTML pendant plusieurs minutes. Purger apres avoir bascule l'interrupteur.

### Licence et references

MIT, voir [LICENSE](LICENSE).

- Article detaille : <https://julienweb.fr/blog/mettre-site-wordpress-en-pause-htaccess/>
- Site pirate, plainte, prestataires agrees : <https://www.cybermalveillance.gouv.fr/>
- Auteur : Julien Guezennec, [julienweb.fr](https://julienweb.fr), contact@julienweb.fr

---

## EN

### The problem

No shared host ships a « pause my site » button. Not IONOS, not OVHcloud, not any of the others. The
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

503 is the only status Google reads as « come back later »: crawling pauses, rankings freeze,
nothing gets deindexed. `robots.txt` keeps being served. `maintenance.html` is self-contained
(no fonts, no CDN, no script, about 1.7 kB, `noindex, follow`) so it renders even when everything
else is dead. Both flag files return 403 over HTTP.

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

### What it does not do

It cuts. It does not fix, and it does not clean.

- Not a remediation: a hacked site stays hacked behind its 503, and the entry point stays open.
- Does not lock out an attacker already inside: stolen SFTP or admin sessions are unaffected.
- Not access control: `/wp-admin` returns 503 like everything else. Use `.htpasswd` or an IP filter.
- No monitoring: add an uptime probe, and configure it to ignore intentional 503s.
- Apache only: on pure nginx, transpose to `try_files` plus `return 503` in the vhost.
- Third-party caches and CDNs may keep serving HTML copies. Purge after flipping the switch.

### License

MIT, see [LICENSE](LICENSE). Written by Julien Guezennec, [julienweb.fr](https://julienweb.fr).
