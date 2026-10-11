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
- `docs/` : petite page de lancement statique (GitHub Pages) avec `manifest.webmanifest` et l'icône, pour créer un raccourci d'écran d'accueil. **En ligne** (activé le 2026-10-11, main / `/docs`) : https://ulyssetoi.github.io/Budget-de-Mziza/
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
- `variable` : catégories du budget quotidien. **Leur `amount` est une PART EN %** (pas un montant). Le budget en euros est calculé : (entrées − charges fixes − mensualités de dettes) réparti entre les catégories **cochées** (`active !== false`) au prorata des parts, arrondi à l'euro (euros restants donnés aux plus grandes fractions, pour qu'un montant tapé en euros reste exact). **Depuis la version 22, l'utilisateur modifie une catégorie EN EUROS** en touchant son montant (`setCategoryEuros`) : la catégorie prend ce montant, les autres catégories cochées se partagent le reste dans leurs proportions actuelles, et les parts sont réenregistrées en % (total 100). Catégories actuelles : Courses 53, Carburant 15 (décoché : pas de voiture), Frais de santé 12, Maison et vêtements 12, Extras 8. ids : `i-courses`, `i-transport`, `i-sante`, `i-maison`, `i-loisirs`.
- **Depuis la version 26, chaque mois ne compte que les dettes réellement prélevées ce mois-là** (`debtsDue(m)` : commencées, mois non signalé impayé, pas encore soldées ; dernière mensualité réduite au reste) — utilisé par `dailyStats`. Les dettes commencent en octobre 2026 (voir plus bas), donc octobre compte bien les 429 €. Le bandeau « Par mois » du haut et le contexte envoyé à Claude gardent le total des mensualités.
- Dettes : Banque Accord 1500 € / 80 €/mois ; Cofidis 3500 € / 107 ; Avance sur salaire (employeur) 1500 / 150 ; Arriéré de loyer 550 / 92 (plan d'apurement sur 6 mois). Tous prélevés automatiquement chaque mois **à partir d'octobre 2026** (`from:"2026-10"`, corrigé le 2026-10-11 à la demande de l'utilisateur : « il y a bien des dettes prélevées en octobre »). Total mensuel 429 €. Disponible pour le quotidien ≈ 2100 − 1502 − 429 = **169 €/mois** (serré : 200 € de courses ne rentre pas).
- Remboursement : pas de bouton « enregistrer un remboursement ». Un mois est compté payé dès qu'il commence, sauf s'il est dans `skipped` (bouton « Signaler un mois non payé »). `payCount`, `paidAt`, `endMonthOf` calculent le payé, le restant et la fin prévue.

`app/config.od` (version 24) : découvert au début de chaque mois `{"YYYY-MM": montant}` ; il est retiré de l'argent du quotidien de ce mois (`odOf`, `dailyStats`). Saisi depuis la carte « Ce mois-ci » de l'accueil ou la page Budget quotidien. L'utilisateur ne voulait PAS le mettre en dette : il veut que ça se répercute sur le mois et suivre où elle en est dans le mois.

Collection `tx` (une dépense par document) : `{date, kind:"expense", ref:<id item>, label (nom de la catégorie), name (nom de la dépense, depuis v25), amount, note}`.

Collection `hist` (historique de TOUTES les demandes à Claude, 30 jours, 120 max) : `{q, t (ms), st:'applied'|'proposed'|'question' (absent = applied), lines:[{t, miss?}], ops, a (réponse), before, after, undone, dead}`. `dead` = annulée puis dépassée par une nouvelle application (plus de « Rétablir »).

Document `app/lastdoc` : dernière analyse de document `{name, t, type:'releve'|'facture'|'ticket'|'autre', resume, alerts:[{titre, explication, ops, state:''|'ok'|'no'}]}`. Les alertes sans `state` font la pastille rouge sur « Charges fixes ».

Écritures de `app/config` : le document entier est remplacé (dernier écrit gagne) ; les dépenses sont des documents séparés (pas de conflit). Écrire dans la base depuis Claude : outil `ArtifactData` avec `if_version` pour un document existant.

Mode local de secours : localStorage `budget-mziza-v2` (cfg + tx), `budget-mziza-hist`, `budget-mziza-doc`.

## Interface (ce qui a été demandé, dans l'ordre d'importance)

- Titre « Budget de M'ziza » = **bouton pastille verte douce avec icône maison + cœur**, plus gros que le texte, espacé du sous-titre ; un clic ramène à l'accueil (aucun onglet ouvert).
- **Accueil** : (barre Claude), bloc hero (reste à rembourser en très gros, jauge circulaire avec % remboursé, déjà remboursé, dette de départ, par mois, fin prévue), puis la carte **« Ce mois-ci »** (entrées, charges fixes, dettes, découvert à combler, pour le quotidien, déjà dépensé, et en gros « Il reste / Il manque pour finir le mois »). Version 26 : flèches ← → pour changer de mois (titre « Ce mois-ci », sinon « Mois de novembre », « Mois d'avril »… + « Revenir à ce mois-ci » ; mois passés : « Il est resté / Il a manqué ») ; si « Pour le quotidien » est négatif : **croix rouge ✕** au lieu du montant, le manque reste affiché en bas, puis les **4 boutons**, puis la **courbe de remboursement**. La courbe a des **points de couleur à la fin de chaque dette** et une légende dessous ; **un toucher l'ouvre en plein écran** (défilement horizontal, boutons − / +, toucher un mois = détail : reste total et par dette, dette qui se termine ce mois-là).
- **Chaque bouton ouvre une page à part** (depuis le 2026-10-11, version 21) : on ne voit plus que le titre (retour accueil), la barre Claude, la pastille Historique, « ← Retour à l'accueil » et la page. Le bouton retour du téléphone revient à l'accueil (history.pushState).
- **4 boutons** : Dettes, Charges fixes, Budget quotidien, Entrées d'argent. Petits, avec icône dans un carré, **titre gros et gras, somme dessous petite et grise** (Dettes : restant ET « par mois », les deux sont obligatoires), **camaïeu de verts du plus clair au plus foncé** (variables `--t1..--t4`, `--t1s..--t4s`), contour de leur vert (le plus clair ne doit pas se fondre dans le fond), plein au survol/sélection.
- **Thème** : tons verts doux, fond blanc-vert clair (`--bg:#EDF3EE`, surfaces `#FBFDFB`) ; l'utilisateur a **rejeté** une version plus verte/moins blanche : rester sur vert et blanc. Polices Bricolage Grotesque + Hanken Grotesk. Mode sombre prévu. Mobile d'abord, gouttière 16 px.
- **Dettes** : liste de dettes (reste, barre, départ, déjà remboursé, mensualité, fin prévue, note) + « Signaler un mois non payé » + « Ajouter une dette ». **Toucher une dette** ouvre une fenêtre : nom, montant de départ, mensualité, premier prélèvement, **note libre** (ex. « payé le 5 novembre par Airton »), supprimer.
- **Charges fixes** : liste simple (nom, montant, Modifier), ajout/suppression possibles ; **aucun statut**. Version 23 : **poignée ⠿ pour ranger les lignes par glisser-déposer** (pointer events, `touch-action:none`, flèches haut/bas au clavier ; l'ordre = ordre des items `fixed` dans `app/config`, `saveFixedOrder`) et **barre de recherche** qui surligne les charges trouvées (nom, note ou montant, accents ignorés), grise les autres et fait défiler jusqu'à la première. Carte **« Ajouter un document »** (Prendre en photo / Choisir un fichier : photo, PDF, CSV/TXT) : Claude identifie le type, résume et liste les **incohérences** avec le budget (charges fixes, entrées, mensualités, dépassements) ; chaque alerte a « Accepter la modification » / « Ne pas modifier ». Un lien ne peut pas être ouvert (réseau bloqué dans l'artifact) : il faut télécharger le fichier. PDF lus dans la page avec pdf.js 3.11.174 (cdnjs ; le script worker est chargé d'abord pour éviter le Worker cross-origin) ; PDF scanné → pages en images ; Excel non lu (demander PDF/CSV).
- **Entrées d'argent** : liste groupée (salaire / enfants / autres), sans statut. **Toucher une ligne** (enfant ou salaire) ouvre la fenêtre de modification du montant.
- **Budget quotidien** : (version 25) **simple bouton « + Ajouter une dépense »** sous le choix du mois (+ « Scanner un ticket » si les images sont disponibles) — **c'est le seul endroit pour ajouter une dépense**. Il ouvre un formulaire : Nom de la dépense (curseur dedans), Montant, Catégorie (devinée d'après le nom avec `KW`/`matchCategory` tant qu'on ne la choisit pas soi-même), Date, Remarque. L'ancien champ texte « 100 euros courses » a été supprimé à la demande de l'utilisateur. Le nom est stocké dans `tx.name` (affiché en gras, catégorie dessous). Carte « Répartition du budget » : **gros camembert** à couleurs bien distinctes + **légende à côté** (sur téléphone : dessous) où chaque ligne = case à cocher, pastille couleur, nom et **montant cliquables pour changer le budget en euros**, **jauge**, montant. Quand une catégorie n'a aucune dépense : le budget est en gros et « il reste X € » en petit ; **dès la première dépense** : « il reste X € » passe en **gros et gras** (vert, rouge si dépassé) et le budget devient petit. Décocher une catégorie recalcule les autres automatiquement. Puis liste « Dépenses du mois ».
- **Barre Claude** (en haut, sticky, visible partout) : voir plus bas. Bouton micro à côté.
- **Historique** : petite pastille « Historique (n) », entourée, en relief au survol, visible dès qu'il y a une demande ; ouvre une **fenêtre fixe qui défile** (plus récente d'abord) : demandes appliquées (« Revenir à avant » / « Rétablir »), propositions non appliquées (« Appliquer maintenant »), simples questions (réponse). Gardé 30 jours. Bouton « Supprimer l'historique » (avec confirmation) en bas de la fenêtre : efface les demandes, pas les modifications.
- Bloc « Ajouter un raccourci sur le téléphone » (replié) en bas de l'accueil.

## Barre « question à Claude » (`askClaude`)

- Utilise `claude.use('sample')` → `sample.json(prompt, {modelTier:'default', cache:false, signal})`. Dépense le quota Claude **du visiteur** ; première utilisation = demande d'autorisation ; il faut que le visiteur ait un compte Claude (gratuit : non vérifié).
- Le prompt reçoit l'état complet en JSON (ids inclus) + l'historique actif, et doit répondre `{"reponse", "changes":[…], "annuler": null|N|"toutes"}`.
- **Changement de décision le 2026-10-11** : l'utilisateur VEUT maintenant une fenêtre « Proposition de Claude » avec l'explication détaillée, la liste « Ce qui changera si vous validez », l'effet sur le disponible du quotidien, et **Valider / Fermer**. Rien n'est modifié avant Valider ; une proposition fermée reste dans l'historique (« proposée »). Les questions sans modification s'affichent dans la même fenêtre (« Réponse de Claude »). Une demande d'annulation passe aussi par Valider.
- Opérations : `set_amount`, `add_item` (income|fixed), `remove_item`, `set_share` (part % d'une catégorie), `set_active`, `set_debt` (monthly/start), `add_debt`, `remove_debt`. Cibles par id ou nom (`findBy`). Éléments introuvables listés « ignoré ».
- « Annule la dernière » / « annule les deux dernières » / « reviens en arrière » → `annuler` → `revertTo`. Revenir en arrière restaure `before` de la demande visée et marque celle-ci et les suivantes `undone` ; `redoTo` restaure `after`. Une nouvelle application rend les demandes annulées définitives (`dead`), sans les supprimer. Limite connue : une modification manuelle faite entre-temps est écrasée par la restauration.
- Cas d'usage voulus : « si Andreia et Auréa donnent chacune 50 € », « si M'ziza voulait 400 € de courses, combien devrait-elle gagner en plus / être aidée / renégocier ses dettes ».
- Ne simule pas les dépenses du quotidien (`tx`).

## Saisie de dépense

- Formulaire (v25) : catégorie suggérée par le dictionnaire de mots-clés `KW` (accents ignorés) à partir du nom. Seules les catégories cochées sont proposées.
- Ticket : photo redimensionnée (1400 px, JPEG) → `sample.json` avec `images` → fenêtre de vérification préremplie (montant, commerce, date, catégorie).
- **Voix** : `webkitSpeechRecognition` est tenté mais **bloqué dans l'iframe de l'artifact** (pas de capacité micro). Repli : place le curseur dans le champ et affiche la consigne selon la plateforme (Windows + H, micro du clavier iPhone/Android). Il n'est pas possible de déclencher la dictée de l'OS depuis une page web.

## Ce qui a été fait / exclu

- Fait : tout ce qui précède, publié (versions 1 à 27 de l'artifact), commité sur GitHub. Version 21 testée dans un navigateur local à 420 px avec un faux Claude (pages, dettes, entrées, proposition/validation, historique, PDF texte, CSV, courbe plein écran) ; pas encore testée avec le vrai Claude ni sur téléphone.
- La page est maintenant partagée « toute personne ayant le lien » (réglé par l'utilisateur).
- Impossible depuis une page web : créer automatiquement une icône d'écran d'accueil ou installer Claude ; changer l'icône du raccourci d'une page claude.ai (elle est choisie par claude.ai) ; lire un compte bancaire (idée évoquée : Open Banking DSP2 via un agrégateur type Powens/Bridge, lecture seule, mais il faudrait un serveur, des clés et un contrat — non fait ; alternative simple proposée : import d'un export CSV de la banque, classé automatiquement, sans aucun accès au compte — **à proposer/construire si l'utilisateur le veut**).
- Partage : la page est **privée**, l'utilisateur doit la partager lui-même ; sur claude.ai, la base est partagée en temps réel entre tous ceux qui ont accès, écriture réservée à ceux qui ont le droit d'écrire (niveau « interact » testé OK). Ouverture par quelqu'un sans compte Claude : non vérifié, probablement impossible.

## À faire / idées en attente

1. ~~Activer GitHub Pages~~ : fait le 2026-10-11, page vérifiée (icône maison + cœur OK) : https://ulyssetoi.github.io/Budget-de-Mziza/ . Reste à tester l'ajout à l'écran d'accueil sur un vrai téléphone.
2. Demander à l'utilisateur de tester sur un vrai téléphone : barre Claude (proposition + Valider), analyse d'un vrai relevé PDF/CSV et d'une photo de facture, micro, scan de ticket, partage. Jamais testés hors stubs.
3. ~~Étiquettes de la courbe qui se chevauchent~~ : corrigé (version 21).
4. ~~Import CSV bancaire~~ : fait via « Ajouter un document » (version 21).
5. Ajuster les parts du budget quotidien si l'utilisatrice donne ses vrais chiffres.
