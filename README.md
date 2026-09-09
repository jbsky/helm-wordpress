# WordPress Hardened -- chart Helm

Chart Helm qui fait tourner WordPress sur Kubernetes avec des images durcies
**FROM scratch** : aucun shell, aucun gestionnaire de paquets, non-root.

| Conteneur | Image | Role |
|---|---|---|
| `nginx` | [`jbsky/nginx-waf-hardened`](https://github.com/jbsky/nginx-hardened) | sert les fichiers statiques, proxifie le PHP, ModSecurity + OWASP CRS |
| `php-fpm` | [`jbsky/php-fpm-hardened`](https://github.com/jbsky/php-fpm-hardened) | execute WordPress |
| `nginx-exporter` | `nginx/nginx-prometheus-exporter` | metriques nginx |
| `phpfpm-exporter` | `hipages/php-fpm_exporter` | metriques du pool FPM |
| `redis` *(optionnel)* | `redis:alpine` | cache d'objets persistant de WordPress |

Un **initContainer** Alpine prepare le volume : il telecharge la derniere
version stable de WordPress et la depose dans un `emptyDir` partage. C'est ce
qui permet a l'image php-fpm de rester sans shell -- elle n'a jamais a copier
un fichier.

## Ce que le chart ne fait pas

- **Pas de base de donnees.** WordPress attend un MySQL/MariaDB joignable ; les
  parametres de connexion arrivent avec la configuration du site (voir
  ci-dessous).
- **Pas d'ingress.** Le `Service` est en `ClusterIP` ; le routage, la
  terminaison TLS et le WAF d'entree vivent en amont.
- **Pas de version de WordPress figee.** L'initContainer prend la derniere
  stable a chaque demarrage du pod. C'est un choix : pas d'image a reconstruire
  a chaque sortie, au prix de la reproductibilite.

## Installation

Le chart ne porte aucune configuration de site. Celle-ci est deposee dans le
repertoire du chart avant l'installation, sous trois chemins que les templates
ramassent :

```
php/*            -> ConfigMap (php.ini, www.conf, wp-config.php...)
nginx/*          -> ConfigMap (nginx.conf, vhost, regles ModSecurity)
files/mu-plugins/*.php  -> ConfigMap, montes en mu-plugins
```

```bash
cp -a mon-site/php mon-site/nginx mon-site/files .
helm upgrade mon-site . --install \
  --namespace mon-site --create-namespace \
  --values mon-site/values.yaml
```

## Valeurs

Les images, les ressources et les contextes de securite se reglent par
conteneur (`nginx`, `phpfpm`, `nginxexporter`, `phpfpmexporter`). Trois blocs
meritent une mention :

### `egress` -- sortie internet de l'initContainer

Par defaut : acces direct, aucun proxy.

```yaml
egress:
  proxy: "http://proxy.example:3128"
  proxyCA: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
```

`proxyCA` n'est necessaire que si le proxy **intercepte le TLS** : il presente
alors sa propre autorite, et sans elle le premier appel HTTPS echoue en
`self-signed certificate in certificate chain` -- `apk` compris, donc avant
meme que `curl` ne soit installe.

### `redis` -- cache d'objets

```yaml
redis:
  enabled: true
```

Desactive par defaut. Quand il est actif, l'initContainer installe le plugin
`redis-cache` s'il manque et **resynchronise le drop-in `object-cache.php` a
chaque demarrage** : c'est la copie que WordPress charge, elle ne doit jamais
diverger de la version du plugin reellement installee. Les constantes
`WP_REDIS_*` restent a votre charge, dans la configuration du site.

### `wordfence` -- protection etendue

```yaml
wordfence:
  extendedProtection: true
```

Ecrit `wordfence-waf.php` puis `.user.ini` dans le volume, dans cet ordre. Un
`auto_prepend_file` qui pointe sur un fichier absent n'est pas un avertissement
mais une **erreur fatale** : toutes les pages en 500. A n'activer que si
l'extension est reellement installee.

## Contexte de securite

Les deux conteneurs applicatifs tournent sous le meme UID non-root (1999 par
defaut) -- c'est ce qui rend le volume partage lisible des deux cotes -- avec
`allowPrivilegeEscalation: false` et, pour nginx, toutes les capabilities
retirees.

## Licence

Ce chart est sous **Apache-2.0**, voir [`LICENSE`](LICENSE).
Copyright 2026 jbsky.

Les images qu'il deploie ont chacune la leur, dans leur propre depot ; les
logiciels qu'elles embarquent gardent celle de leur amont.

Contributions : voir [`CONTRIBUTING.md`](CONTRIBUTING.md), un `Signed-off-by`
est demande.
