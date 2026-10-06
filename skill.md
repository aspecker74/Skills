---
name: thorough-code-review
slug: thorough-code-review
description: >-
  Review, refactor, simplify, clean up, or debug existing code in any language
  or domain — not just when the user explicitly asks for a "bug hunt". Use this
  whenever the user shares working code and asks to simplify, clean up,
  refactor, "aufräumen"/"vereinfachen", remove duplication, or generally improve
  it, even if they don't explicitly mention bugs, edge cases, or concurrency.
  For any request touching existing code, structural cleanup and
  correctness/concurrency review are both part of a thorough job by default, not
  separate asks that need to be requested individually.
---
# Thorough Code Review

Für jede Aufgabe, bei der bestehender Code gelesen, aufgeräumt, vereinfacht,
refaktoriert oder debuggt werden soll — unabhängig von Sprache, Framework
oder Domäne.

## Warum dieser Skill existiert

In einer Session wurde ein Skript zunächst nur strukturell vereinfacht
(Duplikate entfernt, toter Code raus) — sauber, aber unvollständig. Ein
anderes Modell fand danach mehrere echte Laufzeit-Bugs im selben Code
(Race Conditions zwischen Zeitabläufen, ein früher `return`, der eine
Rückstell-Logik übersprang, ein Timing-Gleichstand zwischen zwei
Konfigurationswerten). Der Grund: "vereinfachen" wurde als "Struktur
aufräumen" interpretiert statt zusätzlich als "auf Korrektheit/
Nebenläufigkeit prüfen". Bei jeder Änderung an bestehendem Code gehören
beide Ebenen zusammen — unabhängig davon, ob es sich um ein Haustechnik-
Skript, ein Backend, ein UI-Component oder ein Skript in irgendeiner
anderen Domäne handelt.

## Grundprinzip

"Vereinfachen" / "Aufräumen" / "Refactoring" heißt nicht nur: kürzer,
weniger Duplikate, klarere Struktur. Es heißt auch: der Code tut danach
mindestens so viel Richtiges wie vorher — und das lässt sich nur
beurteilen, wenn man ihn auch auf Korrektheit prüft, nicht nur auf Form.
Beide Ebenen sind Teil derselben Aufgabe, auch wenn nur eine explizit
angefragt wurde.

## Vorgehen: zwei getrennte Durchgänge

Bei jeder Anfrage zu bestehendem Code routinemäßig zwei Durchgänge machen,
auch wenn nur einer explizit gefragt ist:

### Durchgang A — Struktur (Form)

- Für jede Funktion/Konstante/Klasse prüfen, ob sie tatsächlich noch
  irgendwo aufgerufen bzw. gelesen wird (nicht nur definiert). Ungenutzter
  Code ist historischer Ballast und kann raus — aber nur nach echter
  Verifikation, nicht nach Vermutung.
- Wiederholte Codeblöcke (gleiche Bedingungsketten, gleiche Wertelisten,
  gleiche Sequenzen an mehreren Stellen) zu einer "single source of truth"
  bündeln (Funktion, Konstante, Konfigurationsobjekt).
- Kommentare/Dokumentation auf Aktualität prüfen — alter Code hat oft
  Kommentare, die ein längst totes oder verändertes Verhalten beschreiben.

### Durchgang B — Korrektheit/Nebenläufigkeit (Inhalt)

Das ist der Teil, der leicht übersehen wird, weil er nicht "wie Aufräumen
aussieht". Explizit dafür Zeit nehmen, auch ungefragt. Fragen, die
systematisch durchgegangen werden:

- **Frühe Exits vor wichtigem Folgecode:** Gibt es einen Pfad, der eine
  Funktion frühzeitig verlässt (`return`, `break`, Exception, o. ä.), BEVOR
  Code erreicht wird, der eigentlich immer laufen sollte (Aufräumen,
  Zurücksetzen externer Zustände, Benachrichtigungen, Logging)? Für jeden
  frühen Exit-Punkt einer Funktion konkret prüfen: was steht danach noch,
  und würde das eigentlich in diesem Fall auch gebraucht?
