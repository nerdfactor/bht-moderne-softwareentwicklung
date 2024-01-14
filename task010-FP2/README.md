# Task010: Funktionale Programmierung 2

Für diese Aufgabe habe ich vorgesehen das bestehende Bowling Game aus der Aufgabe zu Clean Code in einer funktionalen Sprache neu zu schreiben. Die erste Problematik dabei bestand in der Wahl der Sprache. Ich habe unzählige Forenbeiträge gelesen um heraus zu finden welche Sprache sich besonders gut für Einsteiger eignen würde. Und ob die Empfehlung Haskell, Clojure oder das gerade super angesagt Ocaml war, habe ich nie Vorteile gehört die über "das gefällt mir besser" hinaus gingen. Den einzigen greifbaren Vorteil für mich habe ich bei Clojure auf Grund der Nähe zu Java und der JVM gesehen.

Meine endgültige Wahl viel dann auch auf Clojure, nachdem ich das Paper "Out of the tar pit" von Ben Mosley und Peter Marks gelesen hatte und sie es als Grundlage für die Entwicklung von Clojure beschrieben hatten. So schlecht kann Clojure dann ja nicht sein. Ich habe dann noch eine sehr aufschlussreiche Youtube Serie von Rich Hickey zu ["Clojure for Java Developers"](https://www.youtube.com/watch?v=P76Vbsk_3J0) gesehen und war mit meiner Entscheidung relativ zufrieden.

## Vorüberlegung für die Portierung
Die Kernfunktionalität des Bowling Game ist die Berechnung der Punkte einer Bowling Runde. Dafür bekommt es eine Liste von umgeworfenen Pins im ganzen Spiel. Diese werden dann nach verschiedenen Regeln addiert.

```java
public int countScore() {
	int currentScore = 0;
	int checkedRoll = 0;
	for (int frame = 0; frame < AMOUNT_OF_FRAMES; frame++) {
		if (isRollAStrike(checkedRoll)) {
			currentScore += countScoreForStrike(checkedRoll);
			checkedRoll++;
		} else if (isRollASpare(checkedRoll)) {
			currentScore += countScoreForSpare(checkedRoll);
			checkedRoll += 2;
		} else {
			currentScore += countScoreForOpenFrame(checkedRoll);
			checkedRoll += 2;
		}
	}
	return currentScore;
}
```

Dabei greifen die Methode auf internen State des BowlingGame Objekts zu, in dem die bereits umgeworfenen Pins und der aktuelle Wurf gespeichert sind. Wenn ich davon ausgehe, dass die gleiche Funktion in funktionaler Form keinen State haben und verändern sollte, müssten diese Informationen an die Funktion übergeben werden. Da zur Berechnung der Punktezahl die Liste der aktuell umgeworfenen Pins benötigt werden, muss diese immer übergeben und komplett durchgearbeitet werden.


```java
public int countScore(int[] knockedOverPinsPerRoll) {
	[...]
	return currentScore;
}
```

Diese Funktion hat dann alle notwendigen Daten und kann bei jedem Aufruf das gleiche Ergebnis zurück geben.

## Aufgbau der Umgebung
Die Einrichtung von Clojure ist auf [clojure.org](https://clojure.org/guides/install_clojure) zum Glück sehr verständlich beschrieben. Und die Unterstützung von Clojure in IntelliJ macht es super einfach ein einfaches Projekt zu starten.

Ein einfaches Hello World auszugeben ging daher recht schnell:

```clojure
(defn -main [& args]
  (println "Hello World!"))
```

Und von dort aus nur ein kurzer Sprung zu einer Anwendung, die schon mal eine Liste von Pins entgegen nimmt und diese einfach ausgibt:

```clojure
(ns bowling-game.core)
(require '[clojure.string :as str])

;; take a list of rolls and print them.
(defn calc-score [rolls]
  (doseq [roll rolls]
    (println roll)))


(defn -main [& args]
  ;; check if the arguments contain a string with rolls.
  (if (and (seq args) (string? (first args)) (not-empty (first args)))
    ;; take the first argument, split it by comma, and convert each element to integer.
    (let [rolls (map read-string (clojure.string/split (first args) #","))]
      ;; pass the rolls to calc-score function.
      (calc-score rolls))
    (println "Please provide all rolls of the game in a comma separated string.")))
```

Die Klammern machen mich allerdings schon an dieser Stelle ziemlich wahnsinnig.

## Berechnung des Punktestands
Wenn ich nun meiner Vorüberlegung folge, müsste die Funktion calc-score einfach nur die bestehende Liste von Pins iterieren und einen Score hoch zählen.

```clojure
;; take a list of rolls and return the score
(defn calc-score [rolls]
  ;; create a loop through the rolls that counts the score in each frame.
  (loop [remaining-rolls rolls score 0]
    ;; check if there are no more rolls or the last frame is reached.
    (if (empty? remaining-rolls)
      score
      ;; otherwise sum up the score of the current frame and call the loop again with the remaining rolls.
      (recur (rest remaining-rolls) (+ score (first remaining-rolls))))))
```

Dabei ruft sich die Schleife rekursiv wieder auf, und übergibt dabei die verbleibenden rolls und zählt den score anhand des ersten Wertes in den bestehenden rolls hoch. Jeder Aufruf der Schleife bekommt damit immer den komplett benötigten Bestand der Informationen, um die nächste Iteration zu berechnen.

Der Syntax mit recur finde ich dabei sehr gewöhnungsbedürftig.

So kurz vor dem Ergebnis bin ich dann erstmal gegen eine Wand gelaufen. Während dieser einfache Schleifenkörper mit einem simplen If verständlich war, wollte es mir nicht in den Kopf kommen, wie ich das If Else If Else Konstrukt aus der ursprünglichen Java Methode abbilden konnte. Erschwerend kam dazu, das mir hier IntelliJ mehr im Weg als eine Hilfe war. Immer wenn ich innerhalb des bestehenden Codes etwas ergänzen wollte, hat er wild an Stellen Klammern gesetzt bzw. mich daran gehindert Klammern zu setzen, so dass der Code am Ende nicht mehr funktionierte.

Bis ich dann verstanden habe, dass auf if in clojure einfach nur eine Funktion ist, die wiederum Funktionen für den positiven und negativen Fall übernimmt und die dann weiter ausführt. Dadurch konnte ich dann die verschiedenen Bedingungen einfach als in sich verschachtelten Aufruf mehrere If Funktionen schreiben:

```clojure
;; take a list of rolls and return the score
(defn calc-score [rolls]
  ;; create a loop through the rolls that counts the score in each frame (two rolls).
  (loop [remaining-rolls rolls score 0]
    ;; check if there are no more rolls.
    (if (empty? remaining-rolls)
      score
	  ;; for each frame check the score:
      ;; check if the current roll is a strike.
      (if (is-strike (first remaining-rolls))
        ;; if it is a strike, add the next two rolls to the score and move to the next frame.
        (recur (rest remaining-rolls) (+ score 10 (first (rest remaining-rolls)) (first (rest (rest remaining-rolls)))))
        ;; check if the current roll and the next roll is a spare.
        (if (is-spare (first remaining-rolls) (first (rest remaining-rolls)))
          ;; if it is a spare, add the next roll to the score and move to the next frame.
          (recur (rest (rest remaining-rolls)) (+ score 10 (first (rest (rest remaining-rolls)))))
          ;; if it is not a strike or a spare, add the two rolls to the score and move to the next frame.
          (recur (rest (rest remaining-rolls)) (+ score (first remaining-rolls) (first (rest remaining-rolls)))))))))
```

Die Definition der Hilfsfunktionen is-strike und is-spare machen den Code dabei etwas lesbarer. Der Klammerwahnsinn wird aber insgesamt nicht besser.

Mit dieser Erweiterung der count-score Funktion habe ich erfolgreich die Kernfunktion des Bowling Games neu implementiert und bekomme die gleiche Ausgabe:

```
lein run "1,2,3,4,5,5,10,10,1,2,3,4,5,5,10,10,1,2,3,4"

Total score of the game is:  138
```

## Abschlussbetrachtung
Der kurze Ausflug in die Funktionale Programmierung in dieser und der vorherigen Aufgabe ist sehr aufschlussreich. Ich habe einen Bereich der Programmierung kennen gelernt, den ich bisher nur als mystisches Wesen gesehen habe. Und ich muss als Ergebnis sagen, dass er mir nicht gefällt.

Der Syntax mit den unzählig verschachtelten Funktionen mit tausenden Klammern ist zu weit weg von dem bisher gewohnten Code in Java oder C#. Darüber hinaus die komplett andere Denkweise in der an den Code herangegangen wird, schreckt mich maximal ab Clojure oder eine andere Funktionale Programmiersprache zu lernen.

Allerdings nehme ich aus der Übung mit, dass auch in objektorientierten Programmiersprachen funktionale Ideen implementiert werden können. Vor allem die Problematik von State könnte damit eventuell an einigen Stellen elegant gelöst werden.




