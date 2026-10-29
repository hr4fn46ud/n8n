# Mastodon → ATProto Crossposter für n8n

Importierbarer Workflow für öffentliche Mastodon-Originalbeiträge und ein frei konfigurierbares ATProto-Konto. Enthält keine Zugangsdaten oder vorkonfigurierten persönlichen Accounts. Der Workflow ist beim Import deaktiviert.

## Funktionen

- Mastodon-Polling alle zwei Minuten; ausschließlich öffentliche Originalbeiträge, keine Antworten oder Boosts.
- HTML-zu-Text-Konvertierung mit Links und Inhaltswarnungen.
- Threads mit maximal 300 Graphemen je Teil, einschließlich Nummerierung; ATProto-Verknüpfung über root und parent.
- Link- und Hashtag-Facets mit UTF-8-Bytepositionen.
- Bis zu vier Bilder mit Alt-Text; JPEG-Komprimierung und Prüfung auf unter 1 MB pro Bild.
- Dynamische Handle-, DID- und PDS-Auflösung einschließlich Rückprüfung des Handles im DID-Dokument.
- Persistenter Verarbeitungszustand und deterministische Record-Schlüssel für wiederholbare Veröffentlichungen.
- HTTP-Zeitlimits und begrenzte Wiederholungsversuche.

## Voraussetzungen

Eine selbst gehostete n8n-Installation mit integrierten HTTP Request-, Code- und Edit Image-Nodes sowie GraphicsMagick. Die ursprüngliche Vorlage wurde mit n8n 2.43.x praktisch erprobt. Diese neutrale Ausgabe wurde auf JSON-Struktur, Code-Syntax und Verbindungen geprüft; ein erneuter Live-Import wurde nicht durchgeführt.

## 1. Importieren und konfigurieren

`mastodon-to-atproto.n8n.json` über **Import from File** importieren. Im Node **Configuration** diese drei Werte ersetzen:

```javascript
const fediBaseUrl = 'https://YOUR_MASTODON_HOST';
const fediAccount = 'YOUR_USERNAME@YOUR_MASTODON_HOST';
const atprotoHandle = 'YOUR_ATPROTO_HANDLE';
```

Die Basis-URL enthält `https://`, der Account und das ATProto-Handle kein führendes `@`. Beispielwerte wären `https://social.example`, `alice@social.example` und `alice.example`. Keine Passwörter in diesem Node hinterlegen. Die vorhandene Platzhalterprüfung verhindert versehentliches Starten ohne Konfiguration.

## 2. Mastodon-Credential

In der Mastodon-Weboberfläche unter **Einstellungen → Entwicklung → Neue Anwendung** eine Anwendung erstellen. Als Rechte ausschließlich `read:accounts` und `read:statuses` wählen, andere voreingestellte Rechte entfernen. Die voreingestellte Redirect-URI beibehalten. Speichern und den Benutzer-Zugriffstoken kopieren.

In n8n ein **Bearer Auth** Credential erstellen, beispielsweise `Mastodon read token`. Nur den Token ohne vorangestelltes `Bearer` einfügen. Falls verfügbar, erlaubte Domains auf die eigene Mastodon-Domain beschränken. Dasselbe Credential beiden Nodes zuweisen:

- **Look up Fediverse account**
- **Fetch recent Fediverse posts**

Client-ID und Client-Secret werden bei dieser Einrichtung nicht benötigt. Auf Mastodon erfolgen ausschließlich Lesezugriffe.

## 3. ATProto-Credential

Beim ATProto-Anbieter ein eigenes App-Passwort für den Crossposter erstellen. Im Node **Create ATProto session** ein **Custom Auth** Credential anlegen und folgendes JSON in dessen JSON-Feld eintragen:

```json
{
  "body": {
    "identifier": "YOUR_ATPROTO_HANDLE",
    "password": "YOUR_APP_PASSWORD"
  }
}
```

Beide Platzhalter ersetzen. Das Handle muss mit **Configuration** übereinstimmen. Bindestriche im App-Passwort beibehalten. Das Credential diesem Node zuweisen. Nachfolgende Nodes verwenden automatisch den temporären Session-Token.

