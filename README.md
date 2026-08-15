# Qwen3.8-27B auf Ryzen AI Max+ 395: Vulkan-, MTP- und OpenCode-Praxistest

Stand: 15. August 2026

> Dieser Text darf gern vollständig oder auszugsweise geteilt werden. Er enthält keine Benutzernamen, privaten Verzeichnisse oder Zugangsdaten.

## Kurzfassung

Ich habe Qwen3.8-27B als lokalen Coding- und Allzweck-Agenten auf einem GMKtec EVO-X2 mit Ryzen AI Max+ 395, Radeon 8060S und 128 GB Unified Memory eingerichtet und systematisch getestet.

Das beste stabile Setup war:

- Unsloth UD-Q5_K_XL
- llama.cpp b10436, Vulkan, vollständiger GPU-Offload
- 128K Kontext
- eingebautes MTP mit 4 Draft-Tokens
- K/V-Cache in q8_0
- OpenCode mit dem nativen Qwen3.8-Jinja-Template

Der reproduzierbare 512-Token-Dauerbenchmark erreichte bei 128K Kontext **16,45 tok/s Generation**, **31,44 tok/s Prompt Processing** und **3,06 Sekunden TTFT**. Bei 64K waren es **16,68 tok/s**. In fortgesetzten Agentenschritten mit KV-/LCP-Reuse lagen neue Promptteile häufig über 200 tok/s; einzelne Generationsschritte erreichten im Mittel 23,04 tok/s und kurze Messfenster etwa 24–26 tok/s.

Das gewünschte dauerhafte Mittel von 20 tok/s wurde also nicht erreicht. Für echte Agentenarbeit ist die Konfiguration trotzdem gut nutzbar. Der größte Nachteil ist der Kaltstart einer neuen OpenCode-Sitzung: Der sehr große Tool-Prompt kann etwa zwei bis drei Minuten Vorverarbeitung benötigen.

## Testsystem

- Mini-PC: GMKtec EVO-X2
- APU: AMD Ryzen AI Max+ 395
- GPU: Radeon 8060S, `gfx1151`
- Speicher: 128 GB Unified Memory
- Betriebssystem: Windows 11
- AMD Software: 26.7.1 WHQL
- Grafiktreiber: 32.0.31035.1003

Die bereits installierte AMD-Version war zum Testzeitpunkt die aktuelle offizielle WHQL-Version. Deshalb wurde weder ein Treiber ersetzt noch eine systemweite ROCm-Installation erzwungen.

## Modell und Runtime

- Modell: Qwen3.8-27B
- Base Model: `Qwen/Qwen3.8-27B`
- Architektur im GGUF: `qwen35`
- Parameter: 27.320.697.856
- Quantisierung: Unsloth UD-Q5_K_XL, dynamische Mischquantisierung
- Modelldatei: `Qwen3.8-27B-UD-Q5_K_XL.gguf`
- Modellgröße: 18,83 GiB
- Vision-Projektor: `mmproj-BF16.gguf`, 0,87 GiB
- Native Kontextlänge: 262.144 Tokens
- Verwendete Kontextlänge: 131.072 Tokens
- Runtime: llama.cpp b10436, Commit `6fed9f6ff`
- Backend: Vulkan mit vollständigem GPU-Offload
- Eingebauter MTP-Predictor: eine Next-N-Predict-Schicht

Die GGUF-Datei enthält verschiedene Tensor-Quantisierungen. Das allgemeine Feld `general.file_type=14` beschreibt deshalb nicht allein den vollständigen Unsloth-Quant. LM Studio erkennt die konkrete Variante als `Q5_K_XL`.

## MTP-Tuning bei Q5 und 64K Kontext

Für alle Varianten wurde derselbe 512-Token-Reasoning-Benchmark verwendet.

| MTP-Draft-Tokens | Generation | Prompt | TTFT | Prozess-RAM | Draft-Annahme |
|---:|---:|---:|---:|---:|---:|
| 2 | 15,54 tok/s | 17,02 tok/s | 5,64 s | 41,07 GiB | 87,77 % |
| 3 | 15,77 tok/s | 31,52 tok/s | 3,06 s | 41,15 GiB | 80,13 % |
| **4** | **16,68 tok/s** | **33,87 tok/s** | **2,85 s** | **41,31 GiB** | **83,80 %** |
| 5 | 15,28 tok/s | 32,17 tok/s | 3,00 s | 41,46 GiB | 82,07 % |

Vier Draft-Tokens waren auf diesem System der schnellste stabile Wert. Mehr Draft-Tokens waren nicht automatisch besser.

## Quant- und Kontextvergleich

