# Étape 1 : Créer le dépôt privé sur GitHub

# Étape 2 : Initialiser votre projet en local et lier GitHub

## 	1. Initialiser le dépôt Git local
    git init

## 	2. Ajouter vos fichiers au suivi de Git
    git add .

## 	3. Créer votre premier commit
    git commit -m "Initial commit"

## 	4. Renommer la branche principale en "main" (standard GitHub)
    git branch -M main

## 	5. Connecter votre dossier local au dépôt GitHub privé
## 	(Remplacez l'URL par celle copiée à l'étape 1)
    git remote add origin https://github.com


# Étape 3 : Faire le Push et utiliser le Token
    git push -u origin main

