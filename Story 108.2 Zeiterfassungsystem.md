
```python
import atexit                       # Lädt das atexit‑Modul, um beim Programmende automatisch Aufräumfunktionen auszuführen (z. B. GPIO cleanup).
import RPi.GPIO as GPIO             # Importiert das GPIO‑Modul des Raspberry Pi und gibt ihm den Kurznamen GPIO.
from mfrc522 import SimpleMFRC522   # Importiert die vereinfachte Klasse für den MFRC522‑RFID‑Reader.
import mariadb                      # MariaDB‑Connector für Python, um mit der Datenbank zu kommunizieren.
from datetime import datetime       # Lädt datetime, um Datum/Zeit zu lesen und zu formatieren.

# GPIO-Warnungen unterdrücken und Cleanup am Ende sicherstellen
GPIO.setwarnings(False)             # Deaktiviert GPIO‑Warnungen (z. B. wegen bereits gesetzter Pins) – erhöht Lesbarkeit der Konsole.
atexit.register(GPIO.cleanup)       # Registriert GPIO.cleanup() als Aufräumroutine, die beim Programmende automatisch ausgeführt wird.

# -------------------------------
# Klasse: Datenbank
# -------------------------------

class Datenbank:                    # Definiert eine Klasse, die alle DB‑Operationen kapselt (Verbindung, Queries, Schließen).
    def __init__(self, host="localhost", user="admin", password="raspi", database="timedb"):
        self.host = host            # Speichert den DB‑Host (Standard: lokal).
        self.user = user            # Speichert den DB‑Benutzernamen.
        self.password = password    # Speichert das Passwort – Hinweis: In Produktion sicherer speichern (z. B. .env).
        self.database = database    # Speichert den Datenbanknamen.
        self.connection = None      # Platzhalter für die DB‑Verbindung (wird in verbinden() gesetzt).

    def verbinden(self):
        try:
            self.connection = mariadb.connect(  # Versucht, eine DB‑Verbindung aufzubauen …
                host=self.host,
                user=self.user,
                password=self.password,
                database=self.database
            )
            print("✅ Verbindung zur Datenbank erfolgreich.")  # Infoausgabe bei Erfolg.
        except mariadb.Error as e:                                 # Fängt DB‑Fehler ab …
            print(f"❌ Fehler bei der Verbindung: {e}")           # … und meldet sie in der Konsole.
            self.connection = None                                  # Stellt sicher, dass connection nicht auf einer fehlerhaften Instanz bleibt.

    def ausfuehren(self, query, params=None, fetchone=False, fetchall=False, commit=False):
        if not self.connection:                                      # Prüft, ob es eine aktive Verbindung gibt …
            raise RuntimeError("❌ Keine aktive Datenbankverbindung. Bitte zuerst verbinden().")  # … sonst klare Fehlermeldung.

        cursor = self.connection.cursor()                            # Erzeugt einen Cursor zum Ausführen des SQL‑Statements.
        try:
            cursor.execute(query, params or ())                      # Führt das Statement mit Parametern aus (params oder leeres Tupel).

            if fetchone:                                            # Wenn genau ein Datensatz erwartet wird …
                result = cursor.fetchone()                           # … hole einen Datensatz.
            elif fetchall:                                          # Wenn mehrere Datensätze erwartet werden …
                result = cursor.fetchall()                           # … hole alle Datensätze.
            else:
                result = cursor.rowcount                             # Sonst: Anzahl betroffener Zeilen (für INSERT/UPDATE/DELETE nützlich).

            if commit:                                              # Soll die Transaktion festgeschrieben werden?
                self.connection.commit()                             # Ja → commit, damit Änderungen dauerhaft sind.

            return result                                            # Gibt das Ergebnis an den Aufrufer zurück.

        except mariadb.Error as e:                                   # Bei DB‑Fehlern …
            try:
                self.connection.rollback()                           # … versuche, Änderungen zurückzurollen (DB‑Zustand konsistent halten).
            except Exception:
                pass                                                 # Wenn rollback fehlschlägt, nicht erneut crashen.
            raise e                                                  # Fehler weiterwerfen, damit Aufrufer reagieren kann.
        finally:
            cursor.close()                                           # Cursor in jedem Fall schließen (Ressourcen freigeben).

    def schliessen(self):
        if self.connection:                                          # Wenn es noch eine Verbindung gibt …
            self.connection.close()                                  # … Verbindung ordentlich schließen.
            print("🔒 Verbindung geschlossen.")                    # Infoausgabe.

# -------------------------------
# Klasse: Schueler
# -------------------------------

class Schueler:                                                        # Modellklasse für Schülerdaten.
    def __init__(self, name: str, klasse: str, rfid_uid: str):         # Initialisiert Name, Klasse und Karten‑UID.
        self.name = name
        self.klasse = klasse
        self.rfid_uid = rfid_uid

    def speichern(self, db: Datenbank):                                # Persistiert das Objekt in der DB.
        try:
            db.ausfuehren(
                "INSERT INTO schueler (name, klasse, rfid_uid) VALUES (?, ?, ?)",  # Parametrisiertes INSERT verhindert SQL‑Injection.
                (self.name, self.klasse, self.rfid_uid),
                commit=True
            )
            print(f"✅ Schüler {self.name} wurde gespeichert.")     # Erfolgsmeldung.
        except mariadb.Error as e:
            # 1062 = Duplicate entry
            if getattr(e, "errno", None) == 1062:                    # Prüft speziellen Fehlercode 1062 (Duplikat, z. B. doppelte rfid_uid).
                print("⚠️ Diese Karte ist schon registriert.")     # Nutzerfreundliche Meldung.
            else:
                print(f"❌ Fehler beim Speichern: {e}")             # Allgemeiner DB‑Fehler.

# -------------------------------
# Klasse: Zeiterfassungssystem
# -------------------------------

class Zeiterfassungssystem:                                            # Kapselt den Ablauf der Zeiterfassung (Lesen, Suchen, Buchen, Status).
    def __init__(self, db: Datenbank):
        self.db = db                                                   # Merkt sich die DB‑Instanz für spätere Abfragen.
        self.reader = SimpleMFRC522()                                  # Initialisiert den RFID‑Reader einmalig (teuer, daher nicht pro Lesevorgang).

    def rfid_lesen(self) -> str:
        print("Halte eine Karte an den Leser...")                    # Instruktion für den Benutzer.
        try:
            rfid_id, _text = self.reader.read()                        # Liest UID und evtl. Text vom Chip; hier wird nur die UID genutzt.
            return str(rfid_id).strip()                                # Konvertiert zur String‑UID und trimmt Whitespace.
        except KeyboardInterrupt:
            raise                                                      # Bei STRG+C weiterreichen, damit das Hauptprogramm korrekt abbricht.
        except Exception as e:
            print(f"❌ Fehler beim Lesen der RFID: {e}")             # Andere Lesefehler melden …
            return ""                                                # … und leeren String liefern, damit Aufrufer sauber reagieren kann.
        # KEIN GPIO.cleanup() hier!                                    # Wichtig: Cleanup global durch atexit, sonst würde der Reader mittendrin deaktiviert.

    def schueler_suchen(self, rfid_uid: str):
        result = self.db.ausfuehren(
            "SELECT id FROM schueler WHERE rfid_uid = ?",            # Sucht die Schüler‑ID anhand der Karten‑UID.
            (rfid_uid,),
            fetchone=True
        )
        return result[0] if result else None                           # Gibt die ID oder None zurück (wenn nicht gefunden).

    def checkin(self, schueler_id: int):
        datum = datetime.now().strftime('%Y-%m-%d')                    # Heutiges Datum als YYYY‑MM‑DD (für Tages‑Eindeutigkeit).
        zeit_kommen = datetime.now().strftime('%H:%M:%S')              # Check‑In‑Uhrzeit als HH:MM:SS.
        try:
            self.db.ausfuehren(
                "INSERT INTO zeiterfassung (schueler_id, datum, zeit_kommen) VALUES (?, ?, ?)",
                (schueler_id, datum, zeit_kommen),
                commit=True
            )
            print("✅ Check-In erfolgreich.")                        # Erfolgsmeldung.
        except mariadb.Error as e:
            if getattr(e, "errno", None) == 1062:                    # Bei Unique‑Constraint (z. B. (schueler_id, datum)) → bereits eingecheckt.
                print("⚠️ Heute schon eingecheckt.")                # Benutzerhinweis.
            else:
                print(f"❌ Fehler beim Check-In: {e}")               # Allgemeiner DB‑Fehler.

    def checkout(self, schueler_id: int):
        datum = datetime.now().strftime('%Y-%m-%d')                    # Heutiges Datum erneut ermitteln (muss zum Check‑In passen).
        zeit_gehen = datetime.now().strftime('%H:%M:%S')               # Check‑Out‑Zeitstempel.
        try:
            rc = self.db.ausfuehren(
                "UPDATE zeiterfassung SET zeit_gehen = ? WHERE schueler_id = ? AND datum = ? AND zeit_gehen IS NULL",
                (zeit_gehen, schueler_id, datum),
                commit=True
            )
            if rc and rc > 0:                                         # rowcount > 0 → Es wurde eine offene Buchung geschlossen.
                print("✅ Check-Out erfolgreich.")                  # Erfolg.
            else:
                print("ℹ️ Kein offener Check-In für heute gefunden (evtl. schon ausgecheckt oder nie eingecheckt).")
        except mariadb.Error as e:
            print(f"❌ Fehler beim Check-Out: {e}")                  # DB‑Fehler beim Update melden.

    def status_abfragen(self, schueler_id: int):
        datum = datetime.now().strftime('%Y-%m-%d')                    # Heutiges Datum bestimmen.
        result = self.db.ausfuehren(
            "SELECT zeit_kommen, zeit_gehen FROM zeiterfassung WHERE schueler_id = ? AND datum = ?",
            (schueler_id, datum),
            fetchone=True
        )
        if result:                                                     # Falls es einen Eintrag für heute gibt …
            zk, zg = result                                           # Entpacke Kommen‑ und Gehen‑Zeit.
            if zg is None:                                            # Noch kein Check‑Out?
                print(f"📊 Heute: Gekommen um {zk}, noch nicht ausgecheckt.")
            else:
                print(f"📊 Heute: Gekommen um {zk}, Gegangen um {zg}")
        else:
            print("ℹ️ Keine Einträge für heute.")                    # Kein Datensatz für heute vorhanden.

# -------------------------------
# Hauptprogramm
# -------------------------------

if __name__ == "__main__":                                           # Stellt sicher, dass dieser Block nur läuft, wenn das Skript direkt gestartet wird.
    db = Datenbank("localhost", "admin", "raspi", "timedb")      # Erzeugt eine DB‑Instanz mit Verbindungsdaten (derzeit hartkodiert).
    db.verbinden()                                                     # Baut die DB‑Verbindung auf.
    if not db.connection:                                              # Prüft, ob Verbindung steht …
        raise SystemExit(1)                                            # … sonst beendet das Programm mit Fehlercode 1.

    system = Zeiterfassungssystem(db)                                  # Initialisiert das Zeiterfassungssystem mit DB und RFID‑Reader.

    try:
        while True:                                                    # Endlosschleife für interaktives Menü.
            print("\n--- Zeiterfassung Menü ---")               # Menü‑Header.
            print("1. Schüler hinzufügen")                           # Menüpunkt 1.
            print("2. Check-In")                                     # Menüpunkt 2.
            print("3. Check-Out")                                    # Menüpunkt 3.
            print("4. Status abfragen")                              # Menüpunkt 4.
            print("5. Beenden")                                      # Menüpunkt 5.

            auswahl = input("Option wählen: ").strip()               # Liest Nutzereingabe und trimmt Leerzeichen.

            if auswahl == "1":                                       # Fall 1: Schüler registrieren …
                name = input("Name: ").strip()                       # Name erfassen.
                klasse = input("Klasse: ").strip()                   # Klasse erfassen.
                rfid_uid = system.rfid_lesen()                         # UID live von Karte lesen.
                if not rfid_uid:                                       # Falls Lesen fehlschlug …
                    print("❌ Konnte keine Karte lesen.")            # … Benutzer informieren …
                    continue                                           # … und zum Menüanfang springen.
                neuer_schueler = Schueler(name, klasse, rfid_uid)     # Objekt mit Eingaben erzeugen.
                neuer_schueler.speichern(db)                           # In DB persistieren.

            elif auswahl == "2":                                     # Fall 2: Check‑In …
                rfid_uid = system.rfid_lesen()                         # UID lesen.
                if not rfid_uid:
                    print("❌ Konnte keine Karte lesen.")
                    continue
                sid = system.schueler_suchen(rfid_uid)                 # Schüler anhand UID suchen.
                if sid:                                                # Falls vorhanden …
                    system.checkin(sid)                                # … Check‑In buchen.
                else:
                    print("❌ Schüler nicht gefunden.")              # Sonst Fehlermeldung.

            elif auswahl == "3":                                     # Fall 3: Check‑Out …
                rfid_uid = system.rfid_lesen()
                if not rfid_uid:
                    print("❌ Konnte keine Karte lesen.")
                    continue
                sid = system.schueler_suchen(rfid_uid)
                if sid:
                    system.checkout(sid)                               # Offene Buchung schließen.
                else:
                    print("❌ Schüler nicht gefunden.")

            elif auswahl == "4":                                     # Fall 4: Tagesstatus anzeigen …
                rfid_uid = system.rfid_lesen()
                if not rfid_uid:
                    print("❌ Konnte keine Karte lesen.")
                    continue
                sid = system.schueler_suchen(rfid_uid)
                if sid:
                    system.status_abfragen(sid)                        # Status (heute) ausgeben.
                else:
                    print("❌ Schüler nicht gefunden.")

            elif auswahl == "5":                                     # Fall 5: Programmende …
                print("Programm beendet.")                            # Abschiedstext.
                break                                                  # Endlosschleife verlassen.
            else:
                print("⚠️ Ungültige Auswahl.")                       # Falsche Menüeingabe behandeln.
    except KeyboardInterrupt:                                          # STRG+C im Menü …
        print("\n⏹️ Abgebrochen per Tastatur.")                # … sauber melden.
    finally:
        db.schliessen()                                               # Datenbankverbindung in jedem Fall schließen.
```
