# Task005: DVCS mit GitHub

### Neues Repository anlegen
Ich nutze für diese Aufgabe den Code aus der vorherigen Aufgabe zum Clean Code. Daher wird für dieses gemeinesame Projekt ein neues [Repository auf Github](https://github.com/nerdfactor/bht-bowling-game) erstellt

![new-repo](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task005-DVCS_mit_GitHub/new-repo.png?raw=true)

und danach lokal ein git Repository initialisiert und dann miteinander verbunden.

![remote-repo](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task005-DVCS_mit_GitHub/remote-repo.png?raw=true)

Danach kann das maven Projekt angelegt werden und die ersten Inhalte werden direkt in git commited und [auf github gepusht](https://github.com/nerdfactor/bht-bowling-game/commit/346b4e1832944966f04f93a71419ac8b8b019f6b).

![first-commit](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task005-DVCS_mit_GitHub/first-commit.png?raw=true)

Um nochmal zu sehen, was git tatsächlich gemacht hat, zeigt das lokale log alle Änderungen.

![git-log](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task005-DVCS_mit_GitHub/git-log.png?raw=true)


### Zurück in die Vergangenheit
Während der Entwicklung kommt es natürlich immer wieder zu Momenten, bei denen man Änderungen rückgängig machen möchte. Beim Bowling Game habe ich mich irgendwann entschieden den Frame nicht mehr als separates Objekt abzubilden. Die dafür benötigte Datei konnte [daher natürlich aus git wieder entfernt werden](https://github.com/nerdfactor/bht-bowling-game/commit/17ed5ab981f5fd3e676ffa4b1604f7d90bd73c90).

![remove-frame](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task005-DVCS_mit_GitHub/remove-frame.png?raw=true)


Ein anderer Weg wurde notwendig, als ich Änderungen an den Dateien gemacht habe, die sich als unsinnig heraus gestellt hatten. Die Unit Tests funktionierten nicht mehr und ich wusste einfach nicht mehr was ich alles geändert hatte.

Zum Glück kann git status anzeigen welche Dateien sich geändert haben und diese Änderungen können dann mit einem git reset und einem neuen git checkout in den vorherigen Stand zurück versetzt werden. 

![git-reset](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task005-DVCS_mit_GitHub/git-reset.png?raw=true)

Und schließlich habe ich (natürlich überhaupt nur zum zeigen) einen falschen commit gemacht und sogar schon nach GitHub gepusht. Die Versionsnummer war fälschlich schon auf 1.2.0 hoch gezählt worden.

Dank git diff war es unproblematisch zu sehen welche Änderung im letzten commit war und wo die Versionsnummer falsch angegeben war. Über git log war es dann möglich die passende ID des commits zu ermitteln und über git revert zum vorherigen commit zurück zu gehen.

![git-diff](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task005-DVCS_mit_GitHub/git-diff.png?raw=true)

![git-revert](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task005-DVCS_mit_GitHub/git-revert.png?raw=true)

Die Änderung wurde auch durch einen [zusätzlichen commit transparent aufgezeichnet](https://github.com/nerdfactor/bht-bowling-game/commit/a6ebb0747b881c82a682ec6142f6965b832014f6).


### Arbeit an verschiedenen Computern
Um die unterschiedliche Handhabung zu testen, bin ich während der Entwicklung zwischen meinem Linux und Windows Computer gewechselt. Da an beiden git problemlos funktioniert, war der Austausch des Codes problemlos.

Unter Windows gibt es auch tolle GUI Tools, um git einfacher verwenden zu können (ja gibt es natürlich unter Linux auch :grin:).

![gui-1](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task005-DVCS_mit_GitHub/gui-1.png?raw=true)

![gui-2](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task005-DVCS_mit_GitHub/gui-2.png?raw=true)

Wenn man zeitgleich an mehreren Computern arbeitet, kann es natürlich einfach zu Konflikten kommen. In diesem Fall hatte ich zum [Testen am Frame noch eine Änderung vorgenommen](https://github.com/nerdfactor/bht-bowling-game/commit/cd27cc94bc13706722de5b5d319441ed779dbff0), den ich eigentlich im Rahmen eines Refactorings [bereits entfernt hatte](https://github.com/nerdfactor/bht-bowling-game/commit/17ed5ab981f5fd3e676ffa4b1604f7d90bd73c90).

![gui-3](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task005-DVCS_mit_GitHub/gui-3.png?raw=true)

In meinem IDE gibt es sogar auch direkt ein UI zum commiten in git.

![git-conflict](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task005-DVCS_mit_GitHub/git-conflict.png?raw=true)

Beim Versuch das dann nach GitHub zu pushen, wurde der Konflikt klar und konnte in diesem Fall zum Glück behoben werden, indem es verworfen wird und [wieder gemerged wird](https://github.com/nerdfactor/bht-bowling-game/commit/9ccc5bbe3dc62cde6a7e209ff31c40b446d52c5f).

![git-merge](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task005-DVCS_mit_GitHub/git-merge.png?raw=true)

### Merging Branches
Nach dem [Abschluss der vorherigen Aufgabe](https://github.com/nerdfactor/bht-bowling-game/commit/5b746d8bd3010687864601823c41fe972b144c9f), habe ich noch ein paar kleine Änderungen vornehmen wollen. Damit ich dabei nicht mit der Abgabefrist in Konflikt komme, habe ich das lieber in einem separaten Branch gemacht.

![git-branch1](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task005-DVCS_mit_GitHub/git-branch1.png?raw=true)

Nach einer [kleineren Änderung](https://github.com/nerdfactor/bht-bowling-game/commit/8473b66c048c6e94cb36f2535c2a3cfb5cd85f95) konnten diese dann wieder in den main Branch zurück gemerged werden.

![git-branch2](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task005-DVCS_mit_GitHub/git-branch2.png?raw=true)

Wenn man das nicht selber machen kann, weil der main Branch von jemand anderen verwaltet wird, kann man z.B. mit einem Pull Request dazu auffordern, den commit aus dem eigenen Branch zu pullen.

So konnte ich den oben beschriebenen "Fehler" beim Ändern der Versionsnummer auf einem gesonderten Branch machen und die Korrektur dann [wieder zurück mergen lassen](https://github.com/nerdfactor/bht-bowling-game/commit/b289c52fb341c82b2d215f01fe5104f256de5c09).

![request-1](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task005-DVCS_mit_GitHub/request-1.png?raw=true)

![request-2](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task005-DVCS_mit_GitHub/request-1.png?raw=true)







