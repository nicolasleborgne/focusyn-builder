# Le constructeur

**Ce dossier n'est pas une partie de Focusyn : c'est le contenu d'un second
dépôt.** Il vit ici pour être relu avec le reste, et se pousse ailleurs.

## Pourquoi un dépôt séparé

SLSA niveau 3 demande que le matériel qui signe la provenance soit hors de
portée des étapes de construction définies par le projet. Sur GitHub, cela
signifie exactement une chose : **la construction doit se faire dans un workflow
réutilisable venu d'un autre dépôt**.

Un workflow réutilisable local (`uses: ./.github/workflows/…`) ne suffit pas. Il
vient du même commit que ce qu'il construit : celui qui écrit la release écrit
le constructeur, il n'y a pas deux parties mais une seule. C'est du niveau 2
avec une indirection en plus.

Ici, le dépôt du produit ne fournit que des paramètres. Il ne peut insérer
aucune étape dans la tâche qui signe, donc il ne peut pas atteindre le jeton
OIDC. La provenance porte l'identité de ce constructeur (`job_workflow_ref`), et
c'est elle que l'on vérifie :

```bash
gh attestation verify oci://ghcr.io/nicolasleborgne/focusyn:v1.0.0 \
  --repo nicolasleborgne/focusyn \
  --signer-repo nicolasleborgne/focusyn-builder \
  --predicate-type https://slsa.dev/provenance/v1
```

`--signer-repo` est l'assertion qui compte. Sans elle, on vérifie qu'une
signature existe ; avec elle, on vérifie qu'elle vient d'où l'on croit.

## Ce que cela ne donne pas

Tant que la même personne possède les deux dépôts, **l'isolation est technique,
pas organisationnelle**. Elle protège d'une dépendance compromise, d'une action
tierce compromise, d'un Dockerfile hostile, d'une étape ajoutée par erreur dans
la release. Elle ne protège pas de son propriétaire.

Ce qui la rendrait organisationnelle : une protection distincte sur ce dépôt
(revue par quelqu'un d'autre), ou un constructeur partagé entre plusieurs
produits. Terraform pose déjà la première moitié — les règles de ce dépôt sont
les mêmes que celles du produit.

## Installation

Le dépôt est créé par `terraform/github/builder.tf`. Reste à y pousser ce
dossier :

Surtout **pas** `git init` dans `builder/` : ce dossier est suivi par le dépôt
Focusyn, et un dépôt imbriqué le transformerait en sous-module à la première
mise à jour. On clone ailleurs, on copie, on pousse.

```bash
git clone git@github.com:nicolasleborgne/focusyn-builder.git /tmp/focusyn-builder
cp -a builder/. /tmp/focusyn-builder/
cd /tmp/focusyn-builder
git add .
git commit -S -m "Le constructeur : image, SBOM et provenance"
git push -u origin main
```

Puis épingler l'appel côté Focusyn, **par empreinte** — un workflow réutilisable
référencé par branche se déplace sous les pieds de ce qu'il construit :

```bash
git rev-parse HEAD   # l'empreinte à reporter dans .github/workflows/release.yaml
```

## Le faire évoluer

Toute modification de ce dépôt change ce que signifie une attestation. Deux
règles :

1. **Épingler par empreinte** côté appelant, jamais par branche. Une version
   déjà publiée doit rester vérifiable avec ce qui l'a construite.
2. **Étiqueter** (`builder-v1`, `builder-v2`) pour s'y retrouver : l'empreinte
   dit *quoi*, l'étiquette dit *quand*. Les tags `builder-v*` sont protégés par
   le même Terraform.
