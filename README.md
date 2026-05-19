# P-MON · Google Ads Budget Monitor

Workflow n8n qui surveille automatiquement les budgets Google Ads **en temps réel** et envoie une alerte Telegram dès qu'une campagne dépasse un seuil configurable — avec une recommandation IA par campagne.

**Deux versions disponibles :**
- `google-ads-budget-monitor.json` — **v1 Mock** : données fictives, zéro configuration API, idéal pour la démo Loom
- `google-ads-budget-monitor-v2.json` — **v2 API Live** : connecté à la vraie Google Ads API v19, prêt pour la production

---

## Ce que ça fait

Toutes les 4 heures (8h / 12h / 16h / 20h, Paris, lun-ven), le workflow :

1. Interroge l'API Google Ads v19 pour récupérer les dépenses du jour
2. Parse la réponse (conversion micros → euros)
3. Pour chaque campagne au-dessus du seuil (80% par défaut) :
   - Calcule le burn rate horaire
   - Estime l'heure d'épuisement du budget
   - Génère une recommandation concrète
4. Envoie une alerte Telegram formatée

### Exemple d'alerte reçue

```
🚨 ALERTE BUDGET GOOGLE ADS
📅 lundi 19 mai à 12:00
🏢 Agence SEA Demo  ·  Seuil : 80%
─────────────────────

🔴 Remarketing — Sephora — Sephora
Budget : 600€  ·  Dépensé : 545€  ·  91% utilisé  ·  Reste : 55€
Compte #4405
💡 Budget épuisé estimé à 14h30. Action : réduire enchères -20% ou pauser.
─────────────────────

🟠 Brand — Décathlon — Décathlon
Budget : 800€  ·  Dépensé : 712€  ·  89% utilisé  ·  Reste : 88€
Compte #4403
💡 Budget épuisé estimé à 16h15. Action : réduire enchères -20% ou pauser.
─────────────────────

⚡ 2 campagne(s) à risque
Vérifiez vos budgets avant épuisement total.
```

---

## Architecture — v2 (API Live)

```
⏰ Scheduler (8h/12h/16h/20h, lun-ven)
    ↓
⚙️ Config (seuil, chat_id, nom agence, customer_id, MCC, dev token)
    ↓
📡 Google Ads API — Budgets du jour (GAQL v19, OAuth2)
    ↓
🔄 Parser réponse API (micros → euros, camelCase → structure interne)
    ↓
🔍 Analyser seuils + calcul burn rate + recommandation IA
    ↓
🚨 Des alertes ?
    ├── Oui → 📝 Formater message → 📣 Telegram
    └── Non → ✅ Log silencieux
```

**9 nodes. Données live Google Ads sans export manuel.**

---

## Installation v2 — API Live

### Prérequis

- Accès Google Ads avec droits **admin** sur le compte client
- (Si MCC) Accès au compte Manager
- n8n opérationnel (self-hosted ou cloud)

---

### Étape 1 — Créer les credentials Google Ads dans n8n

**1a. Créer un projet GCP et activer l'API**

