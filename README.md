# Film2090

Busch Microtronic 2090 goes to Hollywood...

Ein kleiner Pixelfilm für die 8x8-LED-Matrix des [ESP2090 Studios](https://github.com/rab-berlin/ESP2090) mit farbigen Pixeln. Er läuft in einer Endlosschleife, ein Durchlauf dauert knapp 16 s (gemessen im [Emulator2090](https://github.com/rab-berlin/Emulator2090)).

- 256 Programmschritte (00-FF), alle belegt
- Dateien: [film_color.mic](program/film_color.mic) und das Userscript [film_color.py](program/film_color.py)

## So startest du den Film

1. In der Weboberfläche des [ESP2090-Studios](https://github.com/rab-berlin/ESP2090) das Userscript [film_color.py](program/film_color.py) laden und starten. Es setzt die Farben der LED-Matrix.
2. [film_color.mic](program/film_color.mic) mit **Upload** zum Microtronic übertragen.
3. **LED on** drücken und den Film auf dem Microtronic starten.

Die Reihenfolge ist wichtig: Der Film legt am Anfang die Farben aus dem Userscript auf den Stapel (push) und holt sie nach jeder Szene wieder zurück (peek, Matrix-Befehl ACK 13).

Im Emulator2090 läuft der Film ohne das Userscript, dann sind alle Pixel rot (Standardfarbe).

## Das Userscript

`film_color.py` besteht aus einer Liste mit 64 Farbnamen, einer je LED (türkis, orange, gelb, rot, blau, weiß), und einem einzigen Aufruf `setColors(colors)`. Damit steht die Grundfarbe jedes Pixels fest, bevor der Film beginnt. Zusätzlich wählt der Film selbst mit ACK 14 Zeichenfarben (magenta, türkis, grün, gelb, orange) für gezeichnete Punkte und Flächen.

## Technik

**Schreibweise in diesem README:** Ein Register heißt `R0` bis `RF` (Speicher: `M0` bis `MF`), ein Wert beginnt mit `#`, zum Beispiel `#3` oder `#E`. `RE` ist also ein Register, `#E` die Zahl vierzehn. ACK-Nummern schreibe ich dezimal als „ACK 14“, der Film sendet sie als Wert, hier `#E`. In den Programmzeilen stehen `R` für Register und `#` für Werte.

Der Film mischt zwei Arten, die Matrix anzusprechen:

- **Ganze Bilder (Blockmodus, ACK 1):** Das Programm baut das Bild in den Registern auf und schickt es mit dem Unterprogramm `bild`. `R0`-`R7` des Bildes liegen dazu in den Speicherregistern `M0`-`M7` und kommen per `EXRL` ins Fenster, `R8`-`RF` liegen direkt in den Registern `R8`-`RF`.
- **Zeichenbefehle:** Pixel und Flächen, ohne das Bild neu zu schicken: ACK 4 (Cursor 1 und Punkt), ACK 6 (gefülltes Rechteck zwischen Cursor 1 und 2), ACK 14 (Farbe wählen) und ACK 13 (Stapel). Ein Befehl ist: auf REQ warten, ACK, zwei Nibbles (x, y), Ausgänge auf `#0`.

Die Zeichenbefehle haben die Zeichenart ohne +`#8`. Dann ändern sie nur die Farbmatrix im ESP, sichtbar wird das erst mit dem nächsten Bild.

```
bild: DIN  R0         ; warten auf REQ (Eingang 1)
      ANDI #1,R0
      BRZ  bild
      DOT  R0         ; ACK 1  (R0 ist jetzt #1)
      EXRL            ; der eine Befehl vor den Daten: Bild R0-R7 ins Fenster
      DOT  R0         ; Bildregister 0
      DOT  R1         ; Bildregister 1
      ...             ; so weiter bis
      DOT  RF         ; Bildregister F
      EXRL
      DOT  R2         ; Ausgänge auf #0 (R2 ist hier immer #0)
      RET
```

### Farbmatrix auf dem Stapel

Eine Szene darf die Farbmatrix verändern, die nächste soll aber wieder mit den Farben aus dem Userscript beginnen. Dafür sichert der Film sie einmal am Start auf dem Stapel (ACK 13, Wert `#1` = push). Nach jeder Szene holt er sie mit ACK 13, Wert `#9` = peek zurück. Peek lässt den Stapel unverändert, die Farben stehen deshalb auch im nächsten Durchlauf wieder bereit.

### Nur eine Rücksprungadresse

Der 2090 merkt sich nur eine Rücksprungadresse, deshalb ruft kein Unterprogramm ein anderes mit `CALL` auf. Die Befehle `punkt`, `rechteck`, `farbe`, `sichern` und `zurueck` setzen nur ACK-Code und Nibble und springen dann mit `GOTO` zum gemeinsamen `cmd`. Dessen `RET` kehrt direkt zum Aufrufer zurück (Tail-Call).

```
punkt:    MOVI #4,R1  ; R1 = ACK-Code #4: Cursor 1 / Punkt
          GOTO cmd    ; kein CALL: das RET von cmd kehrt zum Aufrufer von punkt zurück
rechteck: MOVI #6,R1  ; R1 = #6: Rechteck Cursor 1 - Cursor 2
          GOTO cmd
...
cmd:  DIN  R0         ; warten auf REQ
      ANDI #1,R0
      BRZ  cmd
      DOT  R1         ; ACK
      DOT  R2         ; Nibble 1 (x bzw. Wert)
      DOT  R3         ; Nibble 2 (y)
      MOVI #0,R2      ; R2 = #0 für die nächste Kette
      DOT  R2         ; Ausgänge auf #0
      RET
```

### Eine Kette zählt die Farbnummer

Die Farben werden als Nummer der Palette gesendet. Eine Folge von `ADDI #1,R2` dient als Einsprungtabelle: Wer bei `f_rot` einspringt, bekommt `#1`, bei `f_orange` `#2`, bei `f_gelb` `#3`, bei `f_gruen` `#4`, bei `f_tuerkis` `#5`, und `f_magenta` addiert zuerst `#3` und kommt so auf `#8`. Die Kette fällt danach in `farbe`, das den Code 14 (`#E`) setzt. Dieselbe Kette dient für `zurueck` (`#8` + `#1` = `#9`, peek) und `sichern` (`#1`, push) mit ACK 13 (`#D`). Nach jedem Befehl setzt `cmd` `R2` wieder auf `#0`. Das ist zugleich der Wert für das Ausgangssignal 0 am Ende und der Ausgangspunkt für die nächste Kette.

```
f_magenta: ADDI #3,R2   ; R2 war #0: jetzt #3, dazu fünf mal #1: #8 = magenta
f_tuerkis: ADDI #1,R2   ; Einsprung hier: fünf mal #1 = #5 = türkis
f_gruen:   ADDI #1,R2   ; Einsprung hier: #4 = grün
f_gelb:    ADDI #1,R2   ; #3 = gelb
f_orange:  ADDI #1,R2   ; #2 = orange
f_rot:     ADDI #1,R2   ; Einsprung hier: #1 = rot
farbe:     MOVI #E,R1   ; R1 = #E: ACK 14, Farbe wählen
           GOTO cmd
zurueck:   ADDI #8,R2   ; R2 = #8 ...
sichern:   ADDI #1,R2   ; ... #9 (peek) bzw. bei Einsprung hier #1 (push)
           MOVI #D,R1   ; R1 = #D: ACK 13, Stapel
cmd:       ...          ; läuft in cmd hinein
```

### Register

| Register | Bedeutung |
|---|---|
| `R0` | Warten auf REQ (`DIN`), sonst `#0` |
| `R1` | ACK-Code |
| `R2` | x bzw. Datennibble |
| `R3` | y |
| `R8`-`RF` | Bild, Register `R8`-`RF` |

Die Bilder entstehen mit `MOVI`-Folgen direkt in den Registern: Ein Register ist eine halbe Bildzeile, und der Film ist damit großenteils Daten. Deshalb füllt er die 256 Schritte vollständig.

```
EXRL            ; Fenster auf M0-M7 öffnen
MOVI #F,R0      ; Bildregister 0 (liegt im Fenster, M0) = #F: alle 4 LEDs dieser halben Zeile an
EXRL            ; Fenster schließen
MOVI #6,R8      ; R8 = #6: Bildregister 8 (liegt direkt in R8)
```