- **Zwei zusammenhängende Zeit-/Schwellwerte mit knappem oder gleichem
  Verhältnis:** Wenn ein Wert X "abhängig von" oder "erst nach" einem
  anderen Wert Y wirken soll, und beide standardmäßig gleich oder sehr nah
  beieinander sind, ist das ein Fallstrick — kleine Verschiebungen
  (Ausführungsreihenfolge, Rundung, minimale Verzögerung) können die
  beabsichtigte Pufferzeit/Reihenfolge auf praktisch Null reduzieren.
  Prüfen, ob X strukturell klar größer/kleiner als Y sein sollte,
  unabhängig vom konfigurierten Wert.
- **Bedingungen, die einen Zustand abfragen, der zum Prüfzeitpunkt noch
  gar nicht gesetzt wurde:** Besonders bei asynchronem Code (Callbacks,
  Events, Promises, Timer): eine Bedingung kann syntaktisch korrekt sein,
  aber im tatsächlich relevanten Zeitfenster nie zutreffen, weil der
  geprüfte Wert erst später gesetzt wird. Für jeden asynchronen Ablauf
  fragen: "Was, wenn währenddessen ein externes Ereignis eintrifft, bevor
  der Ablauf abgeschlossen ist — reagiert etwas korrekt darauf?"
- **Event-Handler/Listener ohne Drosselung, die direkt teure oder
  weitreichende Aktionen auslösen:** Häufig wechselnde/„flatternde“ externe
  Werte (Sensoren, Netzwerk-Events, UI-Events) können denselben Handler
  sehr oft in kurzer Zeit feuern. Sicherheitskritische/unmittelbare
  Reaktionen sollten synchron bleiben; teure Neubewertungen sollten
  entprellt/gedrosselt werden.
- **Ein "Abschluss"-Vorgang (Cleanup, Rollback, Shutdown-Sequenz), der
  einen neuen "Start"-Wunsch blockiert statt ihm zu weichen:** Wenn eine
  Funktion früh abbricht, weil ein laufender Abschluss-/Aufräumvorgang
  aktiv ist, bedeutet das oft, dass ein bewusster manueller Neustart-
  Wunsch bis zum vollständigen Ablauf dieses Vorgangs ignoriert wird. Bei
  explizitem Trigger durch den Nutzer/Aufrufer sollte der laufende
  Abschlussvorgang aktiv abgebrochen werden können, nicht nur passiv
  blockieren.
- **Zwei fast identische Codeblöcke für "sofort" vs. "verzögert/
  asynchron":** Fast immer ein Zeichen, dass es eine gemeinsame Funktion
  geben sollte, die von beiden Stellen aufgerufen wird — und ein Hinweis,
  dass ein Fix zuvor nur an einer der beiden Kopien gemacht wurde.
  Suchgriff: nach nahezu identischen Blöcken suchen (grep auf markante
  Teilstrings, oder strukturell durchgehen), nicht nur exakte Duplikate.
- **Asynchrone/verzögerte Bestätigungen eigener Schreibvorgänge:** Wenn
  Code selbst einen Zustand setzt und dann auf ein Ereignis reagiert, das
  diesen Zustand widerspiegelt (z. B. ein State-Change-Callback, ein UI-
  Update), sicherstellen, dass die Reaktion nicht fälschlich als externe
  Aktion interpretiert wird. Robuster als ein reines "gerade beschäftigt"-
  Flag: sich merken, WELCHEN Wert man selbst erwartet, und die Bestätigung
  gezielt daran erkennen (funktioniert auch bei Verzögerung).

## Testen: über Syntax-/Compile-Checks hinausgehen

Ein Syntax-Check (Compiler, Linter, `node --check`, etc.) prüft nur
Grammatik, keine Laufzeit-Logik. Für nicht-triviale, zustands- oder
zeitabhängige Logik:

- Einen minimalen Mock-/Test-Harness bauen, der die externen
  Abhängigkeiten (Datenbank, Hardware, Netzwerk, Dateisystem, o. ä.)
  durch einfache Stubs ersetzt.
