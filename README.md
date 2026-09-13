# ObjectAI

Application mobile qui identifie un objet à partir d'une photo et génère une fiche
intelligente (valeur estimée, conseils d'achat/vente, entretien, historique).

## Structure

```
objectai/
├── apps/
│   ├── mobile/   # App Expo (React Native + TypeScript)
│   └── api/      # Backend Express + Prisma (PostgreSQL)
└── packages/
    └── shared-types/  # Types partagés
```

## Démarrer le backend

```bash
cd objectai
docker compose up -d          # lance PostgreSQL en local (port 5432)

cd apps/api
cp .env.example .env          # DATABASE_URL est déjà pré-rempli pour le docker-compose ci-dessus
                               # renseigne aussi ANTHROPIC_API_KEY et des secrets JWT à toi
npm install
npx prisma migrate dev --name init
npm run dev                   # démarre sur http://localhost:4000
```

## Démarrer l'app mobile

```bash
cd apps/mobile
npm install
```

Crée un fichier `apps/mobile/.env` avec :
```
EXPO_PUBLIC_API_URL=http://<IP_LOCALE_DE_TA_MACHINE>:4000
```
(Ne pas utiliser `localhost` : un téléphone physique ne peut pas y accéder. Trouve ton IP locale avec
`ipconfig getifaddr en0` sur Mac, ou `ipconfig` sur Windows.)

```bash
npm run start                 # ouvre Expo ; scanne le QR code avec l'app Expo Go sur ton téléphone
```

## Mettre le projet sur GitHub

```bash
cd objectai
git init
git add .
git commit -m "Initial commit: ObjectAI v0.1"
git branch -M main
git remote add origin https://github.com/<ton-compte>/objectai.git
git push -u origin main
```

Un workflow GitHub Actions (`.github/workflows/ci.yml`) est déjà en place : il vérifie automatiquement
que le backend et le mobile compilent sans erreur à chaque push.

## Étape 1 : création de compte ✅

- Backend : `/auth/signup`, `/auth/login`, `/auth/refresh`, `/users/me`
- Mobile : écran d'inscription/connexion, session persistée (AsyncStorage), écran d'accueil avec le bouton central « Analyser un objet »

## Étape 2 : capture photo + analyse IA + résultats ✅

- Backend : `POST /objects/analyze` (protégé) — envoie la photo à Claude (vision) via `@anthropic-ai/sdk`,
  parse la réponse JSON structurée, journalise l'appel dans `AiAnalysisLog`. Nécessite `ANTHROPIC_API_KEY` dans `.env`.
- Mobile : écran « Capture » (caméra ou galerie via `expo-image-picker`), écran « Résultat » affichant la fiche
  générée (nom, marque, modèle, catégorie, description, caractéristiques, état, valeur estimée avec disclaimer),
  tous les champs sont éditables avant sauvegarde.

Vérifié à chaque étape : `tsc --noEmit` sans erreur côté `api` et `mobile`.

⚠️ Le bouton « Enregistrer dans Mes objets » est présent mais pas encore branché : la persistance arrive à l'étape 3.

## Étape 3 : enregistrement + Mes objets + fiche détaillée ✅

- Backend : `POST /objects` (upload photo + création avec historique initial), `GET /objects` (liste),
  `GET /objects/:id` (détail + historique). Stockage photo via `storageService`, une interface avec une
  implémentation locale (`/uploads`, servi en statique) — à remplacer par S3 en production sans toucher
  au reste du code.
- Mobile : le bouton « Enregistrer » de l'écran Résultat est maintenant fonctionnel. Écran « Mes objets »
  (liste avec pull-to-refresh, état vide géré) et écran de fiche objet détaillée (description,
  caractéristiques, valeur, historique).

Vérifié à chaque étape : `tsc --noEmit` sans erreur côté `api` et `mobile`.

## Étape 4 : documents associés à un objet ✅

- Backend : `POST /objects/:id/documents` (ajout, upload via `storageService.saveBase64File`, accepte images et PDF),
  `GET /objects/:id/documents`, `DELETE /objects/:id/documents/:docId`. Vérification que l'objet appartient
  bien à l'utilisateur avant toute opération.
- Mobile : section « Documents » dans la fiche objet — ajout (choix du type : facture, notice, certificat,
  garantie, photo, autre, puis sélection dans la galerie), ouverture du fichier, suppression avec confirmation.

Vérifié à chaque étape : `tsc --noEmit` sans erreur côté `api` et `mobile`.

## Étape 5 : rappels d'entretien ✅

- Backend : `POST /objects/:id/reminders`, `GET /objects/:id/reminders`, `DELETE /objects/:id/reminders/:reminderId`.
  Vérification de propriété factorisée dans `lib/ownership.ts` (réutilisée par documents et rappels).
- Mobile : section « Entretien & rappels » dans la fiche objet — formulaire (message libre + échéance
  parmi 7 jours / 30 jours / 3 mois / 6 mois / 1 an), suppression avec confirmation. Une vraie notification
  locale est programmée via `expo-notifications` à la date choisie (permission demandée à la volée),
  et annulée si le rappel est supprimé.

Vérifié à chaque étape : `tsc --noEmit` sans erreur côté `api` et `mobile`.

## Étape 6 : Achat et Vente ✅

- Backend :
  - `POST /purchase/analyze` — analyse une photo d'objet/annonce (+ prix demandé optionnel) et renvoie
    un avis, un verdict sur le prix, les points à vérifier, les questions à poser au vendeur, et les
    facteurs de risque sur la valeur.
  - `POST /objects/:id/listing` — génère une annonce de vente (titre, description, caractéristiques,
    accessoires, points importants, prix conseillé) à partir de la fiche d'un objet déjà enregistré.
- Mobile :
  - Écran « Analyser un achat » (photo + prix optionnel) → écran de résultat avec verdict coloré,
    checklist et questions à poser.
  - Bouton « Vendre cet objet » sur la fiche objet → écran d'annonce entièrement éditable (titre,
    description, caractéristiques, accessoires, prix) avec bouton « Partager » (aucune publication
    automatique, conformément à la consigne).

