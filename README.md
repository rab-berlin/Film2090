\# Film2090



Busch Microtronic 2090 goes to Hollywood...


## So startest du den Film

1. In der Weboberfläche des [ESP2090-Studios](https://github.com/rab-berlin/ESP2090) das Userscript [film_color.py](program/film_color.py) laden und starten – es setzt die Farben der LED-Matrix.
2. [film_color.mic](program/film_color.mic) mit **Upload** zum Microtronic übertragen.
3. **LED on** drücken und den Film auf dem Microtronic starten.

Die Reihenfolge ist wichtig: Der Film sichert am Anfang die Farben aus dem Userscript und holt sie nach jeder Szene wieder zurück (Matrix-Befehl ACK 13).
