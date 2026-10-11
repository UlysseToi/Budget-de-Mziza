# Budget de M'ziza — document de reprise pour Claude Code

Lis ce fichier en entier avant toute modification. Il résume tout ce qui a été décidé et construit avec l'utilisatrice·teur (Ulysse, ulyssebres@gmail.com) dans une longue conversation sur claude.ai.

## Règles de collaboration

- **Répondre en français**, ton simple et concis. Pas de jargon.
- L'utilisateur ne code pas : il décrit ce qu'il veut, souvent à la voix (phrases longues, parfois approximatives, mots mal transcrits : « Budget Mziza », « budget de MCP », « verre » = vert). Interpréter l'intention, ne pas reposer de questions inutiles.
- **Enregistrer chaque version sur GitHub** (demande explicite) : commit + push sur `main` à chaque changement. Dépôt : https://github.com/UlysseToi/Budget-de-Mziza
- Toujours vérifier visuellement (capture d'écran Playwright, largeur téléphone 420 px) avant de dire que c'est fait, et le dire honnêtement quand une chose n'a pas pu être testée (micro, installation sur téléphone, compte gratuit Claude…).

## Le projet

Application de suivi de budget domestique pour **M'ziza** : entrées d'argent (salaire + participations des enfants), charges fixes, dettes (jauge + courbe de remboursement), budget quotidien par catégorie (camembert + jauges), saisie rapide de dépenses (texte / ticket de caisse), barre « question à Claude » qui **modifie le budget** et garde un historique.

Ce n'est PAS un tableur (un premier tableau Excel avait été fait puis rejeté).

### Fichiers du dépôt

- `index.html` : **toute l'application** (une seule page HTML, CSS + JS vanilla, aucune bibliothèque). ~100 Ko. C'est le fichier à modifier.
- `docs/` : petite page de lancement statique (GitHub Pages) avec `manifest.webmanifest` et l'icône, pour créer un raccourci d'écran d'accueil. À activer côté GitHub : Settings > Pages > main / `/docs` (pas encore fait par l'utilisateur ; si le dépôt est privé, Pages peut exiger un abonnement).
- `icones/` : icône (maison avec un cœur, fond vert) en SVG + PNG 180/192/512.
- `README.md`.

### Où l'application tourne

Elle est publiée comme **artifact claude.ai** : https://claude.ai/artifact/JZUEooVNtjQxttN9E8CQr5 (privée tant que l'utilisateur ne la partage pas via le menu Partager de la page ; seul lui peut régler le partage). Capacités déclarées : `db` (base partagée en temps réel) et `sample` avec `images: true` (appeler Claude depuis la page, dont la lecture de tickets).

**Point d'attention pour Claude Code sur PC** : publier/republier l'artifact se fait avec l'outil `Artifact` de claude.ai (action publish, même chemin de fichier = même URL, avec `capabilities: {"db":{}, "sample":{"images":true}}` à repasser explicitement). Si cet outil n'est pas disponible sur le PC, éditer `index.html`, pousser sur GitHub, puis demander à l'utilisateur de faire republier depuis claude.ai (ou lui proposer de tester en local). En dehors de claude.ai la page fonctionne en **mode local** (données dans localStorage, `window.claude` absent) : pas de base partagée, pas de barre Claude, pas de scan de ticket. Hébergée ailleurs (GitHub Pages), l'app perdrait ces fonctions.

Pour tester en local : `node --check` sur le script extrait, puis Playwright sur `file:///.../index.html` avec localStorage `budget-mziza-v2` pré-rempli et, au besoin, un faux `window.claude` (`use('sample')` renvoyant une fonction avec `.json()` et `.limits()`).

## Données (base `db` de l'artifact)