| Quantisierung | Kontext | Generation | Prompt | TTFT | Prozess-RAM |
|---|---:|---:|---:|---:|---:|
| UD-Q5_K_XL, MTP 4 | 64K | 16,68 tok/s | 33,87 tok/s | 2,85 s | 41,31 GiB |
| Q6_K, MTP 4 | 64K | 14,24 tok/s | 28,93 tok/s | 3,33 s | 46,10 GiB |
| Q8_0, MTP 4 | 64K | 12,74 tok/s | 31,44 tok/s | 3,06 s | 56,96 GiB |
| **UD-Q5_K_XL, MTP 4** | **128K** | **16,45 tok/s** | **31,44 tok/s** | **3,06 s** | **44,16 GiB** |

Q6_K und Q8_0 waren auf dieser Hardware größer und gleichzeitig langsamer. Der Wechsel von 64K auf 128K kostete Q5 nur rund 1,4 Prozent Generationstempo und etwa 2,85 GiB zusätzliches Working Set. Deshalb blieb 128K als praktischer Standard aktiv.

## Vulkan gegen ROCm

Im identischen API-Test mit Q5, MTP 4 und 64K Kontext ergaben sich:

- llama.cpp b10436 direkt mit Vulkan: **16,68 tok/s**
- LM Studio 0.4.21 mit ROCm-Runtime 2.28.2: **12,04 tok/s**

Der direkte Vulkan-Server war in diesem Test rund **37 Prozent schneller** als LM Studio/ROCm.

Das offizielle direkte ROCm-Binary von llama.cpp b10436 erkannte die Radeon 8060S in dieser Windows-Konfiguration nicht. LM Studio konnte seine ROCm-Runtime dagegen verwenden. Das beweist keinen grundsätzlichen Defekt von ROCm oder des Treibers, sondern beschreibt nur das Verhalten dieser konkreten Builds auf diesem Rechner.

## Tatsächlich verwendete Serverparameter

Die relevante Konfiguration war:

```text
--ctx-size 131072
--parallel 1
--gpu-layers all
--spec-draft-ngl all
--flash-attn on
--cache-type-k q8_0
--cache-type-v q8_0
--batch-size 2048
--ubatch-size 512
--threads 16
--threads-batch 32
--load-mode mmap
--fit off
--jinja
--reasoning-format deepseek
--reasoning-effort high
--reasoning-preserve
--spec-type draft-mtp
--spec-draft-n-max 4
--spec-draft-p-min 0.75
--metrics
--slots
--cache-ram 4096
```

## Chat-Template und OpenCode-Encoding

Verwendet wurde das im GGUF eingebettete Qwen3.8/Unsloth-Jinja-Template mit:

- ChatML-Rollenmarkern `<|im_start|>` und `<|im_end|>`
- `<think>`-Reasoning und getrenntem `reasoning_content`
- nativem XML-Tool-Calling über `<tool_call>`, `<function=...>` und `<parameter=...>`
- `<tool_response>` für Werkzeugergebnisse
- parallelen Tool Calls und verschachtelten Objektargumenten
- Vision-Markern `<|vision_start|><|image_pad|><|vision_end|>`
- erhaltenem Thinking in mehrstufigen Tool-Schleifen

Ein lokaler Capture-Endpunkt zeichnete die von OpenCode tatsächlich gesendeten Parameter auf:

- `reasoning_effort: "high"`; das Template bildet dies intern auf `xhigh` ab
- `temperature: 1.0`
- `top_p: 0.95`
- `top_k: 20`
- `min_p: 0.0`
- `presence_penalty: 0.0`
- `max_tokens: 32000`
- `tool_choice: "auto"`
- 41 Tools im getesteten Hauptrequest

## Agentenleistung in der Praxis

Ein kalter OpenCode-Hauptrequest enthielt ungefähr 13.264 Prompt-Tokens und alle 41 Tool-Definitionen. Die Prompt-Auswertung dauerte laut Serverlog etwa 154,5 Sekunden beziehungsweise 85,83 tok/s. Zusammen mit einem zusätzlichen Titelrequest brauchte eine sehr einfache exakte Antwort in der kalten Sitzung insgesamt ungefähr 189 Sekunden.

Das ist der wichtigste praktische Engpass: Nicht die Generationsgeschwindigkeit, sondern die erstmalige Verarbeitung des großen OpenCode-Werkzeugprompts. In einer laufenden Sitzung helfen Prompt-Cache, Longest-Common-Prefix- und KV-Reuse deutlich. Dort wurden neue Promptabschnitte wiederholt mit mehr als 200 tok/s verarbeitet.

Die höheren Werte von 23–26 tok/s in einzelnen Agentenphasen sind reale Kurzzeit- beziehungsweise Schrittwerte, aber nicht mit dem reproduzierbaren 512-Token-Dauermittel von 16,45–16,68 tok/s gleichzusetzen.