1. Aller sur [console.cloud.google.com](https://console.cloud.google.com)
2. Créer un projet (ou en sélectionner un existant)
3. Activer **Google Ads API** dans "API et services"
4. Créer des credentials OAuth2 → Application web
5. Ajouter l'URI de redirection n8n : `https://[votre-n8n]/rest/oauth2-callback`
6. Télécharger le JSON → noter `client_id` et `client_secret`

**1b. Obtenir le Developer Token**

1. Aller dans Google Ads Manager → **Outils → API Center**
2. Copier le **Developer Token** (format : `xxxxxxxxxxxxxxxxxxxxxxxx`)
3. Note : si le compte est en accès test, le token est en mode test (suffisant pour commencer)

**1c. Trouver le Customer ID et MCC ID**

- **Customer ID** : Google Ads → coin supérieur droit → `123-456-7890` → **enlever les tirets** → `1234567890`
- **MCC ID** : idem, depuis le compte Manager. Si compte direct (pas de MCC) → mettre le même ID que customer_id

**1d. Créer la credential dans n8n**

1. n8n → **Credentials → New → Google Ads OAuth2 API**
2. Renseigner `client_id` et `client_secret`
3. Cliquer **Connect** → autoriser l'accès Google
4. Sauvegarder → noter l'ID de la credential

---

### Étape 2 — Créer le bot Telegram

1. Ouvrir Telegram → chercher **@BotFather**
2. Envoyer `/newbot` → donner un nom puis un username
3. Copier le **token** (`123456789:ABCDef...`)
4. Envoyer un message à votre bot, puis ouvrir :
   ```
   https://api.telegram.org/bot<VOTRE_TOKEN>/getUpdates
   ```
5. Trouver `"chat":{"id": XXXXXXXXX}` → c'est votre **chat_id**
6. Dans n8n → **Credentials → New → Telegram API** → coller le token → sauvegarder

---

### Étape 3 — Importer et configurer le workflow

1. Dans n8n : **Workflows → Import from file** → sélectionner `google-ads-budget-monitor-v2.json`

2. Dans le node **⚙️ Config**, renseigner les 6 paramètres :

| Paramètre | Valeur |
|---|---|
| `seuil_alerte_pct` | `80` (modifier selon besoin) |
| `telegram_chat_id` | Votre chat_id Telegram |
| `nom_agence` | Nom de votre agence ou client |
| `google_ads_customer_id` | Customer ID sans tirets |
| `google_ads_login_customer_id` | MCC ID (ou même valeur que customer_id) |
| `developer_token` | Token Google Ads API Center |

3. Dans le node **📡 Google Ads API** :
   - Credentials → sélectionner la credential Google Ads OAuth2 créée à l'étape 1d

4. Dans le node **📣 Alerte Telegram** :
   - Credentials → sélectionner la credential Telegram créée à l'étape 2

---

### Étape 4 — Tester

1. Ouvrir le workflow → **Test workflow**
2. Vérifier que le node API retourne des campagnes
3. Vérifier que l'alerte Telegram arrive
4. Si tout OK → **Activate**

---

## Installation v1 — Démo Mock (5 minutes, zéro API)

Pour une démo rapide sans aucune configuration Google Ads :

1. Importer `google-ads-budget-monitor.json`
2. Créer le bot Telegram (étapes 2 ci-dessus)
3. Dans **⚙️ Config** : renseigner uniquement `telegram_chat_id` et `nom_agence`
4. Dans **📣 Alerte Telegram** : sélectionner la credential Telegram
5. **Test workflow** → l'alerte arrive avec 3 campagnes fictives en alerte (Sephora 91%, Décathlon 89%, Leroy Merlin 86%)

---

## Requête GAQL utilisée (v2)

```sql
SELECT
  campaign.name,
  campaign.id,
  campaign_budget.amount_micros,
  metrics.cost_micros,
  customer.descriptive_name
FROM campaign
WHERE campaign.status = 'ENABLED'
  AND segments.date DURING TODAY
ORDER BY metrics.cost_micros DESC
```

Les montants sont en **micros** (1€ = 1 000 000 micros) — la conversion est faite automatiquement dans le node Parser.

---

## Personnalisation

| Paramètre | Node | Valeur par défaut | Description |
|---|---|---|---|
| Seuil alerte | ⚙️ Config | `80` | % de budget consommé pour déclencher l'alerte |
| Fréquence | ⏰ Scheduler | `0 8,12,16,20 * * 1-5` | 4x/jour, lun-ven |
| Emoji seuils | 📝 Format | 95%=🔴, 85%=🟠, 80%=🟡 | Modifiable dans le node Format |

---

## Stack

- **n8n** (self-hosted ou cloud)
- **Google Ads API v19** (OAuth2 + Developer Token)
- **Telegram Bot API** (gratuit, illimité)

---

## Roadmap

- [ ] Multi-comptes : boucle sur plusieurs customer_id
- [ ] Rapport PDF hebdomadaire (n8n → Google Docs → PDF)
- [ ] Slack + Email en parallèle de Telegram
- [ ] Dashboard Notion mis à jour en temps réel
- [ ] Historique des alertes dans Google Sheets pour analyse tendances
# google-ads-monitor
