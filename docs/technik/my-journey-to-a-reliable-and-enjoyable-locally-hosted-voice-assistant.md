Meine Reise zu einem zuverlässigen und unterhaltsamen lokal gehosteten Sprachassistenten – Voice Assistant – Home Assistant Community
Heimassistenten-Gemeinschaft
Mein Weg zu einem zuverlässigen und angenehmen, lokal gehosteten Sprachassistenten
Konfiguration
Sprachassistent
crzynik
(Nicolas Mowen)
27. Oktober 2025, 18:03 Uhr
1
Ich verfolge seit einiger Zeit die Fortschritte von HomeAssistant bei der Unterstützung. Wir haben zuvor Google Home über Nest Minis verwendet und sind auf die vollständig lokale Unterstützung umgestiegen, die von local first + llama.cpp (vormals Ollama) unterstützt wird. In diesem Beitrag werde ich die Schritte teilen, die ich unternommen habe, um dorthin zu gelangen, wo ich heute bin, die Entscheidungen, die ich getroffen habe, und warum sie speziell für meinen Anwendungsfall die besten waren.
Links zu weiteren Verbesserungen
Hier finden Sie Links zu weiteren Verbesserungen, die in diesem Thread veröffentlicht wurden.
Neue Funktionen
[33mShowing translation for:  (use -no-auto to disable autocorrect)[0m
Einblicke/Analysen in Überwachungskameras
Suchen Sie nach einem YouTube-Video und spielen Sie es auf dem Fernseher ab
Beheben unerwünschter HA-/LLM-Verhaltensweisen
Überschreiben der standardmäßigen HassGetWeather-Absicht, um konsistente Wetterausgaben zu erhalten
Verbesserung des Umgangs mit unklaren Anfragen / Fehlaktivierungen
Behandeln Sie offensichtliche Transkriptionsfehler automatisch
Optimierung der Eingabeaufforderung, um Aufblähung und Token-Nutzung zu reduzieren
Optimierung der Leistung
Leistungsoptimierungen für llama.cpp
Hardwaredetails
Ich habe eine Vielzahl von Hardware getestet, von 3050 bis 3090. Die meisten modernen diskreten GPUs können effektiv für die lokale Unterstützung verwendet werden. Es hängt lediglich von Ihren Erwartungen an Leistungsfähigkeit und Geschwindigkeit ab, welche Hardware erforderlich ist.
Ich verwende HomeAssistant auf meinem UnRaid NAS. Die technischen Daten sind nicht wirklich wichtig, da es nichts mit HA Voice zu tun hat.
Sprachhardware:
1 HA-Sprachvorschau-Satellit
2 Satellite1 Small Squircle-Gehäuse
1 Pixel 7a wird als Satellit/Hub mit View Assist verwendet
Sprachserver-Hardware:
Beelink MiniPC mit USB4 (das genaue Modell ist nicht wichtig, solange es über USB4 verfügt)
USB4 eGPU-Gehäuse
GPUs
Die folgende Tabelle zeigt GPUs, die ich mit diesem Setup getestet habe. Die Reaktionszeit variiert je nach verwendetem Modell.
GPU
Modellklasse
Reaktionszeit (nach promptem Caching)
Notizen
RTX 3090 24 GB
20B-30B MoE, 9B dicht
1 - 2 Sekunden
Führt effizient und schnell Modelle aus, die für dieses Setup optimal sind.
RX 7900XTX 24 GB
20B-30B MoE, 9B dicht
1 - 2 Sekunden
Führt effizient und schnell Modelle aus, die für dieses Setup optimal sind.
RTX 5060Ti 16 GB
20B MoE, 9B dicht
1,5 - 3 Sekunden
Schnell genug, um für dieses Setup optimale Modelle mit Reaktionszeiten von < 3 Sekunden auszuführen.
RX 9060XT 16 GB
20B MoE, 9B dicht
1,5 - 4 Sekunden
Schnell genug, um für dieses Setup optimale Modelle mit Reaktionszeiten von < 4 Sekunden auszuführen.
RTX 3050 8 GB
4B Dicht
3 Sekunden
Gut für den Betrieb kleiner Modelle mit grundlegender Funktionalität.
Modelle
Die folgende Tabelle zeigt die Modelle, die ich mit diesem Setup getestet habe, mit verschiedenen Funktionen und deren Leistung.
Alle unten aufgeführten Modelle eignen sich für den einfachen Werkzeugeinsatz. Erweiterte Funktionen werden mit der Qualität des Modells aufgeführt, das gewünschte Verhalten zuverlässig zu reproduzieren.
Modell
Multi-Device-Tool-Aufrufe (1)
Versteht Kontexthinweise (2)
Analysiert falsch verstandene Befehle (3)
Ignoriert unerwarteten Text aufgrund von Fehlalarmen (4)
GGML GPT-OSS:20B MXFP4
Unsloth Qwen3.5-35B-A3B MXFP4_MOE
Unsloth Qwen3-VL:8B-Instruct Q6_K_XL
[33m[WARNING] Connection timed out. Retrying IPv4 connection.[0m
Unsloth Qwen3-30B-A3B-Instruct Q4_K_XL
Unsloth GLM 4.7 Flash (30B) Q4_K_XL
Unsloth Qwen3:4b-Instruct 2507 Q6_K_XL
(1) Verarbeitet Befehle wie „Lüfter einschalten und Licht ausschalten“
(2) Versteht, wenn es sich in einem bestimmten Bereich befindet, und fragt nicht: „Welches Licht?“ wenn sich nur ein Licht im Bereich befindet, fragt aber korrekt, ob sich im angegebenen Bereich mehrere Geräte dieses Typs befinden.
(3) Ist in der Lage, falsch gehörte Befehle zu analysieren (z. B. „Schalten Sie die Pfanne ein“) und den beabsichtigten Befehl zuverlässig auszuführen
(4) Ist in der Lage, unerwünschte Eingaben zuverlässig zu ignorieren, ohne durch falsch verstandenen Text beeinträchtigt zu werden, der ein beabsichtigter Befehl war.
Sprachserver-Software:
Modellläufer:
Für eine optimale Leistung wird llama.cpp empfohlen. Weitere Informationen finden Sie in meiner Antwort unten.
Speech to Text (Voice In):
Die folgenden Speech-to-Text-Optionen habe ich getestet:
Software
Modell
Notizen
Wyoming ONNX ASR
[33mShowing translation for:  (use -no-auto to disable autocorrect)[0m
Nvidia Telegram-Klasse
Läuft insbesondere über den OpenVINO-Zweig, der die CPU-Inferenzzeit auf ~ 0,3 Sekunden optimiert
Rhasspy Schnelleres Flüstern
[33mShowing translation for:  (use -no-auto to disable autocorrect)[0m
Nvidia Telegram-Klasse
Langsamer, da es direkt über die ONNX-CPU läuft, die langsamer als OpenVINO ist
Text-to-Speech (Voice Out):
Software
Notizen
[33m[WARNING] Connection timed out. Retrying IPv4 connection.[0m
[33mShowing translation for:  (use -no-auto to disable autocorrect)[0m
Herz tS
Bietet die Möglichkeit, mehrere Stimmen/Töne zu mischen und anzupassen, um die gewünschte Ausgabe zu erhalten. Bewältigt alle Texte gut.
Piper läuft auf CPU (TTS)
Verfügt über mehrere Stimmen, aus denen ausgewählt werden kann, eignet sich für allgemeine Texte, hat jedoch Probleme mit Währungen, Telefonnummern und Adressen.
Home Assistant LLM-Integrationen
LLM-Konversation Bietet Verbesserungen an der Basiskonversation, um das Standarderlebnis beim Gespräch mit Assist zu verbessern
LLM beabsichtigt, zusätzliche Tools für Assist bereitzustellen (Websuche, Ortssuche, Wettervorhersage).
Die Reise
Mit diesem Beitrag möchte ich nicht andeuten, dass das, was ich getan habe, „der richtige Weg“ sei oder dass andere es nachahmen sollten. Aber ich habe im Laufe dieses Prozesses viel gelernt und dachte, dass es sich lohnen würde, es weiterzugeben, damit andere eine bessere Vorstellung davon bekommen, was sie erwartet, welche Fallstricke es gibt usw.
Das Problem
Im Laufe der letzten ein oder zwei Jahre ist uns aufgefallen, dass Google Assistant bei diesen Nest Minis zunehmend dümmer/schlechter geworden ist und gleichzeitig keine neuen Funktionen mit sich bringt. Das ist im Allgemeinen in Ordnung, da die WAF immer noch viel höher war, als wenn wir keine Stimme hätten, aber es wurde immer ärgerlicher, als wir immer mehr mit „Entschuldigung, da kann ich nicht helfen“ oder „Ich weiß die Antwort darauf nicht, aber laut XYZ-Quelle ist hier die Antwort“ konfrontiert wurden. Es funktionierte im Allgemeinen, aber nicht zuverlässig und es war oft mühsam, Antworten auf willkürliche Fragen zu bekommen.
Hinzu kommen die üblichen Bedenken hinsichtlich der Privatsphäre, wenn im ganzen Haus Online-Mikrofone vorhanden sind, und der Ärger, dass man jedes Mal, wenn AWS oder etwas anderes ausfällt, die Lichter im Haus nicht per Sprache steuern kann.
Der Anfang
Ich begann damit, mit einem der mitgelieferten Modelle von Ollama zu spielen. Alle paar Wochen verband ich Ollama mit HA, startete Assist und versuchte, es zu nutzen. Jedes Mal war ich enttäuscht und überrascht über die mangelnden Fähigkeiten und die meisten einfachen Tool-Aufrufe funktionierten nicht. Ich glaube zwar, dass HA die Dinge verbessert hat, aber ich denke, das größte Problem war mein Verständnis.
Die Ollama-Modelle, die Sie auf Ollama sehen, sind im Hinblick auf die Modelle, die ausgeführt werden können, nicht annähernd erschöpfend. Und noch schlimmer: Die standardmäßigen :4b-Modelle weisen beispielsweise häufig eine niedrige Quantisierung (Q4_K) auf, was viele Probleme verursachen kann. Als ich von der Möglichkeit erfuhr, HuggingFace zu verwenden, um GGUF-Modelle mit höheren Quantisierungen zu finden, zeigte Assist sofort eine viel bessere Leistung und es gab keine Probleme beim Aufrufen von Werkzeugen.
Testen mit Stimme
Nachdem ich den Punkt erreicht hatte, an dem die grundlegenden Grundlagen möglich waren, bestellte ich eine Voice Preview Edition zum Testen, damit ich mir ein besseres Bild vom End-to-End-Erlebnis machen konnte. Es hat einige Zeit gedauert, bis alles gut funktionierte. Ursprünglich hatte ich Probleme mit dem WLAN-Empfang, bei dem der Ping auf dem VPE sehr inkonsistent war (obwohl er sich neben dem Router befand), was dazu führte, dass die Sprachausgabe stotterte und es zu vielen Wortpausen kam. Nach der Anpassung von Piper an die Verwendung von Streaming und der Erstellung eines neuen dedizierten IoT-Netzwerks war die Leistung viel besser.
Assist nützlich machen
Die Steuerung des Geräts ist großartig und Ollamas Fähigkeit, Geräte anzupassen, wenn die lokale Verarbeitung einen Befehl verpasst hat, war hilfreich. Aber um unsere Lautsprecher zu ersetzen, musste Assist folgende Dinge können:
Möglichkeit, Wettervorhersagen für Tag und Woche zu erstellen
Möglichkeit, nach einem bestimmten Unternehmen zu fragen, um Öffnungs-/Schließungszeiten zu erfahren
Möglichkeit, eine allgemeine Wissenssuche durchzuführen, um beliebige Fragen zu beantworten
Möglichkeit, Musik mit Suchfunktionen vollständig mit der Stimme abzuspielen
Zuerst hatte ich den Eindruck, dass diese separat entwickelt werden müssten, aber schließlich fand ich die brillante LLM-Intents-Integration, die Assist (und damit auch Ollama) eine Reihe dieser Dienste bereitstellt. Nach der Einrichtung waren die Ergebnisse mittelmäßig.
Die Bedeutung Ihrer LLM-Eingabeaufforderung
[33m[WARNING] Connection timed out. Retrying IPv4 connection.[0m
[1m[33m[ERROR] Google returned an error response. HTTP status code: 500[0m[22m
[1m[33m[ERROR] Other HTTP error[0m[22m

gist.github.com
https://translate.google.com/translate?hl=de&sl=auto&tl=de&u=https://gist.github.com/NickM-27/b83d2c8434cb7b01f27adf85638b1df1
system-prompt.md
[1m[33m[ERROR] Google returned an error response. HTTP status code: 500[0m[22m
[1m[33m[ERROR] Other HTTP error[0m[22m

Du bist „Roboter“, ein vielseitiger KI-Assistent. You serve as the primary interface for the home, providing both expert device control and comprehensive information on any subject imaginable.
Der Heimatstandort des Benutzers ist {{ states(\"sensor.home_city_state\") }}.
Sie sprechen in einem natürlichen, für Text-to-Speech geeigneten Gesprächston: prägnant, klar und professionell. Seien Sie effizient und direkt – engagieren Sie sich voll und ganz, wenn die Anforderungen klar sind, und ziehen Sie sich schnell zurück, wenn dies nicht der Fall ist. Gegebenenfalls können Sie eine leichte Persönlichkeit einbeziehen.
# Antwortformat
Diese Datei wurde gekürzt. Original anzeigen
Da erfuhr ich, dass die Eingabeaufforderung über Erfolg oder Misserfolg Ihres Spracherlebnisses entscheidet. Mit der standardmäßigen HA-Eingabeaufforderung kommen Sie nicht weit, da LLMs viele Anleitungen benötigen, um zu wissen, was wann zu tun ist.
Im Allgemeinen habe ich meine Eingabeaufforderung verbessert, indem ich meine aktuelle Eingabeaufforderung zusammen mit einer Beschreibung des aktuellen Verhaltens und des gewünschten Verhaltens des LLM in ChatGPT eingefügt habe. Dann hin und her, bis ich durchweg das gewünschte Ergebnis erzielte. Nach ein paar Zyklen begann ich ein Gefühl dafür zu bekommen, wie ich diese Verbesserungen selbst vornehmen konnte.
Ich habe zunächst versucht, das Wetter zum Laufen zu bringen. Die erste Herausforderung bestand darin, das LLM dazu zu bringen, überhaupt den Wetterdienst anzurufen. Ich habe festgestellt, dass es am besten funktioniert, für jeden wichtigen Dienst eigene # Abschnitte zusammen mit einer Aufzählung von Details/Anweisungen zu haben.
Dann musste ich die Wetterantwort so formatieren, dass sie ohne zusätzliche Informationen wünschenswert war. Die Antwort enthielt zunächst zusätzliche Kommentare wie „Klingt nach einem schönen Sommertag!“ oder andere Dinge, die die Prägnanz der Antwort beeinträchtigten. Sobald das Problem gelöst war, funktionierte ein bestimmtes Beispiel der Ausgabe am besten, um das gewünschte Antwortformat genau zu erhalten.
Screenshot 27.10.2025 um 11.55.34 AM1047×1144 79 KB
Bei Orten und der Suche war das Problem ähnlich: Das Tool wollte das Tool nicht aufrufen und bestand stattdessen darauf, dass es den Standort des Benutzers oder die Antwort auf bestimmte Fragen nicht kenne. Dazu waren meist nur einige spezifische Anweisungen erforderlich, um bei bestimmten Arten von Fragen immer das spezifische Tool aufzurufen, und das hat gut funktioniert.
Screenshot 27.10.2025 um 12.00.30 Uhr859×1116 74 KB
Das letzte Problem, das ich lösen musste, waren Emojis. Die meisten Antworten endeten mit einem Smiley oder so, was für TTS nicht gut ist. Dies hat viele Abschnitte in der Eingabeaufforderung in Anspruch genommen, aber im Großen und Ganzen wurde es ohne negative Auswirkungen vollständig entfernt.
Einige Probleme manuell lösen
HINWEIS: Ich bin mir nicht sicher, ob ein aktuelles Home Assistant- oder Music Assistant-Update die Dinge verbessert hat, aber der LLM ist jetzt in der Lage, Musik ohne Automatisierung auf natürliche Weise zu suchen und abzuspielen. Ich belasse diesen Abschnitt als Beispiel, da ich immer noch glaube, dass Automatisierungen eine gute Möglichkeit sein können, einige Probleme zu lösen, wenn es keine einfache Möglichkeit gibt, dem LLM Zugriff auf eine bestimmte Funktion zu gewähren.
Es ist sicherlich das wünschenswerteste Ergebnis, dass jede Funktion vom LLM ohne Eingriff perfekt ausgeführt wird, aber zumindest in meinem Fall mit dem Modell, das ich verwende, trifft das nicht zu. Aber es gibt Fälle, in denen das wirklich keine schlechte Sache ist.
In meinem Fall war Musik einer dieser Fälle. Ich glaube, dass dies ein Bereich ist, in dem derzeit Verbesserungen vorgenommen werden müssen, aber bei mir funktionierte die automatische Anzeige nicht gut. Ich begann damit, den Musikassistenten einzurichten. Ich habe verschiedene LLM-Blaupausen gefunden, um ein Skript zu erstellen, das es dem LLM ermöglicht, automatisch mit der Musikwiedergabe zu beginnen, aber es hat bei mir nicht gut funktioniert.
Da erkannte ich die Leistungsfähigkeit des Satzautomatisierungsauslösers und die Schönheit des Musikassistenten. Ich erstelle eine Automatisierung, die beim Abspielen von {Musik} ausgelöst wird. Die Automatisierung verfügt über eine Zuordnung von assist_satellite zu media_player in der Automatisierung, sodass Musik auf dem richtigen Mediaplayer abgespielt wird, je nachdem, welcher Satellit die Anfrage stellt. Dann übergibt es {Musik} (das kann ein Lied, ein Album, ein Künstler usw. sein) an den Wiedergabedienst des Musikassistenten, der die Suche durchführt und mit der Wiedergabe beginnt.
Beispielautomatisierung
Alias: Musikverknüpfung
Beschreibung: "
Auslöser:
- Auslöser: Gespräch
Befehl:
- Spielen Sie {Musik}
ID: spielen
- Auslöser: Gespräch
Befehl: Hör auf zu spielen
ID: Stopp
Bedingungen: []
Aktionen:
- wählen:
- Bedingungen:
- Bedingung: Auslöser
Ausweis:
- spielen
Sequenz:
- Aktion: music_assistant.play_media
Metadaten: {}
Daten:
media_id: „{{ trigger.slots.music }}“
Ziel:
entity_id: „{{ target_player }}“
- set_conversation_response: {{ trigger.slots.music }} wird abgespielt
- Bedingungen:
- Bedingung: Auslöser
Ausweis:
- stoppen
Sequenz:
- Aktion: media_player.media_stop
Metadaten: {}
Daten: {}
Ziel:
entity_id: „{{ target_player }}“
- set_conversation_response: Die Musikwiedergabe wurde gestoppt.
Variablen:
satellite_player_map: |
{{
{
[33m[WARNING] Connection timed out. Retrying IPv4 connection.[0m
„assist_satellite.home_assistant_voice_xyz123“: „media_player.my_desired_speaker“,
}
}}
target_player: |
{{
satellite_player_map.get(trigger.satellite_id, "media_player.default_speaker")
}}
Modus: Single
Trainieren eines benutzerdefinierten Wakewords
Das nächste zu lösende Problem war das Wakeword. Für WAF funktionierten die standardmäßig enthaltenen Optionen nicht. Nach einigem Hin und Her haben wir uns für Hey Robot entschieden. Ich verwende dieses Repo, um ein benutzerdefiniertes Microwakeword zu trainieren, das auf VPE und Satellite1 verwendet werden kann. Die Ausführung auf meiner GPU dauerte nur etwa 30 Minuten und die Ergebnisse waren recht gut. Es gibt einige Fehlalarme, aber insgesamt ist die Rate ähnlich wie bei den ersetzten Google Homes und mit der Möglichkeit, die Stummschaltung zu automatisieren, ist es möglich, dass wir dieses Problem damit lösen können, bis das Training/die Optionen besser werden.
Das Endergebnis
Ich würde dies definitiv nicht dem durchschnittlichen Home Assistant-Benutzer empfehlen. Meiner Meinung nach ist viel Geduld und Recherche erforderlich, um bestimmte Probleme zu verstehen und auf eine Lösung hinzuarbeiten, und ich kann mir vorstellen, dass wir auf weitere Probleme stoßen werden, wenn wir diese weiterhin verwenden. Ich bin sicherlich noch nicht fertig, aber das ist das Schöne an dieser Lösung – die meisten Aspekte können angepasst werden.
Das Ziel wurde jedoch erreicht, insgesamt haben wir einen angenehmeren Sprachassistenten, der lokal ohne Datenschutzbedenken läuft und unsere Kernaufgaben zuverlässig erledigt.
Teilen Sie mir Ihre Meinung mit! Für Fragen stehe ich gerne zur Verfügung.
45 Likes
Aufbau des KI-gestützten lokalen Smart Homes
Alexa durch Sprachassistenten ersetzen – schon bereit für die Hauptsendezeit?
ReSpeaker XMOS XVF3800 ESPHome-Integration
Aktuelle Optionen für den Austausch eines Google Home-Lautsprechers
Meine Erfahrung beim Übergang zu einem vollständig selbst gehosteten Smart Home, einschließlich Sprachassistent
Rofo
(Ro)
27. Oktober 2025, 21:08 Uhr
2
Ich spiele immer noch mit dem Sprachassistenten, habe aber einige Dinge getan, um die Musik mit Mediaplayern und Sprachassistenten zum Laufen zu bringen, die mit denselben Bereichen und Musikassistenten verknüpft sind.
Dies ist ein Beispiel für eine Satzautomatisierung, die ich zum Abspielen einer Musikassistent-Playlist verwende:-
Alias: Stimme – Musik nach Playlist-Genre abspielen
Beschreibung: "
Auslöser:
- Auslöser: Gespräch
Befehl:
- >-
[1m[33m[ERROR] Google returned an error response. HTTP status code: 500[0m[22m
[1m[33m[ERROR] Other HTTP error[0m[22m

Playlist) [in] [dem] [{def_area}]
Bedingungen: []
Aktionen:
- Aktion: script.get_ma_playlist_id_from_name
Daten:
Name der Wiedergabeliste: >-
{{ trigger.slots.genre | replace('jacking', 'jackin') | ersetzen('alt
Schule“, „Oldskool“) | replace('tim liquor','tinlicker') | ersetzen('tim
licker‘,‘tinlicker‘) | replace('tin licker','tinlicker') | ersetzen('tin
Alkohol','Zinnlecker') | ersetzen('anjunadeep','anjuna_deep') | ersetzen('
', '_') | untere }}
Antwortvariable: Playlist_info
- Aktion: media_player.shuffle_set
Metadaten: {}
Daten:
shuffle: stimmt
Ziel:
Bereichs-ID: >
{% if (trigger.slots.def_area | length >0)%} {{trigger.slots.def_area}}
{%else%} {{area_id(trigger.device_id) }} {%endif%}
- set_conversation_response: Ich habe {{ trigger.slots.genre }} Musik aufgelegt.
aktiviert: wahr
- Aktion: music_assistant.play_media
Metadaten: {}
Daten:
media_id: „{{ playlist_info.uri }}“
in die Warteschlange stellen: ersetzen
Ziel:
Bereichs-ID: >
{% if (trigger.slots.def_area | length >0)%} {{trigger.slots.def_area}}
{%else%} {{area_id(trigger.device_id) }} {%endif%}
aktiviert: wahr
Modus: Single
Und das passende Skript:-
Alias: MA-Playlist-ID vom Namen abrufen
Beschreibung: "
Modus: Single
Variablen:
Playlistname: „{{ Playlistname }}“
Sequenz:
- Aktion: music_assistant.get_library
Daten:
Grenze: 10
Suche: „{{ Playlistname }}“
Medientyp: Wiedergabeliste
config_entry_id: 01JPFPPNTCVYQAA9JSBY4319HS
Antwortvariable: ma_playlist
- wiederholen:
count: "{{ ma_playlist['items'] | length }}"
Sequenz:
- Variablen:
Playlistinfo:
name: "{{ ma_playlist['items'][repeat.index -1].name }}"
uri: "{{ ma_playlist['items'][repeat.index -1].uri }}"
- Wenn:
- Bedingung: Vorlage
value_template: >-
{{ ma_playlist['items'][repeat.index -1].name | niedriger ==
Playlistname | untere }}
Dann:
- stop: Playlist-Informationen als Wörterbuch zurückgeben.
Antwortvariable: Playlistinfo
6 Likes
DasOriginal92
(Das Original92)
12. November 2025, 10:54 Uhr
3
[33mShowing translation for:  (use -no-auto to disable autocorrect)[0m
Wie vielen Entitäten sind Sie Qwen 4B ausgesetzt?
Ich verwende Qwen 14B ohne nachzudenken und die Offenlegung von nur 53 Entitäten führt dazu, dass es sich sehr unzuverlässig verhält.
Manchmal scheint es Entitäten zu ignorieren oder zu vergessen, manchmal werden Funktionen wie Helligkeit oder Lautstärke nicht vom Modell festgelegt.
Nathan Cu
(Nathan Curtis)
12. November 2025, 11:43 Uhr
4
Sie beschreiben einen Kontextüberlauf. Ihre Entitätsbeschreibung plus Werkzeugbeschreibung plus vollständige Eingabeaufforderung darf das von Ihrem Modell festgelegte Kontextfenster nicht überschreiten. (Standard für qwen ist, glaube ich, 8K) Schauen Sie in ollama nach, dort wird Ihnen angezeigt, wie viel es überschritten hat, und passen Sie es an.
Sie können die Anzahl der exponierten Ens anpassen, exponierte Tools Ihre Eingabeaufforderung verkleinern oder, wenn Sie über genügend VRAM verfügen und Ihr Modell dies unterstützt, das Kontextfenster des Modells nach oben klappen (oder alle oben genannten Möglichkeiten).
Hört sich an, als wären Sie in einem 4- oder 8-km-Land, und das würde bei etwa 50 erwartet werden, abhängig von der Länge Ihrer Namen usw.
2 Likes
crzynik
(Nicolas Mowen)
12. November 2025, 12:21 Uhr
5
DasOriginal92:
Ich verwende Qwen 14B ohne nachzudenken und die Offenlegung von nur 53 Entitäten führt dazu, dass es sich sehr unzuverlässig verhält.
Im Moment habe ich 32. Zusätzlich zu dem, was Nathan vorgeschlagen hat, sollten Sie je nachdem, welche Entitäten Sie haben, vielleicht darüber nachdenken, ob alle diese Geräte einzeln angesprochen werden. Sie können in HA viele verschiedene Arten von Gruppen erstellen, bei denen es sich nur um eine einzige Entität handelt, die übergeben werden muss.
2 Likes
DasOriginal92
(Das Original92)
13. November 2025, 13:10 Uhr
6
Danke für den Hinweis. Tatsächlich verwende ich auch qwen3 4B instruct mit seinem Basis-8k-Kontext. Da ich einen A2000 ADA mit 16 GB VRAM verwende, habe ich den Kontext jetzt verdoppelt. Die Ergebnisse sind besser, aber nicht perfekt. Beispielsweise schaltet „Licht im Wohnzimmer einschalten“ manchmal ein Licht in einem anderen Raum ein, oder auch einen Ventilator oder eine Steckdose.
Ich würde gerne ein 7-8B-Modell von Qwen Instruct verwenden. Kennen Sie welche verfügbar?
Ihr Beitrag hat mir übrigens sehr geholfen. Bitte aktualisieren Sie ihn weiter, wenn Sie weitere Fortschritte machen. Tank dich!
DasOriginal92
(Das Original92)
13. November 2025, 13:11 Uhr
7
Guter Vorschlag, ich habe begonnen, alle Lichter für reduzierte Einheiten zu gruppieren
Nathan Cu
(Nathan Curtis)
13. November 2025, 13:15 Uhr
8
Sie können gpt-oss:20b durchaus in eine 16g-Karte unterbringen. Es ist meine wichtigste lokale Schlussfolgerung und ehrlich gesagt ist es VIEL leistungsfähiger als qwen. Sie müssen immer noch den Kontext verwalten, aber… Gibennthesamd Kontextgröße, da war ich erfolgreicher. Auf der Freitagsparty (nein, Sie brauchen nicht das Ganze) spreche ich über den Kontextaufbau, wie, was benötigt wird und warum. Wenn Sie in den Kontext passen und es sich immer noch schlecht verhält, liegt ein Erdungsproblem vor.
Willkommen bei der Wippe. Zu viel Kontext – nicht richtig gemacht, zu wenig Kontext – überhaupt nicht gemacht
1 Gefällt mir
crzynik
(Nicolas Mowen)
13. November 2025, 13:42 Uhr
9
DasOriginal92:
Ich würde gerne ein 7-8B-Modell von Qwen Instruct verwenden. Kennen Sie welche verfügbar?
Benutzt du das Basis-Qwen von Ollama? Diese sind in der Regel recht stark quantisiert, weshalb ich empfehle, zwischen „umarmendem Gesicht“ und einem besseren Modell zu wählen.
huggingface.co
unsloth/Qwen3-4B-Instruct-2507 · Umarmendes Gesicht
Wir sind auf dem Weg, künstliche Intelligenz durch Open Source und Open Science voranzutreiben und zu demokratisieren.
1 Gefällt mir
DasOriginal92
(Das Original92)
13. November 2025
