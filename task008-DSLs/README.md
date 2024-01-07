# Task008: DSLs

Die Aufgabe hat mich vor einige Schwierigkeiten gestellt. Ich konnte mir zu Beginn nicht vorstellen
wie ich in meine bestehenden Anwendungen dieses Kurses (Bowling und Bowling-Service) sinnvoll einen
Builder und eine DSl bekommen sollte. Da wir aber die ganze Weihnachtszeit dafür verfügbar hatten,
habe ich mich dann entschieden Teiles eines anderes meiner Projekte ([Generated Rest](https://jitpack.io/#nerdfactor/generated-rest)) mit Builder 
und DSL umzubauen.


## Von riesiger Method zu kleinen Buildern
Im Kern besteht Generated Rest [zu diesem Zeitpunkt](https://github.com/nerdfactor/generated-rest/releases/tag/v0.0.10) aus einer über 800 Zeilen großen Klasse **GeneraterRestBuilder** die einen Builder von [Java Poet}(https://github.com/square/javapoet) mit allen notwendigen Informationen bestückt, um Code für einen Spring Boot Controller zu erstellen.

```Java
public class GeneratedRestBuilder {

	public TypeSpec buildController(ControllerConfiguration configuration) {
		// Create the new class.
		TypeSpec.Builder builder = TypeSpec
				.classBuilder(configuration.getClassName())
				.addAnnotation(RestController.class)
				.addModifiers(Modifier.PUBLIC);

		// Add elements to load and search entities.
		builder = this.addGetAllEntitiesMethod(builder, configuration);
		builder = this.addSearchEntitiesMethod(builder, configuration);

		// Add crud elements for entity.
		builder = this.addGetEntityMethod(builder, configuration);
		builder = this.addCreateEntityMethod(builder, configuration);
		builder = this.addSetEntityMethod(builder, configuration);
		builder = this.addUpdateEntityMethod(builder, configuration);
		builder = this.addDeleteEntityMethod(builder, configuration);

		[...]

		return builder.build();
	}

	[...]
}
```

Dieser riesige Code Block erledigt zwar seine Aufgabe sehr zuverlässig, lässt sich aber überhaut nicht mehr warten und erst nicht testen. Ich habe sie also schrittweise in kleinere Teile zerlegt.

Zuerst in eine neue Builder Klasse, die immer noch  die Methode der alten Klasse aufruft:

```Java
public class CrudMethodBuilder extends MethodBuilder implements MultiStepBuilder, BuildStep, ConfiguredBuilder {

	ControllerConfiguration configuration;

	@Override
	public TypeSpec.Builder build(TypeSpec.Builder builder) {
		builder = this.addGetEntityMethod(builder, configuration);
		builder = this.addCreateEntityMethod(builder, configuration);
		builder = this.addSetEntityMethod(builder, configuration);
		builder = this.addUpdateEntityMethod(builder, configuration);
		builder = this.addDeleteEntityMethod(builder, configuration);
		return builder;
	}

}
```

Und schließlich jede dieser ursprünglichen Methoden in eine separate Klasse, die als Builder mit Informationen bestückt werden kann.

```Java
public class CrudMethodBuilder extends MultiStepBuilder<TypeSpec.Builder> implements Configurable<ControllerConfiguration>, Buildable<TypeSpec.Builder> {

	protected ControllerConfiguration configuration;

	public TypeSpec.Builder build(TypeSpec.Builder builder) {
		this.and(CreateEntityMethodBuilder.create().withConfiguration(this.configuration));
		this.and(ReadEntityMethodBuilder.create().withConfiguration(this.configuration));
		this.and(UpdateEntityMethodBuilder.create().withConfiguration(this.configuration));
		this.and(SetEntityMethodBuilder.create().withConfiguration(this.configuration));
		this.and(DeleteEntityMethodBuilder.create().withConfiguration(this.configuration));
		this.steps.forEach(buildStep -> buildStep.build(builder));
		return builder;
	}

}
```

Die einzelnen Builder Klassen nutzen dann [Lombok mit @Builder bzw. @With Annotation](https://projectlombok.org/features/With), um die Builder Methoden nicht alle per Hand schreiben zu müssen.

```Java
@With
@NoArgsConstructor(access = AccessLevel.PRIVATE)
@AllArgsConstructor(access = AccessLevel.PROTECTED)
public class CreateEntityMethodBuilder implements Buildable<TypeSpec.Builder>, Configurable<ControllerConfiguration> {

	protected boolean hasExistingRequest;
	protected String requestUrl;
	protected TypeName requestType;
	protected TypeName responseType;
	protected TypeName entityType;
	protected boolean isUsingDto;
	protected SecurityConfiguration securityConfiguration;
	protected TypeName dataWrapperClass;

	public static CreateEntityMethodBuilder create() {
		return new CreateEntityMethodBuilder();
	}
}
```

Als Ergebnis können alle Builder [jetzt}((https://github.com/nerdfactor/generated-rest/releases/tag/v0.0.12)) unabhängig voneinander aufgerufen und in einem fluent Builder mit den notwendigen Informationen versorgt werden und sind endlich testbar.

```Java
@Test
void shouldCreateMethodUsingDto() {
	TypeSpec.Builder builder = TypeSpec.classBuilder("ExampleController")
			.addAnnotation(RestController.class)
			.addModifiers(Modifier.PUBLIC);

	CreateEntityMethodBuilder.create()
			.withHasExistingRequest(false)
			.withUsingDto(true)
			.withRequestUrl("/api/example")
			.withEntityType(ClassName.get(Example.class))
			.withRequestType(ClassName.get(ExampleForm.class))
			.withResponseType(ClassName.get(ExampleDto.class))
			.withSecurityConfiguration(null)
			.withDataWrapperClass(TypeName.OBJECT)
			.build(builder);

	String code = JavaFile.builder("eu.nerdfactor.test", builder.build()).build().toString();
	String expected = "...";
	Assertions.assertTrue(code.contains(expected));
}
```

![code-builder](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task008-DSLs/code-builder.png?raw=true)


## Import mit eigener DSL
Während Generated Rest zwar eigentlich als [Annotation Processor](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/Processor.html) vorgesehen ist, hatte ich nach der Vorlesung über DSLs direkt den Wunsch, die Konfiguration zum Erstellen der Controller auch über eine externe Datei laden zu können. In dieser sollte dann in einer einfachen Sprache beschrieben werden, was sonst über die Annotationen aus dem Code erstellt wird.

Genähert habe ich mich dem Problem, indem ich zuerst die in der bisher bestehende Form erstellte Konfiguration über [einen Exporter](https://github.com/nerdfactor/generated-rest/commit/69088f4fff0794bae5265b6b24697300b7e282eb) verfügbar gemacht habe. Damit kann man auswählen, ob der Processor wie bisher Java Code oder lieber eine Yaml Datei mit der Konfiguration erstellen soll.

```yaml
---
config:
  exporter: "eu.nerdfactor.springutil.generatedrest.export.YamlConfigExporter"
  classNamePrefix: "Generated"
  indentation: "    "
  log: "true"
  classNamePattern: "{PREFIX}Rest{NAME_NORMALIZED}Controller"
  dataWrapper: "java.lang.Object"
  dtoNamespace: ""
controllers:
  GeneratedRestCustomerController:
    className: "eu.nerdfactor.springutil.generatedexample.customer.GeneratedRestCustomerController"
    request: "/api/customers"
    entity: "eu.nerdfactor.springutil.generatedexample.customer.Customer"
    id: "java.lang.String"
    idAccessor: "getId"
	[...]
```

Damit hatte ich schon mal eine gute Übersicht, welche Daten über eine Konfigurationsdatei wieder zurück in die Anwendung kommen müssten. Aber diese automatisch erstellte Datei war mir zu groß und unübersichtlich und ich wollte sie zukünftig in einfacheren Sprache beschreiben können.

Mir schwebte dabei eine etwas sprechendere Beschreibung in der Art vor

```
new GeneratedRestCustomerController 
	for Customer and String
	at /api/customers
	with CustomerDto DTO
	with Security
```

um die Controller in einer Art Builder in der Konfigurationsdatei beschreiben zu können. Da ich das aus einer Textdatei lesen möchte, versuche ich dafür einen [Parser mit JavaCC](https://javacc.github.io/javacc/) zu entwickeln weil es "The most popular parser generator for use with Java applications." ist.

Die Dokumentation auf der Seite von JavaCC ist recht ausführlich, es gibt viele Beispiele und weil es so bekannt ist, gibt es auch Unmengen an Hilfe im Internet.

Der erste Entwurf des Parsers sieht dann so aus:

```
options {
  STATIC = false;
}

PARSER_BEGIN(GeneratedParser)
import java.io.*;

public class GeneratedParser {
  private static GeneratedConfig config = new GeneratedConfig();

  [...]
}
PARSER_END(GeneratedParser)

void parse() :
{}
{
  controllerDeclaration()
}

void controllerDeclaration() :
{
  config.controllerName = "";
  config.entityName = "";
  config.idName = "";
  config.dtoName = "";
  config.path = "";
  config.security = false;
}
{
  "new" ControllerName "for" EntityName "and" IdName
  "at" Path 
  "with" DtoName
  "with" Security
}

```

Im nächsten Schritt werden dann die Stellen, die ich auslesen möchte über eine Funktion als Text auszulesen und den Wert in config zu schreiben.

```
TOKEN : {
  <IDENTIFIER: (["a"-"z", "A"-"Z", "0"-"9", "_", "-", "/"])+ >
}

{
  "new" config.controllerName = Identifier() "for" config.entityName = Identifier() "and" config.idName = Identifier()
  "at" config.path = Identifier() 
  "with" config.dtoName = Identifier()
  "with" config.security = true
}

String Identifier() :
{
  Token t;
}
{
  t = <IDENTIFIER>
  { return t.image; }
}
```

![javacc](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task008-DSLs/javacc.png?raw=true)

Leider erstellt JavaCC damit zwar schon einen Parser, aber hat beim Parsen dann Probleme mit den Leerzeichen und security wird auch nicht richtig ausgewertet.

Sehr viel Recherche im Internet hat mich schließlich zum Ergebnis geführt, dass ich JavaCC noch sagen muss, dass er Leerzeichen etc. überspringen muss und das er die hard coded Strings ("new", "for", "and" etc.) auch noch als extra Token haben möchte.

```
SKIP : {
" "
| "\t"
| "\n"
| "\r"
| "\f"
}

TOKEN : {
  <WITH_SECURITY: "with Security">
  | <NEW: "new">
  | <FOR: "for">
  | <AND: "and">
  | <AT: "at">
  | <WITH: "with">
  | <IDENTIFIER: (["a"-"z", "A"-"Z", "0"-"9", "_", "-", "/"])+ >
}
```

Und das WITH_SECURITY Token kann geprüft werden, um nur dann true zuzuweisen.

```
(<WITH_SECURITY> { config.security = true; })?
```

Damit wird dann der Input richtig geparst und den Werten in der Config zugeordnet.

![javacc2](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task008-DSLs/javacc2.png?raw=true)

Die abschließende Grammatikdatei liegt [hier](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task008-DSLs/GeneratedParser.jj).


## Yaml mit Schema Validierung
Während ich den Parser mit JavaCC entwickelt habe, wurde mir immer klarer, dass die Definition eines vollständigen Grammatik den zeitlichen Rahmen sprengen würde und im Ergebnis die Definition der Konfigurationsdatei damit auch nicht einfacher oder übersichtlicher wird. Eigentlich gefällt sie mir in Yaml schon ganz gut. Das ist ein Standardformat, mit dem die meisten Entwickler zurecht kommen und auch gut funktioniert.

Aber einfach nur ein Export und Import nach Yaml zu machen, reicht mir nicht. Ich wollte schon irgendwie sicherstellen, dass die importierte Datei validiert werden kann und man eventuell auch beim Schreiben der Konfiguration durch Autovervollständigung unterstützt wird.

Die Lösung dafür erschien mir ein Schema zu entwickeln, mit der die Möglichkeiten der sehr flexiblen Markupsprache auf meinen domainspezifischen Anwendungsfall einschränke.

Da es anscheinend  nicht direkt einen Yaml Schema Standard gibt, nutze ich den [Json Schema](https://json-schema.org/) Standard, mit dem dann auch problemlos Yaml Dateien validiert werden können.

Dessen Dokumentation ist sogar noch besser als die von JavaCC und daher ist es relativ leicht aus einem Json Beispiel ein Schema zu erstellen. So leicht, dass es unzählige Generatoren dafür im Internet gibt und ChatGPT das auch machen kann. Aber deren Ergebnisse können leider wiederholende Elemente, wie z.B. die ControllerConfiguration nicht als getrennte Elemente erkennen und wiederholen sie einfach mehrfach. Manuell machen ist einfach doch am besten.

Um ein passendes Json Beispiel zu bekommen, habe ich Generated Rest um einen [zweiten Exporter nach Json erweitert](https://github.com/nerdfactor/generated-rest/commit/048445f1a81c2e29a0a98bdc1405e6b61582c8c6). Dadurch konnte ich dann schnell [dieses Schema](https://github.com/nerdfactor/generated-rest/blob/main/generated-rest.schema.json) entwickeln

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://raw.githubusercontent.com/nerdfactor/generated-rest/main/generated-rest.schema.json",
  "title": "Generated Rest",
  "description": "Generated Rest Controller Configuration",
  "type": "object",
  "properties": {
	"config": {
	  "description": "Configuration used during generation of REST controllers.",
	  "type": "object",
	  "properties": {...}
	},
	"controllers:": {
	  "description": "Configuration for REST controllers.",
	  "type": "object",
	  "additionalProperties": {
		"description": "One configuration for every controller that will be created.",
		"type": "object",
		"properties": {...}
	  }
	}
  }
}
```

und einen Importer schreiben, der die Datei importiert, mit Hilfe von [json-schema-validator](https://github.com/networknt/json-schema-validator) validiert und in eine GenerateRestConfig umwandelt:

```Java
public class ConfigImporter {

	public GeneratedConfigFile importFromFile(String path, String schemaPath) throws IOException {
		ObjectMapper mapper = ConfigMapper.forFile(path);
		JsonSchemaFactory schemaFactory = JsonSchemaFactory.getInstance(SpecVersion.VersionFlag.V202012);
		JsonNode json = mapper.readTree(Files.readString(Path.of(path)));
		JsonSchema schema = schemaFactory.getSchema(Files.readString(Path.of(schemaPath)));
		Set<ValidationMessage> validationResult = schema.validate(json);
		if (validationResult.isEmpty()) {
			return this.importFromFile(path);
		} else {
			throw new RuntimeException("Config file is invalid.");
		}
	}

}
```

Das eröffnet die Möglichkeit Generated Rest in der Zukunft so zu trennen, dass die Bibliothek weiterhin aus den Annotationen eine Konfiguration erstellt und davon losgelöst eine Anwendung aus einer Konfigurationsdatei den passenden Java Code erstellt.