# Contribuer

Deux regles, pas plus.

## 1. Signer ses commits (`Signed-off-by`)

Chaque commit doit porter la ligne :

```
Signed-off-by: Prenom Nom <adresse@example.com>
```

`git commit -s` l'ajoute pour vous.

C'est le [Developer Certificate of Origin](https://developercertificate.org/) :
vous attestez avoir le droit de soumettre ce code sous la licence du depot.
Sans cette ligne une contribution ne peut pas etre integree -- le depot ne
pourrait plus etre distribue sous d'autres termes sans retrouver chaque auteur
un par un.

## 2. Le chart doit rendre, dans les deux sens

`helm lint .` doit passer, et le rendu doit etre verifie **avec et sans** les
valeurs optionnelles -- un bloc conditionnel qui ne rend rien quand il devrait,
ou l'inverse, ne se voit pas autrement :

```bash
helm lint .
helm template t . | grep -c egress-ca                 # attendu : 0
helm template t . --set egress.proxy=http://p:3128 \
                  --set-string egress.proxyCA=... | grep egress-ca
```

## Ce qui rend une pull request facile a integrer

- **Un sujet par pull request.** Un correctif et un changement de valeurs par
  defaut dans le meme diff se relisent deux fois plus mal.
- **Dire ce qui a ete verifie**, et comment. « ca marche chez moi » n'est pas
  une verification, un rendu compare en est une.
- **Toute valeur nouvelle est documentee dans `values.yaml`**, la ou elle est
  declaree, et sa valeur par defaut ne suppose rien de votre infrastructure.
