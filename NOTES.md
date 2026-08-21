# Notes de déploiement

## 1. Correctif du comptage de caractères X (t.co)

**Symptôme initial** : le compteur de caractères pour X ne tenait pas compte du
raccourcissement des URLs par X (`t.co` compte toute URL pour 23 caractères),
ce qui faisait passer des posts valides pour "trop longs".

**Statut** : 4 bugs distincts identifiés et corrigés dans le même sous-système,
sur la branche `fix/x-character-count` du fork
[bernig/postiz-app](https://github.com/bernig/postiz-app), basée sur `main`
upstream (`gitroomhq/postiz-app`). PR upstream pas encore ouverte (en attente
de validation utilisateur).

| # | Fichier | Bug | Commit |
|---|---|---|---|
| 1 | `apps/frontend/.../new-launch/editor.tsx` | Le compteur affiché à l'édition comptait la longueur brute du texte pour toutes les plateformes, y compris X, au lieu d'utiliser `weightedLength()` (t.co = 23). | `a2e1ba08` |
| 2 | `libraries/nestjs-libraries/.../posts.service.ts` | La validation serveur (`tooLong`) calculait bien `weightedLength()` pour X, mais prenait ensuite `Math.max(weighted, raw)` — qui retombe systématiquement sur la longueur brute dès qu'une URL raccourcit le poids, rendant le calcul pondéré mort-code. | `a2e1ba08` |
| 3 | `libraries/helpers/.../count.length.ts` (`textSlicer`) | L'aperçu surlignait en rouge la fin du texte alors que le post était valide : quand le tweet passait la validation pondérée, la fonction retournait quand même la limite brute (280) comme index de découpe au lieu de `text.length`. | `c21ae917` |
| 4 | même fichier | Off-by-one : pour un post réellement trop long, `validRangeEnd` (index **inclusif** du dernier caractère valide) était utilisé directement comme fin de découpe **exclusive**, surlignant 1 caractère de trop (2 caractères au lieu d'1 pour un dépassement d'1 caractère). | `c892603b` |
| 5 | `apps/frontend/.../launches/information.component.tsx` | En mode d'édition globale (multi-plateformes), le compteur agrégé comparait la longueur brute du texte à la limite de **chaque** plateforme sélectionnée, y compris X — sélectionner X faisait donc passer le post en rouge même s'il était valide une fois pondéré, jusqu'à "personnaliser" l'onglet X (qui l'exclut alors de la vérification agrégée). | `72193e0f` |

**Déploiement local pendant les tests** : image construite localement avec
`docker buildx build -f Dockerfile.dev -t postiz-app:x-char-count-fix --build-arg NEXT_PUBLIC_VERSION=v1.47.0-x-char-count-fix .`
depuis le checkout `/var/www/postiz-app` (clone séparé de `postiz-app`, ce
repo-ci `postiz-docker-compose` ne contient que la config Docker Compose).
`docker-compose.yaml` pointe actuellement sur `postiz-app:x-char-count-fix`
au lieu de `ghcr.io/gitroomhq/postiz-app:latest` — à revenir à l'image
officielle une fois la PR upstream mergée (ou à reconstruire à chaque mise à
jour tant qu'elle ne l'est pas).

**Validation** : `tsc --noEmit` propre sur `apps/frontend` et `apps/backend`
à chaque étape ; comportement vérifié manuellement dans l'UI et via
`twitter-text` en script autonome (les calculs de `weightedLength` et
`validRangeEnd` ont été vérifiés caractère par caractère sur les textes
réels utilisés pendant les tests).

## 2. Bug séparé (non corrigé) : échec d'envoi des posts avec média sur X

**Découvert pendant les tests du point 1, sans rapport avec le comptage de
caractères.**

**Symptôme** : un post texte seul part correctement vers X. Un post avec une
image jointe échoue avec une erreur générique côté UI
(`"post is too long, please fix it"` était trompeur dans un cas antérieur,
mais l'échec réel observé est différent : erreur `Unknown Error` /
`bad_body`, non-retryable, dans le workflow Temporal `postWorkflowV106`).

**Cause racine** (confirmée en lisant le message d'erreur complet stocké en
base, colonne `Post.error`, décodé depuis le payload Temporal) :

```
connect ECONNREFUSED 127.0.0.1:4007
  ... readOrFetch (libraries/helpers/src/utils/read.or.fetch.ts:7)
  ... x.provider.ts:680 (uploadMediaEntries)
```

Avec `STORAGE_PROVIDER=local` (notre config), Postiz stocke le chemin de
chaque média uploadé comme l'URL **publique complète** :
`${FRONTEND_URL}/uploads/...` (voir
`libraries/nestjs-libraries/src/upload/local.storage.ts:73` et `:106`), soit
`http://localhost:4007/uploads/...` chez nous.

Cette même valeur `path` est réutilisée par le worker orchestrateur pour
**retélécharger** le fichier en interne avant de le renvoyer à l'API X
(`x.provider.ts` → `readOrFetch(m.path)`, avec `m.path` qui commence par
`http`, donc branche HTTP de `readOrFetch`, jamais la lecture disque directe).

Le problème : à l'intérieur du conteneur `postiz`, rien n'écoute sur le port
4007. Ce port n'existe que côté **hôte** (`docker-compose.yaml` : mapping
`127.0.0.1:4007:5000`) ; en interne, nginx écoute sur le port 5000
uniquement. Le conteneur ne peut donc pas se re-contacter lui-même via
`localhost:4007`, alors même que le fichier est physiquement présent sur son
propre disque (`/uploads/...`).

### Pourquoi ce n'est *pas* corrigé par une simple config

Options envisagées, toutes écartées :

- **Changer `FRONTEND_URL`** vers une valeur joignable en interne (ex.
  `http://localhost:5000`, port réellement écouté par nginx dans le
  conteneur) : `FRONTEND_URL` est utilisé dans **plus de 40 fichiers** du
  code — callbacks OAuth de *tous* les réseaux sociaux (X, LinkedIn, Google,
  GitHub, etc.), CORS, emails, Stripe... Le changer casserait les callbacks
  OAuth déjà enregistrés côté X/LinkedIn/etc. avec l'URL exacte
  `http://localhost:4007/...`. Écarté : trop invasif pour viser un seul bug
  d'upload d'image.
- **Faire écouter nginx aussi sur le port 4007 en interne**, ou activer
  `network_mode: host` : nécessite de modifier la config nginx embarquée
  dans l'image (`var/docker/nginx.conf`) ou de changer fondamentalement la
  topologie réseau du conteneur — sort du périmètre "config uniquement"
  demandé, et présente son propre lot de risques (conflits de port avec
  d'autres services de la machine hôte en mode `host`).
- **Vraie correction propre** (nécessite de toucher le code Postiz, donc
  explicitement mise de côté pour l'instant à la demande de l'utilisateur) :
  faire lire le fichier directement depuis le disque local
  (`UPLOAD_DIRECTORY`) plutôt que de repasser par HTTP quand
  `STORAGE_PROVIDER=local`, ou introduire une URL de base distincte pour les
  usages internes vs. le rendu navigateur.

### Le seul contournement 100% config qui fonctionne

Basculer `STORAGE_PROVIDER` de `local` vers `cloudflare` (Cloudflare R2) dans
`docker-compose.yaml` (section déjà présente, commentée, lignes ~25-32).
Avec ce provider, le chemin stocké est une URL R2 externe
(`https://<bucket>.r2.cloudflarestorage.com/...`,
voir `libraries/nestjs-libraries/src/upload/cloudflare.storage.ts:42`),
réellement joignable aussi bien depuis le navigateur que depuis l'intérieur
du conteneur (les deux sortent sur Internet) — le problème de
"localhost qui ne veut rien dire en interne" disparaît structurellement.

**Prérequis non réunis actuellement** : un bucket Cloudflare R2 existant et
ses identifiants (`CLOUDFLARE_ACCOUNT_ID`, `CLOUDFLARE_ACCESS_KEY`,
`CLOUDFLARE_SECRET_ACCESS_KEY`, `CLOUDFLARE_BUCKETNAME`,
`CLOUDFLARE_BUCKET_URL`). Le code ne supporte que Cloudflare R2 comme
provider distant (le client S3 est câblé en dur sur l'endpoint
`https://<accountID>.r2.cloudflarestorage.com`, donc pas de S3
générique/MinIO auto-hébergé possible sans modifier le code).

**Impact réel en attendant** : uniquement les posts X **avec média joint**
sont affectés. Les posts texte seul fonctionnent normalement (validé
pendant cette session).
