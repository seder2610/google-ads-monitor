# Google Ads · Alertes budget → Slack

> Vision démo vs prod (tous les workflows) : [`../VISION-CLIENT-COMPLETE.md`](../VISION-CLIENT-COMPLETE.md)  
> **IA** : aucune (démo et prod) — règles + seuils uniquement.

Workflow n8n **prêt à livrer** pour agences SEA : surveille les budgets Google Ads en temps réel et publie les alertes dans un canal **Slack** — là où l’équipe media travaille déjà.

**Deux fichiers :**
| Fichier | Usage |
|---------|--------|
| `google-ads-budget-monitor.json` | **Démo** — données fictives, zéro API (Loom, premier call) |
| `google-ads-budget-monitor-v2.json` | **Production** — Google Ads API v19 + Slack |

---

## Ce que le client obtient

- Vérification automatique **4×/jour** (8h, 12h, 16h, 20h — lun–ven, Paris)
- Données **live** du jour (pas d’export Sheets manuel)
- Alerte Slack dès qu’une campagne dépasse le seuil (80 % par défaut)
- Recommandation par campagne (critique / attention / surveillance)
- Alerte Slack aussi en cas d’**erreur API** (token, OAuth, ID compte)

---

## Architecture production (9 nodes)

```
⏰ Planification
    ↓
⚙️ Configuration client   ← seul node à personnaliser
    ↓
📡 Google Ads — dépenses du jour
    ↓
🔄 Normaliser données
    ↓
🔍 Détecter dépassements
    ↓
🚨 Envoyer une alerte ?
    ├── Oui → 📝 Composer message Slack → 💬 Publier sur Slack
    └── Non → ✅ RAS — rien à signaler
```

Un **post-it** dans le workflow rappelle la checklist de livraison.

---

## Hébergement recommandé

| Option | Pour qui |
|--------|----------|
| **n8n Cloud** (compte client) | Défaut — simple, toujours allumé |
| **VPS client** (Docker) | Agence avec IT |
| Local | Démo uniquement — pas en prod |

Le workflow et les credentials restent **chez le client**.

---

## Installation (≈ 45 min, 1ère fois)

### 1. Google Ads API

1. [Google Cloud Console](https://console.cloud.google.com) → projet → activer **Google Ads API**
2. Credentials **OAuth2** (application web) → URI redirect : `https://[votre-n8n]/rest/oauth2-callback`
3. Google Ads Manager → **Outils → API Center** → copier le **Developer Token**
4. Noter le **Customer ID** (sans tirets) et le **MCC ID** si applicable

Dans n8n : **Credentials → Google Ads OAuth2 API** → Connect.

### 2. Slack Bot

1. [api.slack.com/apps](https://api.slack.com/apps) → **Create New App**
2. **OAuth & Permissions** → scope `chat:write`
3. **Install to Workspace** → copier le token `xoxb-...`
4. n8n : **Credentials → Slack API**
5. Dans Slack : `/invite @votre-bot` dans le canal (ex. `#ads-monitoring`)

> Si le canal ne reçoit rien : utiliser l’**ID canal** (`C0123...`) au lieu de `#nom` dans la config.

### 3. Importer le workflow

1. n8n → **Import** → `google-ads-budget-monitor-v2.json`
2. Node **⚙️ Configuration client** :

| Champ | Exemple |
|-------|---------|
| `nom_agence` | Élysée Digital |
| `seuil_alerte_pct` | `80` |
| `slack_channel` | `#ads-monitoring` |
| `google_ads_customer_id` | `1234567890` |
| `google_ads_login_customer_id` | `1234567890` (MCC ou = customer) |
| `developer_token` | token API Center |

3. Nodes **📡 Google Ads** et **💬 Slack** → sélectionner les credentials
4. **Test workflow** → vérifier message Slack
5. **Activate**

---

## Démo rapide (v1 mock)

1. Importer `google-ads-budget-monitor.json`
2. Configurer Telegram uniquement (v1) — idéal pour Loom sans workspace client
3. Test → 3 campagnes fictives en alerte

---

## Exemple d’alerte Slack

```
🚨 ALERTE BUDGET GOOGLE ADS
📅 lundi 19 mai · 12:00 (Paris)
🏢 Élysée Digital · Seuil 80%
────────────────────

🔴 CRITIQUE · Remarketing — Sephora
Client : Sephora · Compte 4405
Budget 600 € · Dépensé 545 € (91%) · Reste 55 €
→ 🔴 Action : pauser ou réduire les enchères de 20 % immédiatement.

⚡ 2 campagne(s) au-dessus du seuil
Workflow P-MON · données du jour (API Google Ads)
```

---

## Requête GAQL

```sql
SELECT campaign.name, campaign.id, campaign_budget.amount_micros,
       metrics.cost_micros, customer.descriptive_name
FROM campaign
WHERE campaign.status = 'ENABLED'
  AND segments.date DURING TODAY
  AND campaign_budget.amount_micros > 0
ORDER BY metrics.cost_micros DESC
```

---

## Personnalisation

| Paramètre | Où | Défaut |
|-----------|-----|--------|
| Seuil % | Configuration client | 80 |
| Horaires | Planification (cron) | 8h, 12h, 16h, 20h lun–ven |
| Canal Slack | Configuration client | `#ads-monitoring` |

---

## Stack

- n8n · Google Ads API v19 · Slack Bot (`chat:write`)

---

## Roadmap

- [ ] Multi-comptes (boucle customer_id)
- [ ] Rapport hebdo PDF / email
- [ ] Historique alertes → Google Sheets
- [ ] Option email en plus de Slack

---

## Licence & support

Workflow fourni dans le cadre d’un **Sprint automation** — maintenance et évolutions via contrat **Partner** mensuel.

Repository : [github.com/seder2610/google-ads-monitor](https://github.com/seder2610/google-ads-monitor)
