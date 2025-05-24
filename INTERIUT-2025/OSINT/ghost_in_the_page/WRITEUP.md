# Ghost in the Page

Pour ce challenge on doit d'abord retrouver l’identité d'un de l'ancien admin sys de l'entreprise.

Toujours avec le useragent castem quand on se rend sur https://norecastem.fr/organisation.php on tombe sur un document intéressant:

![image](orga.png)

On remarque ici qu'une personne a été tuée, ce qui correspond aux informations données dans le challenge. On en déduit donc que la personne que l'on recherche est cette dernière.

Si on se rappelle bien nous avons déja vu ces noms sur la page principale du site de l'entreprise:

![image](equipe.png)

On peut donc s'imaginer qu'à un moment, la personne que nous cherchons était présente sur cette page du site. Nous allons donc vérifier s'il existe d'anciennes versions du site qui ont été archivées sur Waybackmachine: 

![image](wayback.png)

bingo, on retrouve le nom de l'ancien administrateur système.
`Henri Fargelisse`

 Comme l’indique la description du challenge, nous devons maintenant chercher sur les réseaux ce que Henri aurait pu dire. Après quelques recherches sur différents réseaux, nous tombons sur son compte Instagram, qui contient une story étrange : 

![image](insta.png)

 Voici maintenant la partie difficile de ce challenge : il faut revenir sur ses pas pour ceux qui n’ont pas été assez méticuleux. 
 
 Si l’on avait prêté attention aux détails jusqu’à présent, on aurait remarqué que le compte Pastebin ayant publié le paste trouvé sur `/guide.txt` contient également un autre paste nommé `admin_final_note`, protégé par un mot de passe.

Le nom du compte Pastebin en question est Hf-castem, une référence à Henri Fargelisse.

On utilise le mot de passe donné par Henri pour déverrouiller le paste, qui contient les informations suivantes, ainsi que la première partie du flag :

```
Si tu lis ce message… c’est que je n’existe plus.

Ils savent. Je ne sais pas comment, mais ils savent que je ne suis pas des leurs.  
J’ai voulu trop creuser. Trop vite. J’ai vu ce que j’aurais dû ignorer.

J’ai collecté ce que j’ai pu. Pas assez pour tout dévoiler, mais assez pour que quelqu’un poursuive.  

Le Spectre… j’ai trouvé des informations sur lui. Des enregistrements dans une de ses sessions qu’il a oublié d’effacer.

Je n’ai pas pu tout transférer, mais j’ai glissé des éléments dans ma boîte pro avant qu’ils ne me coupent.  
C’est discret. Une réponse automatique sur mon adresse pro. 

Bonne chance.
- H.F.

1st part:
interiut{retrouve_le_tu_es_la
```

On décide d’écouter Henri et d’envoyer un mail à son adresse professionnelle, que l’on peut retrouver grâce à la Wayback Machine. En réponse, on reçoit la deuxième partie du flag qui nous permet d'avoir le flag complet: 

```
interiut{retrouve_le_tu_es_la_cle_pour_faire_tomber_le_spectre}
```


