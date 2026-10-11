# Budget de M'ziza

Application de suivi de budget domestique (page HTML unique, sans dépendance) : dettes avec jauge et courbe de remboursement, charges fixes, budget quotidien par catégorie (camembert et jauges), entrées d'argent, saisie rapide de dépenses et question à Claude.

- `index.html` : le code complet de l'application.
- Version en ligne : publiée comme artifact Claude (les données partagées y sont stockées dans la base de l'artifact).
- Hors de Claude, la page fonctionne en mode local (données enregistrées dans le navigateur) ; la saisie par ticket et la barre Claude n'y sont pas disponibles.

## Page de lancement (icône d'écran d'accueil)

Le dossier `docs/` contient une petite page statique avec l'icône de l'application. Elle sert à créer le raccourci sur l'écran d'accueil ; une fois lancée depuis l'icône, elle ouvre directement le budget. Pour la mettre en ligne : Settings > Pages > Deploy from a branch > `main` / `/docs`.
