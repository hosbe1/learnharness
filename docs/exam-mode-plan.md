# Klausur-Modus: Plan

Status: **Entwurf, wartet auf Freigabe.** Laut `AGENTS.md` beginnt die Umsetzung erst nach Freigabe des Plans. Die offenen Fragen unten werden vorher in den jeweiligen Issues entschieden.

Bezug upstream: [#443 Exam prep vs. full mastery](https://github.com/amosblomqvist/learnharness/issues/443), [#71 Multiple graphs](https://github.com/amosblomqvist/learnharness/issues/71).

## Ziel

Probeklausuren, die sich wie die echte Klausur anfühlen: ein ganzes Aufgabenblatt mit Zeitlimit, gelöst auf Papier, abgegeben als Fotos, korrigiert mit Punkten. Danach gibt es einen direkten Weg in Practice-Sessions zu den Schwachstellen.

Practice bleibt, was es ist: adaptives Training, ausdrücklich „not an exam“ (`src/agent/prompts/practice/10-doctrine.md`). Der Klausur-Modus ist ein eigener Session-Typ und ändert Practice nicht.

## Getroffene Entscheidungen

| Thema | Entscheidung |
|---|---|
| Ablauf | Ganzes Aufgabenblatt auf einmal, Abgabe am Ende, danach Korrektur |
| Zeitlimit | Weich: der Timer läuft sichtbar, nach Ablauf kommt ein Hinweis, Abgabe bleibt möglich, Überziehen wird im Ergebnis vermerkt |
| Abgabe | Lösung auf Papier, Fotos pro Aufgabe hochladen; ein bildfähiges Modell liest und korrigiert |
| Quellen | Altklausuren bestimmen den Stil (Aufgabentypen, Punkte, Formulierung), Folien/Skript und Übungsblätter den Stoffumfang. Der Wissensgraph ist **keine** Quelle |
| Material | Eigener Bereich „Module“: pro Modul Dateien, jede als Altklausur, Folien/Skript oder Übung markiert, getrennt vom Wissensgraphen |
| Graph | Klausurergebnisse schreiben nichts in den Graphen (wie Practice) |
| Nach der Korrektur | Punkte pro Aufgabe plus ein Knopf, der eine Practice-Session zu den schwachen Aufgaben startet |
| Code-Ziel | Entwicklung im Fork `hosbe1/learnharness`, später PR upstream mit Verweis auf #443 |

## Warum Modulmaterial nicht in den Graphen kommt

Der Graph enthält „die Ideen und ihre Zusammenhänge, keine Bewertungen“ (README). Importierte Notizen landen darin gleichberechtigt mit Gelehrtem. Würde man Folien und Altklausuren importieren, stünde im Graphen auch, was erst noch gelernt werden muss. Außerdem gehen beim Extrahieren genau die Merkmale verloren, die eine Klausur ausmachen (Aufgabentypen, Punkte). Deshalb ist die Modul-Bibliothek ein eigener Speicher.

## Architektur-Überblick

Der Klausur-Modus folgt dem Muster von Practice (`kind: "practice"`). Das sind etwa 18 Stellen, die heute auf die Session-Art verzweigen (u. a. `src/core/contracts/chat.ts`, `src/agent/teacher-agent.ts`, `src/server/chat-handler.ts`, `app/api/chats/route.ts`, `components/new-session.tsx`, `components/sidebar.tsx`, `components/chat.tsx`).

1. **Modul-Bibliothek** (neu)
   - Speicher: `data/modules/<moduleId>/module.json` plus kopierte Dateien. Liegt unter `LH_DATA_DIR`, also in der Desktop-App im userData-Verzeichnis.
   - `module.json`: Name, Dateien mit Typ (`exam` | `slides` | `exercises`), Erstellzeit.
   - Store-Adapter in `src/store/`, Port in `src/core/ports.ts`, HTTP-Routen unter `app/api/modules/`.
   - UI: eigener Bereich „Module“, um Module anzulegen und Dateien hinzuzufügen, zu markieren und zu entfernen.
2. **Session-Typ `exam`**
   - `ChatKind` wird um `"exam"` erweitert.
   - Ein `ExamState` am `ChatRecord` hält `moduleId`, Dauer, Start, Abgabe, das Aufgabenblatt und das Ergebnis.
   - Keine Extraktion bei der Settlement-Phase (wie Practice).
3. **Klausur-Agent**
   - Eigener Prompt-Ordner `src/agent/prompts/exam/`.
   - Werkzeuge:
     - `exam_compose`: legt das ganze Blatt mit Aufgaben, Punkten, Musterlösungen und Bewertungsschema fest. Lösungen und Bewertungsschema bleiben bis zur Korrektur im UI verborgen, analog zu `problem` in Practice.
     - `exam_grade`: vergibt Punkte und Feedback pro Aufgabe und gibt die Musterlösungen frei.
   - Liest die Moduldateien über die vorhandenen Datei-, PDF- und Bildwerkzeuge.
4. **Klausur-UI**
   - Ansicht des Aufgabenblatts, Timer, Foto-Upload pro Aufgabe, „Abgeben“.
   - Nach der Korrektur: Ergebnisansicht und der Knopf „Schwachstellen üben“.
5. **Übergang zu Practice**
   - Der Knopf erzeugt eine Practice-Session.
   - Als freies Material bekommt sie die schwach gelösten Aufgaben samt Fehlerdiagnose. Practice unterstützt freies Material bereits.

## Arbeitspakete (Issues)

Der Milestone „Klausur-Modus v1“ fasst alles zusammen. Die Reihenfolge entspricht den Abhängigkeiten.

1. Plan & offene Design-Fragen freigeben
2. Modul-Bibliothek: Datenmodell, Store, API
3. Modul-Bibliothek: UI
4. Session-Typ `exam`: Vertrag, Anlage, Routing
5. Klausur-Agent: Prompt und `exam_compose`
6. Klausur-UI: Aufgabenblatt, Timer, Abgabe mit Fotos
7. Korrektur: `exam_grade` und Ergebnisansicht
8. Übergang „Schwachstellen üben“ → Practice
9. Tests, Doku, Upstream-PR

## Offene Design-Fragen (vor der Umsetzung zu entscheiden)

- **Länge:** Wird die Bearbeitungszeit aus der Altklausur übernommen, beim Start gewählt oder beides (Vorschlag aus der Altklausur, änderbar)?
- **Teil-Klausuren:** Kann man auf einzelne Themen einschränken („nur Berechenbarkeit“), oder entsteht immer eine vollständige Klausur?
- **Modelle:** Welcher Slot erstellt die Klausur (Strong?), welcher korrigiert Fotos (Vision oder Strong, sofern bildfähig)?
- **Fotos:** Mehrere Fotos pro Aufgabe? Ein Sammel-Upload mit Zuordnung durch das Modell?
- **Nachfragen:** Darf man nach der Korrektur über einzelne Punkte diskutieren (Chat in der Klausur-Session)?
- **Sprache:** Richtet sich die Klausursprache nach dem Material (deutsche Altklausur → deutsche Klausur)? Bleibt die UI wie der Rest der App auf Englisch?
- **Wiederholung:** Kann man dieselbe Probeklausur später erneut schreiben?
- **Ohne Altklausur:** Was passiert, wenn ein Modul keine Altklausur hat (Stil nur aus Übungen ableiten oder Hinweis)?

## Nicht im Umfang von v1

- Mehrere Graphen pro Modul (#71)
- Exam/Mastery-Umschalter für Teaching (#443, Teil 1)
- Getippte Abgabe
- Spaced Repetition oder Notenstatistik über mehrere Probeklausuren

## Verifikation

- Unit- und Integrationstests nach dem Muster der Practice-Tests (`tests/practice-flow.test.ts`), mit `LH_DATA_DIR` auf ein temporäres Verzeichnis (vgl. upstream #334).
- `npm test`, Lint und Typecheck gemäß `docs/development.md`.
- Manueller Durchlauf in `npm run desktop:dev` mit einem Testmodul.