Vérifié à chaque étape : `tsc --noEmit` sans erreur côté `api` et `mobile`.

Avec cette étape, toutes les fonctionnalités métier du cahier des charges sont couvertes : identification,
valeur, achat, vente, Mes objets, historique, entretien/rappels, documents.

## Navigation par onglets + écran Profil ✅

- Barre d'onglets (`@react-navigation/bottom-tabs`) : **Accueil / Analyser / Mes objets / Profil**, conforme
  à la section « Interface » du cahier des charges. L'onglet « Analyser » redirige directement vers l'écran
  de capture (hors barre d'onglets, comme les écrans Résultat, Fiche objet, Achat, Annonce).
- Nouvel écran **Profil** : identité de l'utilisateur, déconnexion (avec confirmation), version de l'app.

Vérifié : `tsc --noEmit` sans erreur côté `api` et `mobile`.

## Mise en route facile (Docker + CI) ✅

- `docker-compose.yml` à la racine : lance une base PostgreSQL locale en une commande (`docker compose up -d`),
  avec des identifiants déjà alignés sur `.env.example`.
- `.github/workflows/ci.yml` : vérifie automatiquement la compilation TypeScript du backend et du mobile
  à chaque push/pull request sur GitHub.
- Instructions complètes de démarrage et de mise sur GitHub ci-dessous.

## Prochaines étapes (production)

- Remplacer le stockage local (`/uploads`) par un vrai bucket S3 (l'interface `StorageService` est déjà prête)
- Paiement (Stripe) pour une future version premium
- Notifications push serveur (au-delà des rappels locaux) et suivi d'usage IA plus poussé
- Authentification via un vrai fournisseur (Supabase Auth / Firebase Auth) plutôt que le JWT maison de la V1
- Déploiement (CI/CD, build EAS pour Android/iOS) et publication sur les stores
