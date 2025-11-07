+++
date = '2025-11-07T12:04:07+01:00'
draft = false
title = 'Ma baguette, mon béret et mon container'
+++

Tout pentester ou amoureux de CTF qui se respecte a sûrement déjà lancé fiérement sa machine Kali, en pure adoration des outils présents dessus.  
Mais au bout d'un moment, la distro ne tourne plus, les outils ne sont pas bien configurés ou installés, et le côté cybersécurité laisse place à de la configuration d'OS.

Et aussi, c'est relativement énervant de ne pas pouvoir lancer une game rocket league entre deux challenges Hack The Box.

---

Et c'est là qu'Exegol entre en jeu. Exegol est un environnement pré-configuré permettant de conteneuriser ses pentests.  
Simple, rapide, plus performant qu'une VM, basé sur des images docker. Tout est mis en place pour nous faire gagner du temps.
C'est communautaire, français, et absolument génial.  

### Trucs cools sur Exegol:

- Apps graphiques configurées de base (Firefox, Bloodhound, Burp Suite...), rien besoin de configurer. Et oui, apps graphiques, car le X11 est partagé !
- Crédentials déjà définis pour neo4j, bettercap, empire...  
- Autocomplétion efficace des commandes (le gain de temps est faramineux).  
- La docs exegol est très complète, pour du bonus allez regarder [https://docs.exegol.com/tips-and-tricks](https://docs.exegol.com/tips-and-tricks) qui rassemble plein de shortcuts rajoutés  

--- 

Pour ceux fans des détails techniques: [https://docs.exegol.com/wrapper/features](https://docs.exegol.com/wrapper/features).  
Et pour ceux qui aiment des belles config exegol, vous pouvez retrouver les miennes ici : [https://github.com/m0nkeydbus/exegol-config](https://github.com/m0nkeydbus/exegol-config).