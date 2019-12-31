# E.CON.TROL.CHECK.DEMO
Demo C#-Anwendung für die Implementierung einer (zusätzlichen) Prüfung bzw. den Zugriff auf aufgenommenen Bilder bei E.CON.TROL.

#### Inhaltsverzechnis
* [Beschreibung](#beschreibung)
* [Systemarchitektur](#systemarchitektur)
* [NetMQ](#netmq)
* [Nachrichtenformat / Messageprotocol](messageprotocol)
* [Nachrichten von E.CON.TROL.CORE zu E.CON.TROL.CHECK.X](#core2check)
* [Nachrichten von E.CON.TROL.CHECK.X zu E.CON.TROL.CORE](#check2core)
* [Nachrichten zur Bildübertragung](#imagemessage)

<a name="beschreibung"/>

## Beschreibung
E.CON.TROL besitzt (ab Version 11.0) eine Schnittstelle über die sämtliche aufgenommenen (Roh-)Bilder der an E.CON.TROL angeschlossenen (Flächenkameras) abgerufen werden können.
Diese Bilder können z.B. genutzt werden, um weitere Prüfungen außerhalb von E.CON.TROL zu realisieren. Das Ergebnis dieser externen Prüfungen kann ebenfalls an den E.CON.TROL.CORE zurückgemeldet werden.

<a name="systemarchitektur"/>

## Systemarchitektur
E.CON.TROL ist in verschiedene (separate) Applikationen eingeteilt, jede dieser Applikationen übernimmt eine dedizierte Aufgabenstellung.

### E.CON.TROL.SPS
Pro Insektionstunnel ist eine SPS-Steuerung vorhanden. An diese SPS-Steuerung sind die Lichtschranken angeschlossen, die sich am Einlaufband des Tunnels befinden. Über diese Lichtschranken wird erkannt, wenn ein Behälter in die Anlage einläuft. Weiterhin wird über diese Lichtschranken bestimmt um welchen Behältertyp (A11, A15, A18, A22) es sich handelt. 
Für jeden eingelaufenenen Behälter wird von der SPS eine Box-ID generiert (Zähler). Diese Box-ID wird an den E.CON.TROL.CORE gemeldet und dient zur weiteren Identifikation des Behälters.
Der Behälter bzw. die Box-ID wird von der SPS bis zur Ausschleusung (Pusher) mitgeführt. Erreicht der Behälter die Lichtschranke am Pusher, wird geprüft, ob für die entsprechende Box-ID eine Ausschleusemeldung vorliegt, in diesem Fall wird der Pusher durch die SPS angesteuert.

### E.CON.TROL.CORE
Dies ist die Kernapplikation, welche die Ergebnisse aller Prüfungen "einsammelt" und aufgrund aller dieser Prüfergebnisse entscheidet, ob ein Behälter ausgeschleust werden soll.
E.CON.TROL.CORE übernimmt die Kommunikation mit der unterlagertetn SPS-Steruerung und wird von dieser benachrichtigt, wenn ein neuer Behälter in die Anlage einläuft. Diese Information gibt E.CON.TROL.CORE an alle angeschlossenen mit der entsprechenden Behälter bzw. BoxID weiter.

### E.CON.TROL.CHECK.BOXBACK
Die (Flächen-)Kamera für diese Prüfung sitzt über dem Einlaufbereich (CameraNumber 1) und nimmt von jedem Behälter zwei Bilder auf (Vorderseite innen und Rückseite außen). 

### E.CON.TROL.CHECK.BOXFRONT
Die (Flächen-)Kamera für diese Prüfung sitzt über dem Auslaufbereich (CameraNumber 2) und nimmt von jedem Behälter zwei Bilder auf (Vorderseite außen und Rückseite innen).

### E.CON.TROL.CHECK.BOXBOTTOM
Die (Flächen-)Kamera für diese Prüfung nimmt von jedem Behälter ein Bild von oben auf, welches vor allem die Behälterinnenseite bzw. den Behälterboden zeigt (CameraNumber 5).

### E.CON.TROL.CHECK.BARCODE
Die Prüfung der BAR- bzw. Matrxcodes erfolgt über zwei (Flächen-)Kameras, die rechts und links neben dem Förderband angebracht sind (CameraNumber 6 und CameraNumber 7).

### E.CON.TROL.CHECK.BOXSIDE
Neben den oben genannten Prüfungen, die mit Bildern von Flächenkameras arbeiten, sind noch zwei Prüfungen vorhanden die die Behälterseiten mit einem 3D-Verfahren inspizieren. Die Bilder der 3D-Kameras werden aktuell nicht über eine Schnittstelle nach außen zur Verfügung gestellt.

<a name="netmq"/>

## NetMQ
Die im folgenden Beschriebenen Schnittstellen sind technisch auf Basis von NetMQ (https://github.com/zeromq/netmq) realisiert. Es wird hier das [Push-Pull-Pattern](https://netmq.readthedocs.io/en/latest/push-pull/) für das Zurückmelden der Status- und Ergebnismeldungen von den Prüfungen an E.CON.TROL.CORE sowie das [Pub-Sub-Pattern](https://netmq.readthedocs.io/en/latest/pub-sub/) für das Übertragen der Bilder und das Starten der Auswertungen seitens E.CON.TROL.CORE verwendet.

<a name="messageprotocol"/>

## Nachrichtenformat / Messageprotocol
Für die über [NetMQ](https://github.com/zeromq/netmq) übertragenen Nachrichten bzw. Meldungen, wird ein proprietäres Messageprotocol verwendet:

| Byte          | Name            | Beschreibung                                                      |
| ------------- |-----------------|-------------------------------------------------------------------|
| Byte 0-3      | HeaderLength    | Gibt die Länge des darauffolgenden Headers an                     |
| Byte 4-n      | Header          | BSON codiertes Objekt bzw. Informationen über den Nachrichtentyp  |
| Byte n-...    | User-Data       | Optionale Nutzdaten in Abhängigkeit des Nachrichtentyps           |

Das Encoding/Decoding der Nachrichten erfolgt über Funktionalität die in dem Nuget-Paket [NetMq.Messages](https://github.com/maa-es/e.con.trol.check.demo/blob/master/E.CON.TROL.NuGetPackages/NetMq.Messages.1.0.1.nupkg) gekapselt ist. 
Bei Bedarf kann der Source-Code ebenfalls bereitgestellt werden.

<a name="core2check"/>

## Nachrichten von E.CON.TROL.CORE zu E.CON.TROL.CHECK.X
Nachrichten von E.CON.TROL werden über einen Publisher-Socket verschickt. Die Prüfungen (E.CON.TROL.CHECK.X) verbinden sich mit einen Subscriber-Socket zum Port: 55555 (z.B. "tcp://10.134.3.154:55555") und registrieren sich für alle Meldungen mit ihrem jeweiligen Prüfungsnamen (z.B. Topic == E.CON.TROL.CHECK.BOXBOTTOM). Folgende Meldungen werden von E.CON.TROL.CORE zu E.CON.TROL.CHECK.X versendet:

| Name            | Beschreibung                                                        |
|-----------------|---------------------------------------------------------------------|
| `StateMessage`  | Diese Meldung wird zum Übertagen von Statusinformationen verwendet  |
| `ProcessStartMessage` | Diese Meldung wird von E.CON.TROL.CORE an die Prüfungen gesendet, wenn mit der Auswertung eines Behälters begonnen werden soll |
| `ProcessCancelMessage` | Diese Meldung wird von E.CON.TROL.CORE an die Prüfungen gesendet, wenn die Auswertung eines Behälters abgebrochen werden soll |

<a name="check2core"/>

## Nachrichten von E.CON.TROL.CHECK.X zu E.CON.TROL.CORE
Die Nachrichten bzw. Rückmeldungen der Prüfungen an den E.CON.TROL.CORE werden über einen PushSocket versendet. Die Prüfungen (E.CON.TROL.CHECK.X) instanziieren jeweils einen PushSocket und verbinden diesen zum Port 55556 (z.B. "tcp://10.134.3.154:55556"). Folgende Meldungen werden von den Prüfungen (E.CON.TROL.CHECK.X) an den E.CON.TROL.CORE versendet:
| Name            | Beschreibung                                                        |
|-----------------|---------------------------------------------------------------------|
| `StateMessage`  | Diese Meldung wird zum Übertagen von Statusinformationen verwendet  |
| `ProcessFinishedMessage` | Diese Meldung wird von den Prüfungen an E.CON.TROL.CORE gesendet, um das Prüfergebnis des aktuell geprüften Behälters zu melden |

<a name="imagemessage"/>

## Nachrichten zur Bildübertragung
Die aufgenommenen Bilder der Flächenkameras werden ebenfalls über einen Publisher-Socket verteilt. Um Bilder zu empfangen ist es erforderlich, einen Subscriber-Socket zu instanziieren und diesen z.B. zu folgendem Endpunkt zu verbinden: "tcp://10.134.3.154:55562". 
Die letzte Ziffer der Portnummer gibt hier die Nummer der Kamera an (in diesem Fall CameraNumber 2).
| Name            | Beschreibung                                                        |
|-----------------|---------------------------------------------------------------------|
| `ImageMessage`  | Diese Nachricht wird zur Übertragung von Bilddaten verwendet        |