- Bei zeitabhängigem Code (Timer, Verzögerungen, Debounce, Retry-Logik):
  echte Timer durch eine virtuelle/kontrollierbare Uhr ersetzen (Liste
  ausstehender Callbacks + eine Funktion, die die Zeit gezielt vorspult),
  statt in echter Zeit zu warten. So lassen sich Sekunden- oder Minuten-
  lange Abläufe in Millisekunden reproduzierbar durchspielen.
- Für jeden konkret gefundenen oder behaupteten Bug einen Vorher/Nachher-
  Test schreiben, der den Fehlerpfad tatsächlich durchläuft und reproduziert
  — nicht nur einen Test, der "ohne Fehler durchläuft". Ein grüner Test, der
  den kritischen Pfad nie erreicht (z. B. wegen Restzustand aus einem
  vorherigen Testfall), ist wertlos. Für kritische Testfälle lieber einen
  frischen, isolierten Zustand aufsetzen statt Zustand zwischen Testfällen
  zu teilen.

## Kommunikation

- Wenn zwischen "Struktur" und "Korrektheit/Nebenläufigkeit" eine
  Priorisierung getroffen wird (oder eine davon aus Zeit-/Aufwandsgründen
  nicht gemacht wird), das transparent sagen: "Ich habe X geprüft, Y habe
  ich nicht separat durchgetestet — soll ich das auch machen?" statt
  stillschweigend nur einen Teil zu liefern und den Eindruck von
  Vollständigkeit zu erwecken.
- Wenn der Nutzer eine Liste konkreter Verdachtspunkte liefert (z. B. von
  einem anderen Modell, einem Code-Review-Tool, oder eigener Beobachtung):
  jeden einzeln im Code nachvollziehen (konkrete Stelle zeigen, Mechanismus
  erklären), nicht pauschal zustimmen oder pauschal abtun. Für jeden Punkt
  explizit einordnen: bestätigt als Bug / bestätigt als Verhalten, aber
  kein Bug (Trade-off) / nicht bestätigt (mit Begründung) — bevor irgendwas
  geändert wird.
- Bei mehrdeutigen Verhaltens-Trade-offs (z. B. Glättung, Schwellwerte,
  Sensitivität, Performance vs. Lesbarkeit) nicht einfach den Standardwert
  ändern oder das Verhalten umstellen, ohne zu fragen — lieber
  konfigurierbar machen (gleicher Default, aber einstellbar) oder explizit
  nachfragen, welches Verhalten gewünscht ist.

## Kurz-Checkliste zum Abhaken (bei jedem Review bestehenden Codes)

- [ ] Jede Funktion/Konstante/Klasse: wird sie wirklich verwendet? (toter
      Code raus, aber erst nach echter Verifikation)
- [ ] Wiederholte Listen/Bedingungsketten/Sequenzen zu einer Quelle
      gebündelt?
- [ ] Jeder frühe Exit einer Funktion: überspringt er Folgecode, der
      eigentlich immer laufen sollte?
- [ ] Zwei zusammenhängende Zeit-/Schwellwerte mit knappem oder gleichem
      Verhältnis zueinander?
- [ ] Jeder asynchrone Ablauf: was passiert, wenn währenddessen ein
      externes Ereignis eintrifft?
- [ ] Jeder Event-Handler ohne Drosselung: kann die Quelle "flattern"/oft
      feuern?
- [ ] Blockiert ein laufender Abschlussvorgang einen bewussten Neustart-/
      Override-Wunsch, statt ihm zu weichen?
- [ ] Test mit Stubs/Mocks (und bei Zeitabhängigkeit: virtueller Uhr)
      geschrieben, der den kritischen Pfad tatsächlich durchläuft — nicht
      nur ein Compile-/Syntax-Check?
- [ ] Für jeden behaupteten/gefundenen Bug: reproduzierender Vorher/
      Nachher-Test in frischem, isoliertem Zustand?