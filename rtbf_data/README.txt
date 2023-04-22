Content

1. DOSSIER crawling-output

1.a. FICHIERS: 'school'_courses_'year'.json

contient les informations liées aux cours/unités d'enseignement de l'école 'school' pour l'année 'year'

champs:
- id: identifiant du cours
- name: nom du cours
- year: année académique
- languages: langues (code iso)
- teachers: professeurs
- url: lien vers la page web du cours
- content: texte décrivant le contenu du cours
- goal: texte décrivant les objectifs du cours
- activity: texte décrivant les activités du cours
- other: autre texte intéressant pour le scoring

1.b. FICHIERS: 'school'_programs_'year'.json

contient les informations liées aux formations de l'école 'school' pour l'année 'year'

champs:
- id: identifiant de la formation
- name: nom de la formation
- cycle: cycle (bachelier, master, ...)
- faculties: facultés organisant la formation
- campuses: campus où se donne la formation
- url: lien vers la page web du programme
- courses: noms de cours/unité d'enseignement
- ects: nombre de crédits pour chaque cours/unité d'enseignement

Note: il se peut que certains champs se retrouvent dans la partie cours et inversement. C'est typiquement le cas pour les 'ects'

Note 2: il y a aussi des fichiers finissant par '_pre' qui ne sont pas nécessaires pour l'analyse de données


2. DOSSIER scoring-output

2.a. FICHIERS: 'school'_courses_scoring_'year'.csv

Contient le scoring des cours pour chaque 'thématique' (climat, environnement, ...) ET si le cours est dédié

2.b FICHIERS: 'school'_matches_'year'.json

Contient pour chaque cours qui a matché, la liste des patterns qui ont été trouvé dans sa description

2.c FICHIERS 'school'_programs_scoring_'year'.csv

Contient pour chaque 'thématique' le nombre de cours dans le programme qui ont matché, le nombre de cours dédié dans le programme, et le total du nombre de cours qui ont matché dans au moins une 'thématique'

3. DOSSIER web_data

Contient les données qui sont affichées sur le site. Les fichiers de ce dossier sont construits à partir des fichiers des deux premiers dossiers.

3.a. FICHIERS: 'school'_data_'year'_courses.json

Contient les données des cours qui ont matché avec au moins une thématique

champs:
- id: identifiant du cours
- name: nom du cours
- year: année académique
- languages: langues (code iso)
- teachers: professeurs
- ects
- url: lien vers la page web du cours
- cycles: les cycles dans lesquels le cours est donné
- fields: les champs d'études (dérivés des facultés via le fichier faculties_to_fields.csv) dans lequel le cours est donné
- themes: les thématiques sur lesquelles le cours a matché
- dedicated: s'il le cours est dédié ou non



3.b. FICHIERS: 'school'_data_'year'_programs.json

Contient les données des formations contennant au moins un cours qui a matché

champs:
- id: identifiant de la formation
- name: nom de la formation
- cycle: cycle (bachelier, master, ...)
- url: lien vers la page web du programme
- languages: langues dans laquelle la formation est donnée
- campuses: campus où se donne la formation
- faculties: facultés organisant la formation
- fields: les champs d'études (dérivés des facultés via le fichier faculties_to_fields.csv) dans lequel la formation est donnée
- themes: 'thématiques' couvertes par les cours du programme 
- themes_scores: donne pour chaque thématique, le nombre de cours qui la couvre
- matched_courses: list des identifiants des cours du programme qui ont matché
- nb_dedicated_courses: nombre de cours dédié du programme


4. FICHIER faculties_to_fields.csv

Donne pour chaque faculté, le ou les champs d'études correspondant

5. FICHIER scoring_fields.csv

Indique pour chaque école, quels champs parmi 'content', 'goal', 'activit' et 'other' sont utilisés pour faire le scoring 



