# Configuration Supabase — Projet dédié aux Contrats INFOTELCOM

Ce système est **totalement indépendant** du Supabase déjà utilisé pour le site vitrine et les certificats. Tu vas créer un nouveau projet, à part, avec ses propres identifiants.

## Étape 1 — Créer le nouveau projet Supabase

1. Va sur **[supabase.com](https://supabase.com)** et connecte-toi (avec `contact.infotelcom@gmail.com` ou un autre compte)
2. Clique **New project**
3. Choisis une organisation, donne un nom clair, ex. `infotelcom-contrats`
4. Choisis un mot de passe de base de données solide (note-le en lieu sûr, ce n'est pas la clé API)
5. Choisis une région proche (Europe si tu n'es pas sûr)
6. Clique **Create new project** — patiente 1 à 2 minutes que le projet s'initialise

## Étape 2 — Créer les tables

Une fois le projet prêt : menu de gauche → **SQL Editor** → **New query**, colle ce script en entier, puis **Run**.

```sql
-- ═══════════════════════════════════════════
-- TABLE DES CONTRATS SIGNÉS
-- ═══════════════════════════════════════════
create table if not exists public.contrats_signes (
  id uuid primary key default gen_random_uuid(),
  created_at timestamptz default now(),
  numero_contrat text unique not null,

  -- Identité du signataire
  nom text not null,
  prenom text not null,
  date_naissance date,
  adresse text,
  email text not null,
  telephone text not null,

  -- Détails du poste
  poste text not null,
  type_contrat text not null,        -- CDI / CDD / Stage / Freelance
  date_debut date not null,
  duree_periode_essai text,

  -- Acceptation
  accepte_contrat boolean default false,
  accepte_reglement boolean default false,
  accepte_confidentialite boolean default false,

  -- Signature
  signature_data text not null,      -- image PNG en base64
  ip_signature text,
  user_agent text,

  statut text default 'signé'
);

alter table public.contrats_signes enable row level security;

-- N'importe qui peut ENREGISTRER un contrat signé (le formulaire public)
create policy "Le public peut signer un contrat"
on public.contrats_signes for insert
to anon
with check (true);

-- Personne ne peut lire sans être admin (voir plus bas)
create policy "Seuls les admins peuvent consulter"
on public.contrats_signes for select
to authenticated
using (exists (select 1 from public.admins where admins.id = auth.uid()));

-- ═══════════════════════════════════════════
-- TABLE DES ADMINISTRATEURS AUTORISÉS
-- ═══════════════════════════════════════════
create table if not exists public.admins (
  id uuid primary key references auth.users(id) on delete cascade,
  email text unique not null,
  nom text,
  created_at timestamptz default now()
);

alter table public.admins enable row level security;

create policy "Un admin peut voir la liste des admins"
on public.admins for select
to authenticated
using (exists (select 1 from public.admins a where a.id = auth.uid()));
```

## Étape 3 — Récupérer l'URL et la clé de CE nouveau projet

1. Menu de gauche → **Project Settings** (icône ⚙️) → **API**
2. Copie :
   - **Project URL** → ressemble à `https://xxxxxxxxxxxxx.supabase.co`
   - **anon / public key** (dans "Project API keys") → longue chaîne commençant par `eyJ...` ou `sb_publishable_...`

## Étape 4 — Coller ces identifiants dans les fichiers

Ouvre `index.html` ET `admin-contrats.html`, et remplace dans chacun :

```javascript
const SUPABASE_URL = 'COLLE-ICI-LURL-DE-TON-NOUVEAU-PROJET';
const SUPABASE_KEY = 'COLLE-ICI-LA-CLE-ANON-DE-TON-NOUVEAU-PROJET';
```

par tes vraies valeurs copiées à l'étape 3. **Il faut faire le remplacement dans les deux fichiers**, avec les mêmes valeurs dans les deux.

## Étape 5 — Créer ton compte admin

1. Dans ce même nouveau projet Supabase → **Authentication** → **Users** → **Add user** → **Create new user**
2. Renseigne un email et un mot de passe fort
3. Coche **Auto Confirm User**
4. Clique **Create user**
5. Copie l'**UID** généré (visible dans la liste des utilisateurs)

## Étape 6 — Autoriser ce compte comme admin

Retourne dans **SQL Editor**, lance (en remplaçant les valeurs) :

```sql
insert into public.admins (id, email, nom)
values ('COLLE-L-UID-ICI', 'contact.infotelcom@gmail.com', 'Administrateur INFOTELCOM');
```

Répète les étapes 5 et 6 pour chaque personne devant avoir accès au tableau de bord (ex. Albert, Christ, Juvel).

## Ce qui est maintenant séparé

| | Site principal INFOTELCOM | Système de contrats |
|---|---|---|
| Projet Supabase | `ncssencaoacjbpakshrd.supabase.co` (existant) | Nouveau projet créé ici |
| Tables | `certificats` | `contrats_signes`, `admins` |
| Hébergement | `infotelcom-congo-brazza.netlify.app` | Nouveau site Netlify séparé |

Aucune donnée, aucune clé, aucune table n'est partagée entre les deux systèmes. Une panne, une erreur de config ou une fuite sur l'un n'affecte jamais l'autre.

## Sécurité — ce qu'il faut savoir

- La clé `anon` publique dans le code JS est **normale et sans danger** : c'est la RLS (Row Level Security) ci-dessus qui protège réellement les données, pas le secret de la clé.
- Un visiteur ne peut qu'**insérer** un contrat (signer), jamais lire les contrats des autres.
- Seuls les comptes présents dans la table `admins` peuvent lire `contrats_signes` — même avec un compte Supabase valide, sans entrée dans `admins`, l'accès est refusé.
- Le mot de passe de chaque admin doit rester privé — c'est lui qui protège l'accès au dashboard.
