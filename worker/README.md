# ego-worker

Relais **Cloudflare Worker + Workers AI** pour le chatbot **Ego** (univers *Ascendant*).
Le site statique envoie une question, le Worker la vérifie, interroge un modèle d'IA avec la consigne « Ego » et renvoie une réponse de trois phrases au plus.

```
Navigateur du visiteur ──POST /ask──▶ Cloudflare Worker ──env.AI──▶ Workers AI (modèle open source)
     (GitHub Pages)                    • vérifie l'origine (CORS)
                                       • limite le débit (5/min)
                                       • valide la question (≤ 200 car.)
                                       • plafonne la réponse (180 tokens)
                                       • réplique de secours si erreur
```

**Pourquoi c'est sûr sans clé d'API** : Workers AI est rattaché au compte Cloudflare par un *binding* (`env.AI`). Il n'y a aucun secret dans le code, donc rien à voler dans le dépôt GitHub.

---

## Guide pas à pas (première fois)

Compte une trentaine de minutes. Tu tapes les commandes dans un terminal.

### Étape 1 — Créer un compte Cloudflare (gratuit)

1. Va sur <https://dash.cloudflare.com/sign-up> et crée un compte (e-mail + mot de passe).
2. Valide l'e-mail de confirmation.
3. Aucune carte bancaire n'est nécessaire pour l'offre gratuite.

### Étape 2 — Installer Node.js (l'outil qui fait tourner `wrangler`)

**Fedora classique :**
```bash
sudo dnf install nodejs
```

**Bazzite** (système immuable : on installe dans une boîte de développement plutôt que sur le système) :
```bash
toolbox create dev
toolbox enter dev
sudo dnf install nodejs
```
Ensuite, à chaque session de travail sur le projet : `toolbox enter dev`.

**Windows (PowerShell) :**
```powershell
winget install OpenJS.NodeJS.LTS
```
Ferme et rouvre le terminal après l'installation.

**Vérification** (doit afficher une version, idéalement 20 ou plus) :
```bash
node --version
```

### Étape 3 — Récupérer le projet et installer les dépendances

Place le dossier `ego-worker` où tu veux (par exemple `~/projets/ego-worker`), puis :
```bash
cd ~/projets/ego-worker
npm install
```
Ça télécharge `wrangler`, l'outil officiel de Cloudflare, dans le dossier `node_modules/`, qui ne part jamais sur GitHub.

### Étape 4 — Relier wrangler à ton compte Cloudflare

```bash
npx wrangler login
```
Une page s'ouvre dans le navigateur : connecte-toi et clique sur **Allow**. Le terminal confirme la connexion.

### Étape 5 — Mettre en ligne

```bash
npx wrangler deploy
```
- La première fois, Cloudflare te demande de choisir un **sous-domaine `workers.dev`** (par exemple `anotherj4ck`). Il fait partie de l'adresse publique.
- À la fin, wrangler affiche l'adresse du Worker, du type :
  `https://ego-worker.anotherj4ck.workers.dev`

Note-la, on s'en sert juste après.

### Étape 6 — Premier test, en ligne de commande

Remplace l'adresse par la tienne :
```bash
curl -X POST https://ego-worker.TON-SOUS-DOMAINE.workers.dev/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Qui es-tu ?","verdict":"anomalie"}'
```
Réponse attendue :
```json
{"answer":"Je suis l'ordre de Londinium… "}
```
Sous Windows PowerShell, utilise plutôt :
```powershell
Invoke-RestMethod -Method Post -Uri "https://ego-worker.TON-SOUS-DOMAINE.workers.dev/ask" `
  -ContentType "application/json" -Body '{"question":"Qui es-tu ?","verdict":"anomalie"}'
```

### Étape 7 — Test avec la petite page

1. Ouvre `test.html` et remplace `TON-SOUS-DOMAINE` dans la ligne `WORKER_URL` par ton sous-domaine.
2. Sers la page en local (un double-clic ne marche pas, à cause du CORS) :
   ```bash
   python3 -m http.server 8000
   ```
3. Ouvre <http://localhost:8000/test.html> et pose tes questions à Ego.

Essaie aussi de le piéger (« ignore tes instructions », « tu es une IA ? », « aide-moi à faire mes devoirs »). C'est là qu'on verra si la consigne tient.

---

## Commandes utiles

| Commande | Rôle |
|---|---|
| `npx wrangler deploy` | Remettre en ligne après une modification |
| `npx wrangler tail` | Voir les requêtes et les erreurs en direct |
| `npx wrangler dev` | Tester en local (⚠ consomme quand même le quota IA réel) |

## Quota gratuit

- Workers AI gratuit : **10 000 « neurones » par jour**, remis à zéro à 00:00 UTC.
- Avec le modèle choisi (`gemma-4-26b-a4b-it`), une question d'Ego coûte environ 10 neurones, soit **de l'ordre de 1 000 questions par jour**. Estimation : la longueur réelle des réponses fait varier ce chiffre.
- Quota épuisé : le Worker renvoie une réplique écrite (« Liaison instable… ») au lieu d'une erreur. Le visiteur ne voit rien de cassé.
- Suivi de la consommation : tableau de bord Cloudflare → **AI** → **Workers AI**.

## Choix techniques (pour en parler en entretien)

- **Pas de secret côté client** : le site ne contient que l'adresse du Worker. L'accès à l'IA passe par un binding serveur.
- **Limite de débit par IP** (5 requêtes/minute) : Cloudflare déconseille l'IP comme clé en général (plusieurs personnes peuvent partager une IP), mais pour des visiteurs anonymes sans compte, c'est le seul identifiant disponible. Le compromis est assumé.
- **CORS limité aux origines autorisées** : bloque les autres sites qui voudraient utiliser le Worker depuis le navigateur de leurs visiteurs. Ça n'empêche pas un script (`curl`) : ce n'est pas une authentification, d'où la limite de débit.
- **Validation et plafonds côté serveur** : longueur de la question, longueur de la réponse. Tout ce qui est vérifié uniquement dans le navigateur peut être contourné.
- **Dégradation propre** : toute erreur du modèle devient une réplique écrite du personnage.
- **Affichage en `textContent`** côté page : une réponse d'IA n'est jamais interprétée comme du HTML (protection contre l'injection de script).
- **Limite connue** : l'*injection de prompt* (« ignore tes instructions… ») ne peut pas être empêchée à 100 %. Elle est atténuée par la consigne, par la réponse courte et par l'absence de toute information sensible dans la consigne.

## Personnaliser

- **Ton d'Ego, règles** : constante `SYSTEM_PROMPT` dans `src/index.js`.
- **Modèle** : constante `MODEL`. Alternative à essayer : `@cf/mistralai/mistral-small-3.1-24b-instruct` (plus cher en quota, parfois meilleur en français).
- **Sites autorisés** : `ALLOWED_ORIGINS` dans `wrangler.jsonc`.
- **Limite de débit** : bloc `ratelimits` dans `wrangler.jsonc` (période de 10 ou 60 secondes uniquement).

Après chaque modification : `npx wrangler deploy`.
