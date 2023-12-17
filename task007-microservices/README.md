# Task007: Microservices

Da ich die Beispielanwendung gleichzeitig für die vorhergehende Aufgabe verwendet habe, ist der komplette Quellcode in einem extra Repository: https://github.com/nerdfactor/bht-bowling-service. Um die Beschreibung durchgängig zu halten, habe ich allerdings Screenshots und Codebeispiele direkt übernommen.

## Bowling Service
Als Grundlage für einen kleinen Service nutze ich den Code der Clean Code Aufgabe und transformiere die Bowling Konsolenanwendung in einen Spring Boot Service, der als Teil einer (theoretischen) größeren Anwendung die Verarbeitung der Spiele übernimmt.

Der Service beginnt als Gradle Projekt, in dem ich [Spring Boot mit einem ganz einfachen ersten Controller](https://github.com/nerdfactor/bht-bowling-service/commit/9417f89b6c9e8a90da6fa4f81200d6e5a80ca86c) zum Laufen bekomme.

```Java
@RestController
@RequestMapping("/api/v1")
public class MainController {

	/**
	 * The main endpoint of the application should just show a life sign
	 * so that consumers know that there is something behind the api.
	 *
	 * @return ResponseEntity with HTTP 200.
	 */
	@GetMapping
	public ResponseEntity<?> index() {
		return ResponseEntity.ok().build();
	}

	/**
	 * A simple status endpoint to show more information about the api.
	 *
	 * @return ResponseEntity with HTTP 200 and basic status info.
	 */
	@GetMapping("/status")
	public ResponseEntity<?> status() {
		return ResponseEntity.ok("{\"status\":true}");
	}
}
```

```
eu.nerdfactor.bowling.App                : Starting App using Java 17.0.9 with PID 4625
eu.nerdfactor.bowling.App                : No active profile set, falling back to 1 default profile: "default"
o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 8080 (http)
o.apache.catalina.core.StandardService   : Starting service [Tomcat]
o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.16]
o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
eu.nerdfactor.bowling.App                : Started App in 1.695 seconds (process running for 2.413)
```

Der erste Aufruf sieht schon sehr vielversprechend aus.

![service-first-status](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task007-microservices/service-first-status.png?raw=true)

Danach kommen in mehreren Schritten [Entities für das Spiel](https://github.com/nerdfactor/bht-bowling-service/commit/fda8014144ea14cd0cb1945ca675f496d21775db), [Service und Repository Layers](https://github.com/nerdfactor/bht-bowling-service/commit/42a5dff9093bf31078847779c566cf349f468c10) zur Verarbeitung der Daten und schließlich ein [Api Layer mit REST Operationen](https://github.com/nerdfactor/bht-bowling-service/commit/8de3e085e5cbaf9dac8d76a908706dde0f9b2b4f) dazu. Damit können die Bowling Informationen schon mal angenommen und abgelegt werden.

```json
POST http://localhost:8080/api/v1/bowling

{
	"currentRoll": 14,
	"knockedOverPinsPerRoll": [8,0,4,3,2,5,10,6,1,8,0,9,1,6]
}
```

```json
GET http://localhost:8080/api/v1/bowling/1

{
	"id": 1,
	"currentRoll": 14,
	"knockedOverPinsPerRoll": [8,0,4,3,2,5,10,6,1,8,0,9,1,6],
	"currentScore": 0,
}
```

```json
GET http://localhost:8080/api/v1/bowling

[
	{
		"id": 1,
		"currentRoll": 14,
		"knockedOverPinsPerRoll": [8,0,4,3,2,5,10,6,1,8,0,9,1,6],
		"currentScore": 0,
	}
]

```


### Das große Refactoring
Während es bei der Clean Code Aufgabe sehr sinnvoll war die Funktionalität des Bowlings innerhalb der TenPinBowlingGame Klasse zu halten, sollte der Web Service sauber nach Layers getrennt werden.

Die Entity sollte daher nur noch die Daten halten:

```Java
@Entity
public class BowlingGame {

	/**
	 * Internal identifier for this specific {@link BowlingGame}.
	 */
	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private int id;

	/**
	 * The roll this {@link BowlingGame} is on. Required to access the correct
	 * array element in the knockedOverPins.
	 */
	private int currentRoll;

	/**
	 * The amount of knocked over pins for each roll. The array will
	 * never exceed the maximum amount of rolls.
	 */
	private List<Integer> knockedOverPinsPerRoll = new ArrayList<>();

	/**
	 * The current score for this {@link BowlingGame}.
	 */
	private int currentScore;

}
```

Der Service Layer führt die Logik aus:

```Java
@Service
public class BowlingService {

	/**
	 * Inject a specific implementation of a repository for data access.
	 */
	private final BowlingGameRepository gameRepository;

	/**
	 * Adds the next roll to a game by executing its next roll with the provided amount
	 * of knocked over pins.
	 *
	 * @param id              The id of the game.
	 * @param knockedOverPins The amount of knocked over pins.
	 * @return The updated game.
	 * @throws WrongAmountOfPinsException        If a wrong amount off knocked over pins was passed.
	 * @throws MaxAmountOfRollsExceededException If the maximum of rolls in the game was exceeded.
	 * @throws EntityNotFoundException           If the game could not be found.
	 */
	public BowlingGame addNextRoll(int id, int knockedOverPins) {
		[...]
	}

	/**
	 * Calculates the score for a game with the currently used scoring strategy.
	 *
	 * @param id The id of the game.
	 * @return The updated game.
	 * @throws EntityNotFoundException If the game could not be found.
	 */
	public BowlingGame calculateCurrentScore(int id) {
		[...]
	}

}
```

Und der Api Layer bietet die Funktionen öffentlich an:

```Java
@RestController
@RequestMapping("/api/v1/bowling")
public class BowlingController {

	private final BowlingService bowlingService;

	@PostMapping(value = "/{id}/roll/{pins}")
	public ResponseEntity<BowlingGame> nextRoll(@PathVariable int id, @PathVariable(name = "pins") int knockedOverPins) {
		BowlingGame game = this.bowlingService.addNextRoll(id, knockedOverPins);
		return ResponseEntity.ok(game);
	}

	@GetMapping(value = "/{id}/score")
	public ResponseEntity<BowlingGame> calculateScore(@PathVariable int id) {
		BowlingGame game = this.bowlingService.calculateCurrentScore(id);
		return ResponseEntity.ok(game);
	}

}
```
Um die Funktionalität des Services weiter konfigurierbar zu machen, werden sie als Strategy implementiert und über die Dependency Injection von Spring hinein gereicht.

```java
@Component
public class TenPinBowlingScoring implements ScoringStrategy {

	/**
	 * Calculates the score for a {@link BowlingGame} using a specified {@link BowlingRuleset}.
	 * It will check every frame for strikes, spares or open frames and count their score
	 * accordingly. Can be called multiple times during the game and provides the correct
	 * score for the current game state.
	 *
	 * @param game    The {@link BowlingGame} to score.
	 * @param ruleset The {@link BowlingRuleset} use for scoring.
	 * @return The total score.
	 */
	@Override
	public int countScore(BowlingGame game, BowlingRuleset ruleset) {
		this.game = game;
		this.ruleset = ruleset;
		int currentScore = 0;
		int checkedRoll = 0;
		for (int frame = 0; frame < ruleset.amountOfFrames(); frame++) {
			if (game.isRollAStrike(checkedRoll, ruleset)) {
				currentScore += countScoreForStrike(checkedRoll);
				checkedRoll++;
			} else if (game.isRollASpare(checkedRoll, ruleset)) {
				currentScore += countScoreForSpare(checkedRoll);
				checkedRoll += 2;
			} else {
				currentScore += countScoreForOpenFrame(checkedRoll);
				checkedRoll += 2;
			}
		}
		return currentScore;
	}

	[...]

}
```

Damit kann über den passenden Http Endpunkt die Berechnung des Spielergebnisses genauso schön durchgeführt werden, wie in der Bowling Konsolenanwendung.

```json
GET http://localhost:8080/api/v1/bowling/1/score

{
	"id": 1,
	"currentRoll": 20,
	"knockedOverPinsPerRoll": [3,3,3,3,3,3,3,3,3,3,3,3,3,3,3,3,3,3,10,3,4
	],
	"currentScore": 71,
}

```

## Spring Features entdecken
Der Bowling Service ist natürlich nicht meine erste Spring Boot Anwendung, aber es gibt viele Features, die ich mir noch nie ansehen konnte. Die Aufgabe dient daher als tolle Möglichkeit mit diesen Features zu spielen.

### Custom Validation
Ich habe in anderen Anwendungen bereits ganz grundlegend Spring Validation verwendet, um einfach die eingehenden Daten zu prüfen. Im Bowling Game konnte ich das bereits einbauen. Roll und Score müssen daher mindestens 0 sein, sondern würden die Daten z.B. beim Speichern in der Datenbank abgelehnt werden.

```java
@Entity
@Validated
public class BowlingGame {

	/**
	 * The roll this {@link BowlingGame} is on. Required to access the correct
	 * array element in the knockedOverPins.
	 */
	@Min(0)
	private int currentRoll;

	/**
	 * The current score for this {@link BowlingGame}.
	 */
	@Min(0)
	private int currentScore;
}
```

Soweit war mir das bisher bekannt. Ich habe allerdings noch nie selber Custom Validator implementiert. Und beim Bowling Game bietet sich eine gute Gelegenheit dazu. Den Rolls und dem Score kann ich zwar ein festes Minimum geben, aber die maximale Größe ergibt sich ja dynamisch nach dem konfigurierten BowlingRuleset.

```java
/**
 * Ruleset for a bowling game. Defines game rules and checks.
 */
public interface BowlingRuleset {

	/**
	 * The amount of maximum rolls in a bowling game.
	 */
	int amountOfMaxRolls();

	/**
	 * The maximum score that can be achieved.
	 */
	int amountOfMaxScore();

	[...]

}
```

Die Custom Validierung kann mit Hilfe einer Annotation und dem passenden Validator implementiert werden.

```java
@Target({FIELD, PARAMETER})
@Retention(RUNTIME)
@Constraint(validatedBy = MaxPossibleScoreValidator.class)
public @interface MaxPossibleScore {

	String message() default "{BowlingGame.MaxPossibleScore.Exceeded}";

	Class<?>[] groups() default {};

	Class<? extends Payload>[] payload() default {};
}
```

```java
@Component
public class MaxPossibleScoreValidator implements ConstraintValidator<MaxPossibleScore, Object> {

	@Autowired
	private BowlingRuleset bowlingRuleset;

	@Override
	public void initialize(MaxPossibleScore constraintAnnotation) {
		ConstraintValidator.super.initialize(constraintAnnotation);
	}

	@Override
	public boolean isValid(Object value, ConstraintValidatorContext context) {
		if (this.bowlingRuleset == null) {
			return false;
		}
		if (value instanceof Integer || value instanceof Long) {
			return ((int) value) <= this.bowlingRuleset.amountOfMaxScore();
		}
		return false;
	}
}
```

Damit kann diese Prüfung dann wie die bekannten Validations in der Entity genutzt werden.

```java
/**
 * The current score for this {@link BowlingGame}.
 */
@Min(0)
@MaxPossibleScore
private int currentScore;
```

Ist schon ziemlich cool.

Beim Test wurde eine Besonderheit klar, die ich so nicht erwartet hatte. Der Converter möchte einen BowlingRuleset als Dependency mit @Autowired injected haben und scheitert daran. Denn die Entity wird nicht von Spring verwaltet sondern über Hibernate. Selbst von man sie als Spring @Component in das Ecosystem zwingt, wird der Converter weiterhin von Hibernate erzeugt.

Ich musste also einen Weg finden, damit Hibernate irgendwie den BowlingService erzeugen kann, damit der in den Converter injected werden kann. Zum Glück hat jemand im Internet das Problem bereits gelöst und zeigt wie man die Hibernate Properties so erweitern kann, dass die bestehende Factory genutzt wird um Validators zu erzeugen.

```java
@Configuration
public class ValidationConfig {


	/**
	 * Add Springs validation factory to Hibernate, so that it uses a factory
	 * that Spring can inject bean into. This allows the custom validators to
	 * get a {@link BowlingRuleset} bean.
	 *
	 * @param validator Some {@link Validator} from Spring.
	 * @return Customized Spring Properties.
	 * @see <a href="https://stackoverflow.com/a/56557189"/>
	 */
	@Bean
	public HibernatePropertiesCustomizer hibernatePropertiesCustomizer(final Validator validator) {
		return hibernateProperties -> hibernateProperties.put("javax.persistence.validation.factory", validator);
	}
}
```


### OpenApi (Swagger)
Zur Beschreibung einer Api bietet sich OpenApi besonders gut. Vor allem wenn man teamübergreifend zusammenarbeiten möchte, werden Unklarheiten so effektiv entgegen gewirkt. Da die Beschreibung allerdings regelmäßig mit der Implementierung auseinander geht und mühsam nachgepflegt werden muss, habe ich immer schon mit Swagger geliebäugelt, womit die Spezifikation aus dem Code erstellt wird. Allerdings hatte ich nie die Chance das irgendwo einzubauen.

Für Swagger gibt es für Spring Boot natürlich auch die passende Library, die als Dependency dem Projekt hinzugefügt werden kann.

```kotlin
dependencies {
	implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.3.0")
}
```

Die Beschreibung erfolgt dann als Domain Specific Language mit einem riesigen Satz von Annotation direkt in den Controllern.

```java
@PostMapping(value = "/{id}/roll/{pins}", produces = "application/hal+json")
@Operation(
		summary = "Execute the next roll of a Bowling Game.",
		description = "The next roll of a Bowling Game can be executed by providing the Id of the specific game and the pins that should be next over by the roll."
)
@ApiResponse(responseCode = "200", content = {@Content(schema = @Schema(implementation = BowlingGameDto.class), mediaType = "application/hal+json")})
@ApiResponse(responseCode = "404", content = {@Content()})
public ResponseEntity<BowlingGameDto> nextRoll()

```

Und daraus wird dann automatisch die passende OpenApi Beschreibung erzeugt:

```json
"/api/v1/bowling/{id}/score": {
	"get": {
		"tags": [
			"Bowling"
		],
		"summary": "Calculate the score of a Bowling Game.",
		"description": "The score of a Bowling Game can be calculated by providing the Id of the specific game. The score will be calculated for the current state of the game. It does not have to be finished to be scored.",
		"operationId": "calculateScore",
		"parameters": [
			{
				"name": "id",
				"in": "path",
				"required": true,
				"schema": {
					"type": "integer",
					"format": "int32"
				}
			}
		],
		"responses": {
			"200": {
				"description": "OK",
				"content": {
					"application/hal+json": {
						"schema": {
							"$ref": "#/components/schemas/BowlingGameDto"
						}
					}
				}
			},
			"404": {
				"description": "Not Found"
			}
		}
	}
}
```

Das macht die Pflege deutlich leichter. Ich sehe aber auch die Gefahr, dass die Spezifikation dann zu leicht verändert werden kann und die Nutzer sich nicht auf eine stabile Api verlassen können. Ich werde das auf jeden Fall in einem meiner nächsten Projekte mit einbauen.

Als schicker Zusatz wird auch ein Swagger UI erzeugt, mit dem man sich etwas durch die Api klicken kann.

![swagger](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task007-microservices/swagger.png?raw=true)

Das würde ich aber in produktiven Umgebungen nur ungerne mit ausliefern.


### Hateoas
Eine OpenApi Beschreibung ist schon eine ziemlich gute Möglichkeit um eine Api zu entdecken. Interessanter könnte aber auch eine [Hateoas Hypermedia](https://en.wikipedia.org/wiki/HATEOAS) Beschreibung sein, mit der die Clients die Api selber entdecken können. Das ist auch so ein Feature, von dem ich schon unzählige Konferenztalks gesehen es aber noch nie implementiert habt.

Spring bringt natürlich auch dafür die passende Library mit.

```kotlin
dependencies {
	implementation("org.springframework.boot:spring-boot-starter-hateoas")
}
```

Die Antworten der Controller können als RepresentationModel geschickt werden, die über die passenden Builder Method der Library um Hypermedia Referenzen angereichert werden. Der Content der Antwort wird dann nicht als json sondern im mit Metainformationen angereichertem Format [HAL](https://en.wikipedia.org/wiki/Hypertext_Application_Language) serialisiert.

```json
GET http://localhost:8080/api/v1/bowling/1

{
	"id": 1,
	"currentRoll": 14,
	"knockedOverPinsPerRoll": [8,0,4,3,2,5,10,6,1,8,0,9,1,6],
	"currentScore": 0,
	"_links": {
		"self": {
			"href": "http://localhost:8080/api/v1/bowling/1"
		}
	}
}
```

Diese Referenz unter **_links.self.href** sagt dem Client eindeutig, unter welcher URI er die Resource wieder finden kann. Bei der Abfrage nach einer einzelnen Resource ist das vielleicht noch nicht wirklich spannend, aber sobald man eine Liste von Resourcen abfragt, wird der Vorteil klar.

```json
[
	{
		"id": 1,
		"currentRoll": 14,
		"knockedOverPinsPerRoll": [8,0,4,3,2,5,10,6,1,8,0,9,1,6],
		"currentScore": 0,
		"_links": {
			"self": {
				"href": "http://localhost:8080/api/v1/bowling/1"
			}
		}
	},
	{
		"id": 2,
		"currentRoll": 20,
		"knockedOverPinsPerRoll": [3,3,3,3,3,3,3,3,3,3,3,3,3,3,3,3,3,3,10,3,4
		],
		"currentScore": 71,
		"_links": {
			"self": {
				"href": "http://localhost:8080/api/v1/bowling/2"
			}
		}
	},
	{
		"id": 3,
		"currentRoll": 0,
		"knockedOverPinsPerRoll": [],
		"currentScore": 0,
		"_links": {
			"self": {
				"href": "http://localhost:8080/api/v1/bowling/3"
			}
		}
	}
]
```
Da jede Resource in dieser Liste ebenfalls ein Referenz auf sich selber haben, weiß der Client direkt wo er diese weiter abfragen kann. Es muss nirgendwo fest rein programmiert werden, wie die einzelnen Resourcen gefunden werden und es wäre auch kein großes Problem, wenn sich an der URI etwas ändert. Eine Änderung wäre ja wieder Teil der Antwort und könnte direkt wieder verwendet werden.

Wenn man dann noch einen zentralen Einstiegspunkt mit einer Übersicht der Funktionen anbietet, hat der Client alles was er zum Nutzen der Api benötigt.

```json
{
	"status": true,
	"version": "1.0.0",
	"_links": {
		"self": {
			"href": "http://localhost:8080/api/v1"
		},
		"docs": {
			"href": "http://localhost:8080/api/v1/docs"
		},
		"swagger": {
			"href": "http://localhost:8080/api/v1/swagger"
		},
		"startBowlingGame": {
			"href": "http://localhost:8080/api/v1/bowling/start"
		},
		"listBowlingGames": {
			"href": "http://localhost:8080/api/v1/bowling"
		}
	}
}
```

Hier wird z.B. beschrieben wo die Docs der Api liegen und wo man ein neues Bowling Game starten bzw. sich die bestehenden Bowling Games anzeigen lassen kann. Da die Antwort von **startBowlingGame** und **listBowlingGame** dann wieder Referenzen auf die Resourcen beinhalten werden, findet der Client immer einen Weg weiter.

Und in Verbindung mit einer [RESTful](https://en.wikipedia.org/wiki/REST) Api, weiß der Client auch wie er sie lesen (Http GET), anlegen (Http POST), ändern (Http PUT) oder löschen (Http DELETE) kann.

<br>

Um die Hateoas Links an die Resourcen hängen zu können, müssen die Klassen RepresentationModel erweitern. Das vermischt mir allerdings zu sehr die Entities mit dem Api Layer. Daher trenne ich eine Data Transfer Object (DTO) ab, das dann von RepresentationModel erben kann. Zwischen der Entity und DTO kann der Controller dann mit Hilfe von [ModelMapper](https://modelmapper.org/) automatisch übersetzen.

```Java
@RestController
@RequestMapping("/api/v1/bowling")
public class BowlingController {

	private final ModelMapper modelMapper;
	private final BowlingService bowlingService;

	@PostMapping(value = "/{id}/roll/{pins}")
	public ResponseEntity<BowlingGameDto> nextRoll(@PathVariable int id, @PathVariable(name = "pins") int knockedOverPins){
		BowlingGame game = this.bowlingService.addNextRoll(id, knockedOverPins);
		BowlingGameDto dto = this.modelMapper.map(game, BowlingGameDto.class);
		dto.add(linkTo(methodOn(BowlingRestController.class).readGame(dto.getId())).withSelfRel());
		return ResponseEntity.ok(dto);
	}

	@GetMapping(value = "/{id}/score")
	public ResponseEntity<BowlingGameDto> calculateScore(@PathVariable int id) {
		BowlingGame game = this.bowlingService.calculateCurrentScore(id);
		BowlingGameDto dto = this.modelMapper.map(game, BowlingGameDto.class);
		dto.add(linkTo(methodOn(BowlingRestController.class).readGame(dto.getId())).withSelfRel());
		return ResponseEntity.ok(dto);
	}

}
```
Das DTO bekommt dann die Links mit den notwendigen Hateoas verweisen. Auch hier wieder ein schönes Beispiel von Builder Methoden **linkTo** und **methodOn**.

### Automatisches Deployment
Wenn man davon ausgeht, dass der Bowling Service als Microservice Bestandteil eines größeren Systems ist, wird es natürlich notwendig ihn nicht manuell einzusetzen. Damit bietet die Aufgabe auch hier wieder die Chance mich um mit einer komplett neuen Thematik zu beschäftigen. Denn es wäre sehr schön, wenn ich Änderungen am Service nur commiten müsste und dann landet am Ende die neue Version automatisch auf meinem Server.

Dafür habe ich bereits in der vorherigen Aufgabe (Build und CI/CD) die notwendigen Workflows mit Github Actions abgebildet. Sobald ich in git einen neuen Tag einrichte, wird über den Workflow

- der Code aus dem Repository ausgecheckt
- ein neues Jar der Spring Boot Anwendung gebaut
- das Jar in einem Docker Container gepackt
- und der Docker Container in die Github Registry abgelegt

Auf dem Server wird der Docker Container dann regelmäßig durch [Watchtower](https://github.com/containrrr/watchtower) automatisch aktualisiert.

```
time="2023-12-18T02:00:44Z" level=info msg="Found new ghcr.io/nerdfactor/bowling-service:latest image (0c19d1cd6adb)"
time="2023-12-18T02:01:31Z" level=info msg="Stopping /bowling-service (a9ac73fb32ef) with SIGTERM"
time="2023-12-18T02:01:32Z" level=info msg="Creating /bowling-service"
```
```
eu.nerdfactor.bowling.App                : Starting App v1.2.0 using Java 17.0.9 with PID 1
eu.nerdfactor.bowling.App                : No active profile set, falling back to 1 default profile: "default"
.s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.
.s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 341 ms. Found 1 JPA repository interface.
o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 8080 (http)
o.apache.catalina.core.StandardService   : Starting service [Tomcat]
o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.16]
o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 13765 ms
com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection conn0: url=jdbc:h2:mem:db user=SA
com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
o.s.b.a.h2.H2ConsoleAutoConfiguration    : H2 console available at '/h2-console'. Database available at 'jdbc:h2:mem:db'
o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]
org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.3.1.Final
o.h.c.internal.RegionFactoryInitiator    : HHH000026: Second-level cache disabled
org.hibernate.orm.deprecation            : HHH90000021: Encountered deprecated setting [javax.persistence.validation.factory], use [jakarta.persistence.validation.factory] instead
o.s.o.j.p.SpringPersistenceUnitInfo      : No LoadTimeWeaver setup: ignoring JPA class transformer
o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)
j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
JpaBaseConfiguration$JpaWebConfiguration : spring.jpa.open-in-view is enabled by default. Therefore, database queries may be performed during view rendering. Explicitly configure spring.jpa.open-in-view to disable this warning
o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 8080 (http) with context path ''
eu.nerdfactor.bowling.App                : Started App in 34.868 seconds (process running for 39.398)
```

Und der Server gibt dann unter https://bowling.nrdfctr.app/api/v1 tatsächlich die Statusinformationen mit der aktuellen Versionnummer zurück:

```json
{
    "status": true,
    "version": "1.2.0",
    "_links": {
        "self": {
            "href": "http://bowling.nrdfctr.app/api/v1"
        },
        "docs": {
            "href": "http://bowling.nrdfctr.app/api/v1/docs"
        },
        "swagger": {
            "href": "http://bowling.nrdfctr.app/api/v1/swagger"
        },
        "startBowlingGame": {
            "href": "http://bowling.nrdfctr.app/api/v1/bowling/start"
        },
        "listBowlingGames": {
            "href": "http://bowling.nrdfctr.app/api/v1/bowling"
        }
    }
}
```