## Bestandene Funktions- und Stabilitätstests

- Coding-Agent: Ein temporäres Mehrdatei-Projekt wurde gelesen, vier fehlschlagende Tests wurden ausgeführt, die Ursache diagnostiziert, eine Validierungsdatei ergänzt und zwei Produktionsdateien geändert. Danach bestanden 4/4 Tests; die Testdateien blieben unverändert.
- Tool Calling: Zwei parallele Calls, verschachtelte JSON-Objekte und Arrays sowie die Fortsetzung nach einem absichtlich fehlgeschlagenen Tool Call funktionierten.
- Web: Websuche und WebFetch waren im echten OpenCode-Request enthalten; eine aktuelle Information wurde von der offiziellen Python-Website abgerufen.
- Browser/MCP: Playwright MCP 0.0.79 navigierte zu einer lokalen Testseite, las berechnete DOM-Stile und erzeugte einen Screenshot.
- Vision: Das Modell erkannte im Screenshot Überschrift und unsichtbaren Statustext korrekt. Vorder- und Hintergrundfarbe des Problems waren beide `#18D38A`.
- Lange Agentenschleife: Keine Wiederholungsschleifen oder auffälligen Token-Artefakte beobachtet.

## Was noch verbessert oder untersucht werden könnte

Besonders interessant wären reproduzierbare Vergleiche zu diesen Punkten:

1. Warum erkennt das offizielle direkte llama.cpp-ROCm-Binary die `gfx1151`-GPU unter Windows nicht, während die LM-Studio-ROCm-Runtime funktioniert?
2. Verbessern neuere llama.cpp-Builds die Vulkan-Shader, MTP-Unterstützung oder ROCm-Geräteerkennung auf Ryzen AI Max+?
3. Sind andere Werte für `--spec-draft-p-min` bei MTP 4 schneller oder stabiler?
4. Wie stark lässt sich der kalte OpenCode-Start durch kleinere Tool-Schemata, einen separaten Titel-Agenten oder konsequentes Prompt-Caching verkürzen?
5. Liefern andere dynamische Q5-GGUFs auf derselben Hardware ein besseres Qualitäts-/Tempo-Verhältnis?
6. Wie verhalten sich tatsächlich sehr lange Prompts nahe 128K? Der hier dokumentierte Test konfigurierte 128K, füllte das Fenster aber nicht vollständig aus.
7. Bringt der Verzicht auf den Vision-Projektor bei reinem Textbetrieb messbare Vorteile bei Startzeit oder Speicherbedarf?

Rückmeldungen mit exakten Buildnummern, Startparametern und reproduzierbaren Messbedingungen wären besonders hilfreich. Die Werte hier stammen von einem einzigen Windows-System und sollten nicht ungeprüft auf Linux, andere Treiberversionen oder andere Ryzen-AI-Max-Geräte übertragen werden.

## Fazit

Qwen3.8-27B UD-Q5_K_XL ist auf dem Ryzen AI Max+ 395 als lokaler allgemeiner OpenCode-Agent stabil und brauchbar. Vulkan war in dieser Konfiguration klar schneller als die getestete LM-Studio-ROCm-Runtime. MTP 4 war der beste Wert, 128K Kontext verursachte nur einen kleinen Geschwindigkeitsverlust, und die Agenten-, Tool-, Browser- und Vision-Tests funktionierten.

Die ehrliche Einschränkung: **16,45 tok/s im langen Benchmark sind keine dauerhaften 20 tok/s.** Dank Cache-Reuse fühlt sich eine laufende Agentensitzung deutlich schneller an, während der kalte OpenCode-Werkzeugprompt weiterhin zwei bis drei Minuten kosten kann.

## Primärquellen

- [Offizielles Qwen3.8-27B-Modell](https://huggingface.co/Qwen/Qwen3.8-27B)
- [Unsloth Qwen3.8-27B GGUF](https://huggingface.co/unsloth/Qwen3.8-27B-GGUF)
- [llama.cpp b10436](https://github.com/ggml-org/llama.cpp/releases/tag/b10436)
- [LM Studio Changelog](https://lmstudio.ai/changelog/lmstudio-v0.4.14)
- [LM Studio CLI-Dokumentation](https://lmstudio.ai/docs/cli)
- [OpenCode Provider](https://opencode.ai/docs/providers)
- [OpenCode Tools](https://opencode.ai/docs/tools)
- [AMD Software 26.7.1 Release Notes](https://www.amd.com/de/resources/support-articles/release-notes/RN-RAD-WIN-26-7-1.html)
- [AMD ROCm-Kompatibilität unter Windows](https://rocm.docs.amd.com/projects/radeon-ryzen/en/latest/docs/compatibility/compatibilityryz/windows/windows_compatibility.html)