Das tatsächliche Schreibziel ist der aus dem DID-Dokument ermittelte PDS, nicht zwangsläufig die Handle-Domain. Für eine Domain-Beschränkung des Credentials zuerst den PDS prüfen: **Resolve ATProto handle → Build DID document URL → Fetch DID document → Extract PDS endpoint**. Den ausgegebenen PDS-Host im Credential erlauben; bei einem PDS-Umzug erneut prüfen und anpassen. Niemals das App-Passwort an einen ungeprüften Host senden.

## 4. Aktivieren und testen

1. Einen gegebenenfalls vorhandenen Crossposter für dieselbe Quelle deaktivieren.
2. Workflow speichern und veröffentlichen/aktivieren.
3. Ersten geplanten Produktionslauf abwarten. `bootstrapMode: 'skip_existing'` merkt die aktuell abgerufenen Beiträge als Ausgangsbestand und veröffentlicht sie nicht.
4. Danach einen neuen öffentlichen Originalbeitrag veröffentlichen. Nach spätestens dem nächsten erfolgreichen Polling-Lauf das Ziel prüfen.
5. Einen langen Beitrag und bis zu vier Bilder mit unterschiedlichen Alt-Texten testen. Replies, Boosts und nicht öffentliche Beiträge dürfen nicht erscheinen.

Manuelle Testläufe speichern statische Workflow-Daten nicht zuverlässig. Den Ausgangsbestand daher durch einen geplanten Produktionslauf etablieren. Ausführungen seriell betreiben und überlappende Läufe verhindern.

## Wiederaufnahme und Grenzen

Fertige Quellen werden in statischen Workflow-Daten gespeichert. Zielbeiträge erhalten deterministische Record-Schlüssel `masto-QUELL_ID-TEILINDEX`. Ein erneuter putRecord-Aufruf schreibt denselben Datensatz. Nach einem Teilfehler rekonstruiert ein späterer Lauf den Thread unter denselben Schlüsseln. Währenddessen weder Quelltext noch Aufteilungslogik ändern. Es gibt keine Synchronisierung nachträglicher Bearbeitungen oder Löschungen und keine Garantie bei gleichzeitig laufenden Workflow-Kopien.

Pro Abruf werden höchstens 40 aktuelle Status geladen, ohne vollständige Rückwärtspaginierung. Nach längerer Abschaltung oder bei mehr als 40 neuen Status können Beiträge fehlen. Ein Neuimport erhält einen eigenen Verarbeitungszustand und etabliert einen neuen Ausgangsbestand.

Nach ausgeschöpften HTTP-Wiederholungen stoppt die Ausführung. Beim nächsten geplanten Lauf werden noch nicht abgeschlossene Beiträge erneut versucht. Keine automatische Retry-After-Auswertung, keine separate Fehlerbenachrichtigung. Fehler unter **Executions** prüfen; bei Bedarf einen n8n-Fehlerworkflow zuweisen.

JPEG-Konvertierung entfernt Animation und Transparenz. Zu große komprimierte Bilder führen zu einem Fehler; dann in **Configuration** `imageMaxDimension` oder `imageJpegQuality` verringern. Videos, Audio, Umfragen und native Quote-Beziehungen werden nicht übertragen. HTML-Konvertierung ist auf typische Mastodon-Beitragsinhalte ausgelegt.

## Weitergabe und Zugangsdaten

Diese JSON und README können als neutrale Vorlage weitergegeben werden. Zugangsdaten ausschließlich im n8n-Credential-Speicher hinterlegen. Nach eigener Nutzung exportierte Dateien vor Weitergabe erneut auf persönliche Werte, angeheftete Testdaten und Credential-Verweise prüfen.

Erfolgreiche und manuelle Ausführungen werden standardmäßig nicht gespeichert. Fehlerausführungen können temporäre Session-Tokens enthalten: Zugriff und Aufbewahrungsdauer entsprechend begrenzen. Die JSON enthält nur Credential-Platzhalter; jeder Empfänger legt eigene Credentials an.

## Dokumentation

- [Mastodon Accounts API](https://docs.joinmastodon.org/methods/accounts/)
- [ATProto-Beiträge und Threads](https://docs.bsky.app/docs/tutorials/creating-a-post)
- [n8n HTTP Request Credentials](https://docs.n8n.io/integrations/builtin/credentials/httprequest/)
- [n8n Edit Image](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.editimage/)
