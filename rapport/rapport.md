# HPC Projet 1: Optimisation d'un projet open source

## Description du software et Motivations

Le software que j'ai choisit pour ce projet est un interpréteur de [Brainfuck](https://fr.wikipedia.org/wiki/Brainfuck) écrit en C.Vous pouvez trouver ici le repository du projet initial:https://github.com/fabianishere/brainfuck. J'ai choisi un interpréteur car c'est un projet open source assez courant et de fait bien documenté, et de plus l'optimisation d'un interpréteur peut se faire de manière générale selon deux méthodes: optimiser l’interpréteur en lui même ou bien se concentrer sur l'optimisation du code Brainfuck.J'ai également choisit de travailler sur du brainfuck plutôt qu'un autre langage car déjà il s'agit d'un langage exotique et très amusant mais surtout car il contient très peut d'instruction tout en permettant une grande complexité et donc facile a prendre en main et la lecture et la compréhension du code de l’interpréteur n'a pas prit trop de temps en comparaison a d'autre langages plus compliqués.Et pour finir j'ai également choisit ce projet car le repo github était bien présenté,le code documenté et de manière générale on constate qu'il s'agit d'un projet exécuté sérieusement et dans le but d’être utilisable par tout le monde et non pas juste ses créateurs.

## Installation
Le projet initial peut être trouvé [ici](https://github.com/fabianishere/brainfuck) et mon repository contenant mon optimisation peut être accéder depuis [ici](https://github.com/kaerdhalis/brainfuck.git).L'installation du software est simple et est décrite dans le Readme du repository.J'appuie cependant sur le fait que le projet utilise la librairie **libedit-dev** et qu'il est nécessaire de l'installer pour que cela fonctionne correctement.

## Benchmarking

Pour le benchmarking je me suis principalement servit de l'outil **perf** pour effectuer mes mesures et chercher une optimisation. J’ai donc en premier exécuté la commande ```stat``` pour avoir une mesure générale du programme(pour mon premier benchmark j'ai utilisé le programme hanoi-opt.bf,en effet ne parlant pas couramment le brainfuck j'ai préféré utiliser un exemple fournit et dont la taille était assez conséquente pour avoirs des donnés pertinentes).
```
sudo perf stat -e cpu-clock,branch-instructions,mem-stores,branch-misses,bus-cycles,cache-misses,cache-references,cpu-cycles,instructions,page-faults ./brainfuck ../examples/hanoi-opt.bf
```
 et on obtient les résultats suivant:

 ```
 Performance counter stats for './brainfuck ../examples/hanoi-opt.bf':
               Written by Clifford Wolf <http://www.clifford.at/bfcpu/>
             873.57 msec cpu-clock                 #    0.992 CPUs utilized          
      1’775’489’519      branch-instructions       # 2032.456 M/sec                    (74.45%)
        434’936’243      mem-stores                #  497.885 M/sec                    (74.96%)
          3’948’943      branch-misses             #    0.22% of all branches          (75.46%)
         20’846’484      bus-cycles                #   23.864 M/sec                    (75.43%)
          1’384’995      cache-misses              #   16.826 % of all cache refs      (50.08%)
          8’231’431      cache-references          #    9.423 M/sec                    (49.61%)
      2’703’461’765      cpu-cycles                #    3.095 GHz                      (61.86%)
      8’938’232’421      instructions              #    3.31  insn per cycle           (74.55%)
                215      page-faults               #    0.246 K/sec                  
   -----------------------------------     -----------------------------------
        0.880631859 seconds time elapsed

        0.874199000 seconds user            
        0.000000000 seconds sys    
```
La mesure qui nous intéresse la est le nombre de **mem-stores** car si on étudie le code on peut voir que celui-ci charge le code BF complet en mémoire puis le parse en instructions stockés dans une liste chaînée avant de l’exécuter.Le choix de stocker toues les instructions avant de les exécuter plutôt que de les exécuter durant la lecture du programme est un choix intéressant et j'aurais voulu faire une comparaison entre les deux méthodes pour voir la plus efficaces mais de ce fait j'aurais du réécrire une grosse partie du code.Je me suis donc concentré sur un moyen d’accélérer la lecture des ficher BF lors de leur exécution.

Une deuxième analyse a ensuite était faite avec la commande suivante:
```
sudo perf record -e cpu-clock,branch-instructions,mem-stores,branch-misses,bus-cycles,cache-misses,cache-references,cpu-cycles,instructions,page-faults ./brainfuck ../examples/hanoi-opt.bf
```
En analysant les résultats on constate que la plupart des instruction se font dans la fonction *brainfuck_execute* qui se charge d’exécuter le code BF.On va donc chercher a optimiser l’exécution du code Brainfuck plutôt que l’interpréteur.

## Proposition d'optimisation
Après le benchmark précédent on a donc deux pistes d'optimisation,optimiser le chargement en mémoire du code BF et optimiser l’exécution du code.Optimiser le code peut se faire de plusieurs manières par exemple au lieu d'effectuer les instructions similaires al la suite comme les incrémentations ou les décrémentations de manière séquentielle sur la tape, on peut compter le nombre d’incrémentations puis augmenter la valeur de la case en une seule fois et donc réduire le nombre d’accès mémoire ie. dans le cas du code suivant ```+++++``` au lieu d’incrémenter 5 fois la case on ajoute juste 5 a la valeur de la case après avoir lu les instructions. cependant en regardant le code on peut voir que cet interpréteur effectue déjà plusieurs optimisation sur la lecture du code.En effet comme expliqué précédemment lors de la lecture les instructions sont stockés dans une liste chaînée et si plusieurs instructions similaires se suivent,il ne va stocker l'instruction qu'une seule fois mais avec une valeur égale au nombre d'instructions lus.De la même manière les boucles sont stockés sous formes de listes chaînées auxiliaires et l'instruction `[` contient la racine de cette boucle.
En cherchant donc une optimisation a effectuer sur le code j'ai constaté qu'une structure est très souvent utilisé en brainfuck, il s'agit de `[-]` qui va remettre la case pointé a 0.Cependant si la valeur de la case est élevé la boucle va mettre plus longtemps a s’exécuter car elle décrémente de 1 a chaque itération. Le but de cette optimisation va donc être de repérer dans le code ce type de boucle et au lieu de la parcourir simplement remettre la valeur de la case a 0.

## Implémentation

### Lecture de fichier
En regardant le code on voit deux fonctions très similaires,*parse_stream* et *parse_string*, qui font exactement la même chose mais l'une sur un stream et l'autre sur une string,mon optimisation est donc de stocke le fichier dans une string grâce a la methode *mmap(void *addr, size_t length, int prot, int flags,
                  int fd, off_t offset)* qui est plus efficace que la fonction *read()* et ensuite utiliser le parse_string ce qui permet en plus de réduire les redondances dans le code.

```c
int run_file(char *file) {

    int fd = open(file, O_RDONLY);
    int len = lseek(fd, 0, SEEK_END);
    char *data = (char*)mmap(0, len, PROT_READ, MAP_PRIVATE, fd, 0);
    if (data == NULL) {
        return EXIT_FAILURE;
    }

    return run_string(data);
}

```
### Optimisation des boucles de clear

Pour cette optimisation on va utiliser deux booléens ```loop``` et ```minus``` pour indiquer si l'on rentre dans une boucle et si l’instruction minus est la seule dans se boucle et dans le cas présent on remplace l'instruction de boucle par une remise a 0
```c
switch(c) {
case BRAINFUCK_TOKEN_PLUS:
case BRAINFUCK_TOKEN_MINUS:

    //check if token is a minus and is in a loop
    if(c==BRAINFUCK_TOKEN_MINUS){
        if(loop==true&& minus==false){
                  minus=true;
                  loop=false;
              }
              else if(minus==true){
                  minus=false;
              }}

  (*ptr)++;
  for (; *ptr < end && (temp_c = str[*ptr]) &&
      (temp_c == BRAINFUCK_TOKEN_PLUS || temp_c == BRAINFUCK_TOKEN_MINUS); (*ptr)++) {
      //if there is multiple tokens in the loop pass to false
      minus=false;
    if (temp_c == c) {
      instruction->difference++;
    } else {
      instruction->difference--;
    }
  }
  (*ptr)--;
  break;
case BRAINFUCK_TOKEN_NEXT:
case BRAINFUCK_TOKEN_PREVIOUS:
          if(minus==true){
              minus=false;
          }
  (*ptr)++;
  for (; *ptr < end && (temp_c = str[*ptr]) &&
      (temp_c == BRAINFUCK_TOKEN_NEXT || temp_c == BRAINFUCK_TOKEN_PREVIOUS); (*ptr)++) {
    if (temp_c == c) {
      instruction->difference++;
    } else {
      instruction->difference--;
    }
  }
  (*ptr)--;
  break;
case BRAINFUCK_TOKEN_OUTPUT:
case BRAINFUCK_TOKEN_INPUT:
          if(minus==true){
              minus=false;
          }
  (*ptr)++;
  for (; *ptr < end && (str[*ptr] == c); (*ptr)++) {
    instruction->difference++;
  }
  (*ptr)--;
  break;
case BRAINFUCK_TOKEN_LOOP_START:
          if(minus==true){
              minus=false;
          }
  (*ptr)++;
  loop=true;
  instruction->loop = brainfuck_parse_substring_incremental(str, ptr, end);
  break;
case BRAINFUCK_TOKEN_LOOP_END:

    //if minus is the only token of the loop we have a clear loop
    if(minus==true){
             root->type = BRAINFUCK_TOKEN_ZERO;
    }
    loop=false;
  return root;

```


## resultats

Pour effectuer le benchmark final j'ai ecrit le test **benchmark.bf** qui  est un code très simple qui incrémente une case de 200 puis repasse la case a 0 avec une loop et cela un très grand nombre de fois. En effet le brainfuck étant un langage simple il est difficile de trouver un programme ou la différence d'execution est significative.
Avec le programme initial:
```
sudo perf stat -e cpu-clock,branch-instructions,mem-stores,branch-misses,bus-cycles,cache-misses,cache-references,cpu-cycles,instructions,page-faults ./brainfuck ../examples/benchmark.bf

 Performance counter stats for './brainfuck ../examples/benchmark.bf':

              1.39 msec cpu-clock                 #    0.850 CPUs utilized          
         2’269’652      branch-instructions       # 1637.615 M/sec                  
         1’129’042      mem-stores                #  814.634 M/sec                  
            12’272      branch-misses             #    0.54% of all branches        
            32’612      bus-cycles                #   23.530 M/sec                  
     <not counted>      cache-misses                                                  (0.00%)
     <not counted>      cache-references                                              (0.00%)
     <not counted>      cpu-cycles                                                    (0.00%)
     <not counted>      instructions                                                  (0.00%)
               100      page-faults               #    0.072 M/sec                  

       0.001630675 seconds time elapsed

       0.001675000 seconds user
       0.000000000 seconds sys
```

et ensuite avec notre code optimisé:
```
sudo perf stat -e cpu-clock,branch-instructions,mem-stores,branch-misses,bus-cycles,cache-misses,cache-references,cpu-cycles,instructions,page-faults ./brainfuck ../examples/benchmark.bf

 Performance counter stats for './brainfuck ../examples/benchmark.bf':

              0.79 msec cpu-clock                 #    0.785 CPUs utilized          
           199’841      branch-instructions       #  251.444 M/sec                    (4.57%)
           509’156      mem-stores                #  640.630 M/sec                  
            11’251      branch-misses             #    5.63% of all branches        
            18’920      bus-cycles                #   23.806 M/sec                  
            28’122      cache-misses              #    0.000 % of all cache refs      (95.43%)
     <not counted>      cache-references                                              (0.00%)
     <not counted>      cpu-cycles                                                    (0.00%)
     <not counted>      instructions                                                  (0.00%)
                98      page-faults               #    0.123 M/sec                  

       0.001012133 seconds time elapsed

       0.001036000 seconds user
       0.000000000 seconds sys


```

On peut donc voir que le programme s'effectue plus vite donc notre optimisation de code fonctionne et de plus le nombre d'entrée mémoire a diminuer par 2, ce qui peut être du au mappage du programme ainsi qu'a la réduction d'instruction effectué après la modification de la boucle de clear.

## Conclusion
Pour conclure le projet était très intéressant surtout le fait de se plonger dans un code que l'on n'a pas écrit et essayer de le rendre plus performant,cependant dans mon cas le code était déjà bien écrit et optimisé et il a donc été compliqué de trouver quoi rajouter sans avoir a modifier toute sa structure. De plus dans mon cas je trouve que mon optimisation rend le code moins flexible et moins extensible même s'il augmente ses performances et que les auteurs ont du faire ce choix de manière considérés et que les optimisations que j'aurais pu ajouté on déjà été reflechis de leur coté.
