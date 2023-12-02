# Task004: Clean Code

## Cheat Sheet
![Cheat Sheet](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task004-clean_code/clean_code_cheat_sheet.jpg?raw=true)

Ein komprimiertes Cheat Sheet in dem die (meiner Meinung nach) wichtigsten Bestandteile von Clean Code visuell aufbereitet werden. Ich unterteile sie thematisch in Punkte die ich vor dem Programmieren, während dessen und danach beachten sollte. 

Natürlich verschwimmen die Bereiche und sind nicht immer klar zu trennen. Aber so werde ich immer daran erinnert zuerst die Anforderungen richtig zu verstehen und erst mit Design und Tests den Rahmen der Anwendung abzustecken, bevor ich die erste Zeile Code schreibe.

Auf der anderen Seite erinnert mich der "After Coding" Block an alle wichtigen Dinge, die zusätzlich zum Programmieren gemacht werden sollten.


## Beispielanwendung

Da ich die Beispielanwendung gleichzeitig für die nächste Aufgabe verwendet habe, ist der komplette Quellcode in einem extra Repository: https://github.com/nerdfactor/bht-bowling-game. Um die Beschreibung durchgängig zu halten, habe ich allerdings Screenshots und Codebeispiele direkt übernommen.

Die Beispielanwendung baut ganz grob auf dem [Bowling Game Code Kata](https://codingdojo.org/kata/Bowling/) auf. Allerdings habe ich mich am Ende nicht zu fest an die Beschreibung gehalten. Das Ergebnis ist eine winzig kleine Konsolenanwendung zum Berechnen eines Spielergebnisses.

![bowling_game](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task004-clean_code/bowling-game.png?raw=true)

### Understand your Requirements
Bevor ich angefangen habe, sollte ich die Anforderungen an die Anwendungen richtig verstehen. Die Regeln eines Bowling Spiels schienen mir leider offensichtlich genug, dass ich sie nicht nochmal nachgesehen habe. Also hier schon mal den ersten Fehler gemacht. Die Liste aus der Beschreibung des Katas habe ich nur überflogen:

-	Each game, or “line” of bowling, includes ten turns, or “frames” for the bowler.
-	In each frame, the bowler gets up to two tries to knock down all the pins.
-	If in two tries, he fails to knock them all down, his score for that frame is the total number of pins knocked down in his two tries.
-	If in two tries he knocks them all down, this is called a “spare” and his score for the frame is ten plus the number of pins knocked down on his next throw (in his next turn).
-	If on his first try in the frame he knocks down all the pins, this is called a “strike”. His turn is over, and his score for the frame is ten plus the simple total of the pins knocked down in his next two rolls.
-	If he gets a spare or strike in the last (tenth) frame, the bowler gets to throw one or two more bonus balls, respectively. These bonus throws are taken as part of the same turn. If the bonus throws knock down all the pins, the process does not repeat: the bonus throws are only used to calculate the score of the final frame.
-	The game score is the total of all frame scores.

### Design Before You Code
Auch wenn es nur eine sehr kleine Anwendung ist, wollte ich zumindest etwas Design im Vorfeld erledigen, um den Code dann besser Strukturieren zu können.

![Design](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task004-clean_code/bowling_design.jpg?raw=true)

Ich bin davon ausgegangen, dass jedes Spiel aus zehn Frames mit jeweils zwei Werten für umgefallene Pins bestehen wird.

### Write Tests
Bevor es tatsächlich mit Code los ging, habe ich dem Projekt zwei erste Unit Tests spendiert.

```Java
class BowlingGameTest {

	/**
	 * Check if a game without any knocked over pins has a score of zero.
	 */
	@Test
	void gameWithoutKnockedOverPinsHasScoreOfZero() {
		int expectedScore = 0;
		int amountOfRolls = 20;
		BowlingGame game = new BowlingGame();
		for (int i : new int[amountOfRolls]) {
			game.nextRoll(i);
		}
		int totalScore = game.countScore();
		Assertions.assertEquals(expectedScore, totalScore);
	}

	/**
	 * Check if the total score of the game is counted correctly using
	 * the same amount of knocked over pins for each roll.
	 */
	@Test
	void countTotalScoreOfKnockedOverPins() {
		int expectedScore = 60;
		BowlingGame game = new BowlingGame();
		for (int currentRoll = 1; currentRoll < 20; currentRoll++) {
			game.nextRoll(3);
		}
		int totalScore = game.countScore();
		Assertions.assertEquals(expectedScore, totalScore);
	}
}
```

Mit diesen sollten die beiden grundlegenden Funktionen zum Umwerfen von Pins und Berechnen des Ergebnis erfasst werden, auch wenn der Code dahinter noch nicht implementiert war.

```Java
public class BowlingGame {

	/**
	 * Executes the next roll in the game. The result of the roll
	 * are the knocked over pins, which will be recorded in order to
	 * count the score.
	 *
	 * @param knockedOverPins The amount of pins that where knocked over in the roll.
	 */
	public void nextRoll(int knockedOverPins) {

	}

	/**
	 * Calculates the score for the game. It will check every frame for strikes, spares
	 * or open frames and count their score accordingly.
	 * Can be called multiple times during the game and provides the correct score for
	 * the current game state.
	 *
	 * @return The current score for the game.
	 */
	public int countScore() {
		return 0;
	}
}
```
Von diesem Punkt aus habe ich die Methode tatsächlich implementiert und mit Hilfe von weiteren Tests die Funktionalität immer genauer abgeprüft.


### Meaningfull Names
In der Vergangenheit habe ich natürlich immer wieder den Fehler gemacht, Klassen und Variablen sehr willkürlich zu benennen ohne darauf zu achten, sie ein halbes Jahr später noch zu verstehen. Daher lege ich dieses Mal einen besonderen Augenmerk darauf.

```Java
/**
 * The amount of knocked over pins for each roll. The array will
 * never exceed the maximum amount of rolls.
 */
private final int[] knockedOverPinsPerRoll;

public TenPinBowlingGame() {
	this.currentRoll = 0;
	this.knockedOverPinsPerRoll = new int[AMOUNT_OF_MAX_ROLLS];
}
```
Daher heißen die Variablen hier **knockerOverPinsPerRoll**, wenn der Array die Anzahl an Pins enthält, die in einem Wurf umgeworfen sind. Anstatt sie einfach nur **rolls** zu nennen. Ich versuche die Namen dafür erst so sprechend wie möglich zu schreiben und kürze später eventuell Füllwörter (z.B. den Anfang von **amountOfKnockedOverPinsPerRoll**) weg. So sollte aber weiterhin der beschreibende Charakter nicht angefasst werden.

Weitere Beispiele dafür sind die Benennung von Methoden für logische Prüfungen

```Java
private boolean wouldKnockOverWrongAmountOfPins(int knockedOverPins) {
	return knockedOverPins < 0 || knockedOverPins > AMOUNT_OF_PINS;
}
```

bei denen es mir wichtiger ist, dass klar wird was sie machen, anstatt die logischen Vergleiche zu sehen.

Oder aber auch die Nutzung von klar definierten Werten für **Magic Numbers** mit zutreffenden Namen, anstatt die Nummern im Code zu verstreuen.

```Java
public static final int AMOUNT_OF_MAX_ROLLS = 21;

public static final int AMOUNT_OF_FRAMES = 10;

public static final int AMOUNT_OF_PINS = 10;
```


### Code is Read More Than Is Is Written
Mit dem Gedanken im Kopf, dass der Code besonders gut zu lesen sein sollte, habe ich einige meiner Methoden inhaltlich überarbeitet. Das Ziel war es immer, die Methode wie eine Anweisung runter lesen zu können und nicht über komplizierte logische Konstrukte zu stolpern. Diese sollten lieber hinter entsprechend benannten Variablen oder Methodenaufrufen versteckt werden.

Der Inhalt der Prüfungen wird daher in separate Methoden ausgelagert.

```Java
public void nextRoll(int knockedOverPins) throws WrongAmountOfPinsException, MaxAmountOfRollsExceededException {
	if (wouldKnockOverWrongAmountOfPins(knockedOverPins)) {
		throw new WrongAmountOfPinsException();
	}
	if (wouldExceedMaxRolls()) {
		throw new MaxAmountOfRollsExceededException();
	}
	this.knockedOverPinsPerRoll[this.currentRoll] = knockedOverPins;
	this.currentRoll++;
}
```

Das hilft auch dabei den Leser nicht zu überraschen (**Principle Of Least Astonishment**).

### Single Level Of Abstraction / Single Resposibility Principle
Eine sehr starke Überschneidung mit der Lesbarkeit und der Benennung gab es auch bei dem Ziel innerhalb eines Abstraktionslevels zu bleiben und nicht ständig zu springen. 

Aus der erstmal sehr stur herunter programmierten Methode zum Berechnen des Ergebnis

```Java
public int countScore() {
	int currentScore = 0;
	int checkedRoll = 0;
	for (int frame = 0; frame < AMOUNT_OF_FRAMES; frame++) {
		if (this.knockedOverPinsPerRoll[checkedRoll] == AMOUNT_OF_PINS) {
			int strikeBonus = this.knockedOverPinsPerRoll[checkedRoll + 1] + this.knockedOverPinsPerRoll[checkedRoll + 2];
			currentScore += AMOUNT_OF_PINS + strikeBonus;
			checkedRoll++;
		} else if (this.knockedOverPinsPerRoll[checkedRoll] + this.knockedOverPinsPerRoll[checkedRoll + 1] == AMOUNT_OF_PINS) {
			int spareBonus = this.knockedOverPinsPerRoll[checkedRoll + 2];
			currentScore += AMOUNT_OF_PINS + spareBonus;
			checkedRoll += 2;
		} else {
			currentScore += this.knockedOverPinsPerRoll[checkedRoll] + this.knockedOverPinsPerRoll[checkedRoll + 1];
			checkedRoll += 2;
		}
	}
	return currentScore;
}
```

werden daher die Prüfungen und Berechnungen der Werte für die einzelnen Frames herausgenommen.
Diese Methode soll sich nur um die Berechnung des Ergebnisses für das ganze Spiel kümmern und nicht um alles was dafür irgendwie notwendig ist.

Nebenher wird sie damit auch deutlich lesbarer.

```Java
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

Nebenbei hilft es auch dabei bzw. profitiert es stark davon, wenn seine Methoden klein hält (**Keep Functions Small**).

### Review und Refactoring
Während der Entwicklung habe ich gemerkt, dass ich mich beim ursprünglichen Design in eine falsche Richtung verrannt hatte. Die Trennung zwischen Game und Frames erschien zwar logisch, führte bei der Implementierung des Scorings zu richtig hässlichem Code, der auch nicht funktionierte.

```Java
public void nextRoll(int knockedOverPins) {
	Frame lastFrame = null;
	if (!this.frames.isEmpty()) {
		lastFrame = this.frames.get(this.frames.size() - 1);
		if (lastFrame.isPlayed()) {
			lastFrame = new Frame();
			this.frames.add(lastFrame);
		}
	} else {
		lastFrame = new Frame();
		this.frames.add(lastFrame);
	}
	lastFrame.nextRoll(knockedOverPins);
}
```

Hier wurde es notwendig die ursprüngliche Entscheidung zu überdenken und vom Design abzuweichen. Anstatt die einzelnen Würfe direkt in Frames zu unterteilen, war es vielleicht deutlich einfacher sie als Würfe aufzuzeichnen bzw. direkt die umgeworfenen Pins pro Wurf. Das macht die Methode zum Erfassen deutlich schlanker und führt am Ende auch bei der Berechnung des Ergbnisses zu weniger Durcheinander. 

```Java
public void nextRoll(int knockedOverPins) {
	this.knockedOverPinsPerRoll[this.currentRoll] = knockedOverPins;
	this.currentRoll++;
}
```

Eine weitere Änderung wurde sinnvoll, als mir aufgefallen ist, dass ich eine bestimmte Form von Bowling implementiere. Da Bowling neben dem "10 Pin Bowling" auch noch andere Formen kennt, wollte ich diese Möglichkeiten auch innerhalb der Anwendung durscheinen lassen.

Mit Hilfe der Funktionen in meinem IDE habe ich daher einfach aus der bestehende BowlingGame Klasse ein entsprechendes Interface extrahiert.

![extract_interface](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task004-clean_code/extract_interface.png?raw=true)

Das eigentliche Spiel ist dann eine spezifische Implementierung davon.

```Java
/**
 * Implementation of {@link BowlingGame} for ten pin bowling.
 */
public class TenPinBowlingGame implements BowlingGame {
}
```

### Code Analysis
Da mir in der letzten Aufgabe SonarQube besonders gut gefallen habe, hab ich auch diese kleine Anwendung zum Ende nochmal prüfen lassen.

![sonarqube-1](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task004-clean_code/sonarqube-1.png?raw=true)

![sonarqube-2](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task004-clean_code/sonarqube-2.png?raw=true)

Auch wenn bei diesem kleinen Projekt eigentlich nicht mehr viele Probleme hochkommen sollten, kann man immer noch die Vererbung der Exceptions richtig rücken und eine bewusste Entscheidung darüber treffen in der Main Methode der Konsolenanwendung doch eine Ausgabe in die Konsole zu geben.

### Abschließende Gedanken
Die bewusste Auseinandersetzung mit Clean Code war äußerst angenehm. Denn obwohl die Prinzipien eigentlich offensichtlich sein und intuitiv angewendet werden sollten, sehe ich an dieser kleinen Beispielanwendung wie viel mehr ich mich darauf konzentrieren sollte.

Außerdem wurde bei der Übung klar, wie sehr die Prinzipien miteinander verzahnt sind. Sobald man an der einen Stelle anfängt den Code schöner zu machen, ziehen die anderen Bereiche gleich mit. 

Der größte Nachteil ist aber natürlich die zusätzliche Zeit die während der Entwicklung investiert wird von deren langfristigen Nutzen man den Managementbereich nur schwer überzeugen kann. Den Spagat schafft man wahrscheinlich nur, wenn man an der Qualität seiner eigenen Arbeit konsequent festhält.