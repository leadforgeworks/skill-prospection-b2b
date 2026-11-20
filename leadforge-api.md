# Mode boosté : brancher ce skill sur LeadForge

Le skill fonctionne seul. Branché sur LeadForge (https://leadforge.works), il travaille sur des données réelles au lieu de modèles à remplir : entreprises du registre officiel, décideurs identifiés, emails professionnels vérifiés, suivi de chaque prospect jusqu'au devis.

## Mise en place (une fois, 2 minutes)
1. Créer un compte sur https://leadforge.works (crédits offerts à l'inscription).
2. Dans l'application LeadForge, rubrique **Compte**, copier la **Clé API**.
3. La rendre disponible à Claude Code dans une variable d'environnement :
   - macOS / Linux : ajouter `export LEADFORGE_API_KEY="la-clé"` dans `~/.zshrc` ou `~/.bashrc`, puis rouvrir le terminal.
   - Windows (PowerShell) : `setx LEADFORGE_API_KEY "la-clé"`, puis rouvrir le terminal.

## Règles pour Claude
- Vérifier la présence de la clé SANS jamais l'afficher : bash `[ -n "$LEADFORGE_API_KEY" ] && echo cle-presente || echo cle-absente` ; PowerShell `if ($env:LEADFORGE_API_KEY) { 'cle-presente' } else { 'cle-absente' }`.
- Ne jamais écrire la clé dans un fichier, un message, un commit ou une sortie.
- Avant tout appel qui consomme des crédits, annoncer le coût maximum et demander l'accord de l'utilisateur.
- Commencer par vérifier le solde (gratuit).
- Ne jamais inventer un email ou un résultat : seules les données renvoyées par l'API font foi. Une adresse non vérifiée n'est jamais présentée comme vérifiée.
- En cas de réponse 402 : la fonction demande un abonnement LeadForge ; le dire simplement et continuer en mode autonome. En cas de 429 : attendre une minute, ne pas insister en boucle.

## Base
- URL : `https://leadforge.works`
- En-tête d'authentification : `x-access-token: $LEADFORGE_API_KEY`
- Contenu : JSON (`Content-Type: application/json`)

## Fonctions disponibles
| Besoin | Appel | Coût | Accès |
|---|---|---|---|
| Solde de crédits | `POST /v1/enrich/all` avec `{"data":[{}]}` | gratuit | tous les comptes |
| Trouver et vérifier les emails d'une liste | `POST /v1/enrich/all` avec `{"data":[{"first_name":"…","last_name":"…","company":"…","website":"…"}],"siren":true}` → `request_id`, puis `GET /v1/enrich/all/<request_id>` jusqu'à `success: true` | 1 crédit par email vérifié trouvé, rien sinon | tous les comptes |
| Ajouter des contacts au suivi commercial | `POST /v1/contacts` avec `{"data":[{"first_name":"…","last_name":"…","email":"…","company":"…"}]}` | gratuit | tous les comptes |
| Cible issue de l'analyse « Cibler » (segments, décideurs, mots-clés, problème, bénéfice) | `GET /v1/targeting` | gratuit | abonnés |
| Dirigeants d'une entreprise (registre public) et contacts déjà suivis | `GET /v1/company?siren=…` ou `?name=…` ou `?domain=…` | gratuit | abonnés |
| Fiche et signaux d'un contact (contacté, a visité le site, a cliqué, a répondu, rendez-vous) | `GET /v1/contacts/lookup?email=…` ou `?linkedin=<url>` ou `?name=…&company=…` | gratuit | abonnés |
| Enregistrer une étape dans le suivi | `POST /v1/contacts/event` avec `{"email":"…","type":"email_sent|call|linkedin|reply|meeting|won|lost|note","text":"…"}` | gratuit | abonnés |
| Rédiger un message privé personnalisé (premier contact ou relance 1 à 3) | `POST /v1/write` avec `{"net":"linkedin","first_name":"…","company":"…","headline":"…","step":0}` | 1 crédit par message | abonnés |

## Exemples
bash :
```bash
curl -s -X POST https://leadforge.works/v1/enrich/all \
  -H "x-access-token: $LEADFORGE_API_KEY" -H "Content-Type: application/json" \
  -d '{"data":[{"first_name":"Camille","last_name":"Martin","company":"Atelier Morel"}],"siren":true}'
```
PowerShell :
```powershell
$h = @{ 'x-access-token' = $env:LEADFORGE_API_KEY }
$b = @{ data = @(@{ first_name = 'Camille'; last_name = 'Martin'; company = 'Atelier Morel' }); siren = $true } | ConvertTo-Json -Depth 5
Invoke-RestMethod -Method Post -Uri https://leadforge.works/v1/enrich/all -Headers $h -ContentType 'application/json' -Body $b
```
Résultat d'un enrichissement : pour chaque contact, `email[0].email` et `email[0].qualification` (`nominative@pro` ou `deliverable@pro` = vérifié ; `catch_all@pro` = probable ; autre = non vérifié), plus les données d'entreprise (SIREN, secteur, effectif, site, téléphone) quand elles existent.