Document `app/config` :
```
{ startDate: "2026-10-11",
  items: [ {id, type:"income"|"fixed"|"variable", name, amount, note, group?:"salary"|"family"|"other", active?:false} ],
  debts: [ {id, name, start, monthly, from:"YYYY-MM", skipped:["YYYY-MM"], note} ] }
```
- `income` : entrées mensuelles systématiques (salaire 1600 ; Marcio 100, Airton 200, Alyssa 200, Andreia 0, Auréa 0).
- `fixed` : charges fixes prélevées d'office, sans statut (loyer 998, EDF 273, assurance 39, MAIF 90, SFR 45, eau 36, banque 21 = 1502 €).
- `variable` : catégories du budget quotidien. **Leur `amount` est une PART EN %** (pas un montant). Le budget en euros est calculé : (entrées − charges fixes − mensualités de dettes) réparti entre les catégories **cochées** (`active !== false`) au prorata des parts, arrondi à l'euro (reste attribué à la plus grosse part). Catégories actuelles : Courses 53, Carburant 15 (décoché : pas de voiture), Frais de santé 12, Maison et vêtements 12, Extras 8. ids : `i-courses`, `i-transport`, `i-sante`, `i-maison`, `i-loisirs`.
- Dettes : Banque Accord 1500 € / 80 €/mois ; Cofidis 3500 € / 107 ; Avance sur salaire (employeur) 1500 / 150 ; Arriéré de loyer 550 / 92 (plan d'apurement sur 6 mois). Tous prélevés automatiquement chaque mois **à partir de novembre 2026** (`from:"2026-11"`). Total mensuel 429 €. Disponible pour le quotidien ≈ 2100 − 1502 − 429 = **169 €/mois** (serré : 200 € de courses ne rentre pas).
- Remboursement : pas de bouton « enregistrer un remboursement ». Un mois est compté payé dès qu'il commence, sauf s'il est dans `skipped` (bouton « Signaler un mois non payé »). `payCount`, `paidAt`, `endMonthOf` calculent le payé, le restant et la fin prévue.

Collection `tx` (une dépense par document) : `{date, kind:"expense", ref:<id item>, label, amount, note}`.

Collection `hist` (historique des demandes à Claude, 48 h) : `{q, t (ms), lines:[{t, miss?}], before:{items,debts}, after:{items,debts}, undone}`.

Écritures de `app/config` : le document entier est remplacé (dernier écrit gagne) ; les dépenses sont des documents séparés (pas de conflit). Écrire dans la base depuis Claude : outil `ArtifactData` avec `if_version` pour un document existant.

Mode local de secours : localStorage `budget-mziza-v2` (cfg + tx), `budget-mziza-hist`.

## Interface (ce qui a été demandé, dans l'ordre d'importance)

- Titre « Budget de M'ziza » = **bouton pastille verte douce avec icône maison + cœur**, plus gros que le texte, espacé du sous-titre ; un clic ramène à l'accueil (aucun onglet ouvert).
- **Accueil** : (barre Claude), bloc hero (reste à rembourser en très gros, jauge circulaire avec % remboursé, déjà remboursé, dette de départ, par mois, fin prévue), puis les **4 boutons**, puis la **courbe de remboursement**. **Aucun onglet n'est déplié par défaut** ; recliquer sur l'onglet actif le referme. La courbe n'apparaît **que sur l'accueil**. Elle a des **points de couleur à la fin de chaque dette** et une légende « Fin de chaque dette » dessous.
- **4 boutons** : Dettes, Charges fixes, Budget quotidien, Entrées d'argent. Petits, avec icône dans un carré, **titre gros et gras, somme dessous petite et grise**, **camaïeu de verts du plus clair au plus foncé** (variables `--t1..--t4`, `--t1s..--t4s`), contour de leur vert (le plus clair ne doit pas se fondre dans le fond), plein au survol/sélection.
- **Thème** : tons verts doux, fond blanc-vert clair (`--bg:#EDF3EE`, surfaces `#FBFDFB`) ; l'utilisateur a **rejeté** une version plus verte/moins blanche : rester sur vert et blanc. Polices Bricolage Grotesque + Hanken Grotesk. Mode sombre prévu. Mobile d'abord, gouttière 16 px.
- **Dettes** : liste de dettes (reste, barre, départ, déjà remboursé, mensualité, fin prévue) + « Signaler un mois non payé ». Pas de « Modifier », pas d'historique de remboursements.
- **Charges fixes** : liste simple (nom, montant, Modifier), ajout/suppression possibles ; **aucun statut**, pas de « marquer payé ».
- **Entrées d'argent** : liste groupée (salaire / enfants / autres), sans statut « attendu/reçu ». Entrées considérées systématiques chaque mois.
- **Budget quotidien** : champ « Ajouter une dépense » **en haut** (texte « 100 euros courses » classé automatiquement, bouton micro, bouton « Scanner un ticket de caisse » si les images sont disponibles) — **c'est le seul endroit pour ajouter une dépense**. Carte « Répartition du budget » : **gros camembert** à couleurs bien distinctes + **légende à côté** (sur téléphone : dessous) où chaque ligne = case à cocher, pastille couleur, nom (cliquable pour changer la part), **jauge**, montant. Quand une catégorie n'a aucune dépense : le budget est en gros et « il reste X € » en petit ; **dès la première dépense** : « il reste X € » passe en **gros et gras** (vert, rouge si dépassé) et le budget devient petit. Décocher une catégorie recalcule les autres automatiquement. Puis liste « Dépenses du mois ».
- **Barre Claude** (en haut, sticky, visible partout) : voir plus bas. Bouton micro à côté.
- **Historique** : petite pastille discrète « Historique (n) », **entourée en permanence**, qui se met **en relief au survol** ; **n'apparaît qu'après une première demande** ; se déplie en liste (plus récente d'abord) avec, par demande, ses modifications et « Revenir à avant » / « Rétablir ». Gardé 48 h puis purgé.
- Bloc « Ajouter un raccourci sur le téléphone » (replié) en bas de l'accueil.

## Barre « question à Claude » (`askClaude`)

- Utilise `claude.use('sample')` → `sample.json(prompt, {modelTier:'default', cache:false, signal})`. Dépense le quota Claude **du visiteur** ; première utilisation = demande d'autorisation ; il faut que le visiteur ait un compte Claude (gratuit : non vérifié).
- Le prompt reçoit l'état complet en JSON (ids inclus) + l'historique actif, et doit répondre `{"reponse", "changes":[…], "annuler": null|N|"toutes"}`.
- **Les `changes` sont appliqués IMMÉDIATEMENT** (pas de fenêtre « valider », l'utilisateur l'a rejeté deux fois) puis enregistrés dans `hist` ; seule une courte confirmation (toast avec bouton Annuler) s'affiche. Une carte de réponse n'apparaît que pour une pure question de calcul (« combien faudrait-il… »).
- Opérations : `set_amount`, `add_item` (income|fixed), `remove_item`, `set_share` (part % d'une catégorie), `set_active`, `set_debt` (monthly/start), `add_debt`, `remove_debt`. Cibles par id ou nom (`findBy`). Éléments introuvables listés « ignoré ».
- « Annule la dernière » / « annule les deux dernières » / « reviens en arrière » → `annuler` → `revertTo`. Revenir en arrière restaure `before` de la demande visée et marque celle-ci et les suivantes `undone` ; `redoTo` restaure `after`. Une nouvelle demande supprime les demandes annulées. Limite connue : une modification manuelle faite entre-temps est écrasée par la restauration.
- Cas d'usage voulus : « si Andreia et Auréa donnent chacune 50 € », « si M'ziza voulait 400 € de courses, combien devrait-elle gagner en plus / être aidée / renégocier ses dettes ».
- Ne simule pas les dépenses du quotidien (`tx`).

## Saisie de dépense

- Texte : dictionnaire de mots-clés `KW` par catégorie (accents ignorés), montant par regex (priorité au nombre suivi de €/euro), « hier » = date de la veille, sinon classification par Claude (`classifyWithClaude`, tier quick), sinon fenêtre pour choisir la catégorie. Seules les catégories cochées sont proposées.
- Ticket : photo redimensionnée (1400 px, JPEG) → `sample.json` avec `images` → fenêtre de vérification préremplie (montant, commerce, date, catégorie).
- **Voix** : `webkitSpeechRecognition` est tenté mais **bloqué dans l'iframe de l'artifact** (pas de capacité micro). Repli : place le curseur dans le champ et affiche la consigne selon la plateforme (Windows + H, micro du clavier iPhone/Android). Il n'est pas possible de déclencher la dictée de l'OS depuis une page web.

## Ce qui a été fait / exclu

- Fait : tout ce qui précède, publié (versions 1 à 20 de l'artifact), commité sur GitHub.
- Impossible depuis une page web : créer automatiquement une icône d'écran d'accueil ou installer Claude ; changer l'icône du raccourci d'une page claude.ai (elle est choisie par claude.ai) ; lire un compte bancaire (idée évoquée : Open Banking DSP2 via un agrégateur type Powens/Bridge, lecture seule, mais il faudrait un serveur, des clés et un contrat — non fait ; alternative simple proposée : import d'un export CSV de la banque, classé automatiquement, sans aucun accès au compte — **à proposer/construire si l'utilisateur le veut**).
- Partage : la page est **privée**, l'utilisateur doit la partager lui-même ; sur claude.ai, la base est partagée en temps réel entre tous ceux qui ont accès, écriture réservée à ceux qui ont le droit d'écrire (niveau « interact » testé OK). Ouverture par quelqu'un sans compte Claude : non vérifié, probablement impossible.

## À faire / idées en attente

1. L'utilisateur doit activer GitHub Pages (`main` / `/docs`) s'il veut la page de lancement avec l'icône ; ensuite lui donner l'URL à envoyer (`https://ulyssetoi.github.io/Budget-de-Mziza/`, à confirmer).
2. Demander à l'utilisateur de tester sur un vrai téléphone : barre Claude (une vraie demande qui applique bien les changements), micro, scan de ticket, partage avec une autre personne. Jamais testés hors Playwright/stubs.
3. Étiquettes de l'axe horizontal de la courbe qui se chevauchent un peu à droite (« déc. 28 / juil. 29 ») : à corriger.
4. Éventuel import CSV bancaire (voir plus haut).
5. Ajuster les parts du budget quotidien si l'utilisatrice donne ses vrais chiffres.
