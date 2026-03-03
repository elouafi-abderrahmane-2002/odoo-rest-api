# 🔌 Odoo REST API Module

Odoo expose ses données via XML-RPC et JSON-RPC par défaut — deux protocoles que personne
n'utilise vraiment pour des intégrations modernes. Ce module change ça : il transforme
n'importe quelle instance Odoo en une API REST standard, accessible via des endpoints HTTP
classiques avec un token de session.

J'ai travaillé dessus dans le cadre de mon stage chez NextDefense, où on avait besoin de
connecter Odoo à des systèmes tiers sans passer par le protocole RPC natif.

---

## Comment ça fonctionne

```
  Système externe (frontend, ERP tiers, script Python...)
          │
          │  POST /auth/
          │  { login, password, db }
          │
          ▼
  ┌─────────────────────────────────────┐
  │            Odoo Instance            │
  │                                     │
  │  ┌──────────────────────────────┐   │
  │  │      AuthController          │   │
  │  │  POST /auth/                 │   │
  │  │  → retourne un cookie        │   │
  │  │    de session authentifiée   │   │
  │  └──────────────┬───────────────┘   │
  │                 │ cookie session    │
  │  ┌──────────────▼───────────────┐   │
  │  │      REST Controllers        │   │
  │  │                              │   │
  │  │  GET  /api/{model}/          │   │
  │  │  GET  /api/{model}/{id}      │   │
  │  │  POST /api/{model}/          │   │
  │  │  PUT  /api/{model}/{id}      │   │
  │  │  DELETE /api/{model}/{id}    │   │
  │  └──────────────┬───────────────┘   │
  │                 │                   │
  │  ┌──────────────▼───────────────┐   │
  │  │      Odoo ORM Layer          │   │
  │  │   env['res.partner'].search  │   │
  │  │   env['product.product']...  │   │
  │  └──────────────┬───────────────┘   │
  │                 │                   │
  └─────────────────┼───────────────────┘
                    │
                    ▼
             PostgreSQL
```

---

## Ce que le module expose

Tous les modèles Odoo sont accessibles — pas besoin de configurer chaque endpoint manuellement.
Le contrôleur générique mappe les routes automatiquement vers l'ORM :

```
GET  /api/res.partner/              → env['res.partner'].search([])
GET  /api/res.partner/42            → env['res.partner'].browse(42)
GET  /api/product.product/?query={"domain":[["active","=",true]]}
POST /api/sale.order/               → env['sale.order'].create({...})
PUT  /api/sale.order/7              → order.write({...})
DELETE /api/res.partner/42          → partner.unlink()
```

---

## Installation

```bash
# 1. Copier le module dans le dossier addons Odoo
cp -r odoo-rest-api/ /path/to/odoo/addons/

# 2. Installer les dépendances
pip install -r requirements.txt

# 3. Mettre à jour la liste des modules dans Odoo
# Apps → Update Apps List → rechercher "rest_api" → Install
```

---

## Exemple d'utilisation (Python)

```python
import requests, json

# Étape 1 : Authentification
auth = requests.post('http://localhost:8069/auth/', json={
    'params': {
        'login': 'admin',
        'password': 'admin',
        'db': 'mydb'
    }
})
cookies = auth.cookies  # session cookie

# Étape 2 : Lire les partenaires
partners = requests.get(
    'http://localhost:8069/api/res.partner/',
    params={'query': json.dumps({'fields': ['name', 'email', 'phone']})},
    cookies=cookies
)
print(partners.json())

# Étape 3 : Créer un bon de commande
order = requests.post(
    'http://localhost:8069/api/sale.order/',
    json={'partner_id': 7, 'order_line': [...]},
    cookies=cookies
)
```

---

## Structure du module

```
odoo-rest-api/
├── __manifest__.py         ← déclaration du module Odoo
├── controllers/
│   └── controllers.py      ← routes REST génériques (tous modèles)
├── models/
│   └── models.py           ← extensions ORM si nécessaire
├── views/
│   └── views.xml           ← vues admin (config module)
└── requirements.txt
```

---

## Ce que j'ai appris

Le plus intéressant dans ce projet : comprendre comment Odoo gère ses sessions côté serveur.
Contrairement à une API stateless avec JWT, Odoo utilise des cookies de session stockés en base
(ou filesystem). Ça implique que chaque requête REST doit transporter ce cookie — ce qui
fonctionne bien pour des clients serveur-à-serveur, mais devient problématique pour des SPAs
qui ne gèrent pas les cookies nativement.

La vraie leçon : les abstractions d'Odoo (ORM, contrôleurs) sont puissantes mais il faut
comprendre les couches en dessous pour les utiliser correctement dans un contexte d'intégration.

---

*Projet réalisé dans le cadre de ma formation ingénieur — ENSET Mohammedia*
*Par **Abderrahmane Elouafi** · [LinkedIn](https://www.linkedin.com/in/abderrahmane-elouafi-43226736b/) · [Portfolio](https://my-first-porfolio-six.vercel.app/)*
