Challenge Information
---------------------
Challenge type: Forensics ... kinda?
Rating: Easy    2-3 hours

Installing Matrix-Synapse
-------------------------
We've previously recovered a PostgreSQL dump and a file archive.

Looking into the database we can conclude it's a [matrix-synapse](https://github.com/matrix-org/synapse) dump.

We'll need to install it and import the database.

I didn't really take notes of this, I mainly followed the [installation instructions](https://github.com/matrix-org/synapse/blob/master/INSTALL.md).

Then configured it to use psql according to [this documentation](https://github.com/matrix-org/synapse/blob/master/docs/postgres.md).

I didn't want to crack the users passwords because it's bcrypt with a work factor of 10 and I know better than trying to run even the simplest of dictionnaries on my antique machine, so I just replaced them all with a hash I generated:
```
postgres=# UPDATE users SET password_hash = '$2a$10$3TIGn2nHyXMo.oIgm9YPreFlVor2pz0tPGdX/Xsj0OcmMT0GQJfmm' WHERE 1=1;
```

Then we're ready to start!

Starting the investigation
--------------------------
Let's just log as anyone on the server:
[./]

TripleChaCha and Gwrizienn seem to have talked together. Let's investigate.

```
En bref, ils se sont mis à utiliser Kazh-Boneg à fond, ce qui fait que je me suis rapproché d'eux, en faisant attention bien sûr.
quand ils ont commencé à me demander de participer activement à la préparation d'un attentat, j'ai refusé et ils n'ont pas aimé.
donc au cas où je ne donne pas signe de vie dans les prochains jours, voici quelques informations qui pourront t'aider à accéder à mon compte pour dénoncer les agissements du Sivi-Ha-Kerez :
Voici le backup de mes clés Megolm, protégé par mot de passe.

Je ne vais pas te donner le mot de passe comme ça, mais vais te dire comment il est construit.
Initialement, j'utilisais des mots de passe comme SSTIC{3e43df4fc2e11c9226bbc2a22bc12a4d083678e6a2f3e9ca2fae05e19ed42ba7}, mais comme tout le monde s'est mis à faire la même chose, j'ai changé de méthode.
Le mot de passe est donné par un logiciel que j'ai écrit, qui prend en entrée un mot de passe qui sert de « clé de dérivation ».
En fait, je me suis inspiré de ton pseudo pour construire les clés que j'utilise : il s'agit de la concaténation de trois mots parmis un ensemble de mots écrits sur une feuille de papier que je garde sur moi tout le temps.
ça me permet d'éviter de devoir utiliser un coffre-fort de mot de passe là où ce n'est pas pratique)
Par exemple, sur le site de la météo de la Biérique qui indique combien de fois il fera beau demain, ma clé est AlloBruineCrachin et le mot de passe donné par le logiciel est Jbeh3AWZIve1Tez1Ptw+qg.
Je vais t'envoyer la feuille par courrier. Sans elle, je ne pourrai plus accéder à mon compte même si le Sivi-Ha-Kerez me torture.
Pour fonctionner, le logiciel a besoin d'un fichier de configuration. Voici le logiciel et sa configuration :
Decrypt show_password (326.39 KB)
Decrypt hashes.txt (167 B)
TripleChacha
Merci de ta confiance :)
Salut, tu es là ? J'ai reçu une feuille et je voudrais savoir si c'est la tienne ?
photo.jpg
Decrypt photo.jpg (257.27 KB)
```

The flag SSTIC{3e43df4fc2e11c9226bbc2a22bc12a4d083678e6a2f3e9ca2fae05e19ed42ba7} validates step2 and confirms we're on the right track!
