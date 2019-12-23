# REST API

Spezifikation des Speicher-Backend für das Kassenprogramm

#### Funktionsweise generell

Die Abfrage funktioniert über JSON/HTTP. Es wird nur mit POST-Requests gearbeitet, die auf die URL /api/_service_/_methode_ mit einer payload als JSON ausgeführt werden.


### Genereller Datenaufbau

#### Request

Allgemeine Datenfelder für (fast) alle Requests:

```json
{
  "token": "...some secret string...",
  "username": "bernd"
}
```

**token** ist fest vergeben (API-Key), obligatorisch
**username** ist der Benutzername, der über account/checkPassword ermittelt wurde. Entfällt beim Aufruf von account/checkPassword. Eine Authentifizierung des Users findet im backend nicht statt.


#### Response

```json
{
  "status": "error",
  "errors": [
      {
       "code": 123,
       "text": "error message",
       "value": "xxx"
      }
  ],
  "data": [
      { ... },
      { ... }
    ]
}
```

Im Fehlerfall wird errors mit näheren Informationen zum Fehler belegt, **data** ist dann ein leeres array. Im Erfolgsfall ist **data** belegt, gleichzeitig können in **errors** Warnungen enthalten sein. Das array **errors** sollte in jedem Fall ausgewertet werden.


### Objekte

#### User

Ein Useraccount, z.B. als Antwort auf ein checkPassword

```json
{
  "id": 1,
  "username": "bernd",
  "realname": "Bernd Wurst",
  "roles": [
    "admin", "user"
  ]
}
```

#### Case

```json
{
  "id": 123123,
  "handle": "abc12345678", //alphanumerisch, UUID
  "version": 1,
  "is_current_version": true,
  "customerno": "k123",
  "author": 1,
  "timestamp_create": "2019-12-12T08:30:00",
  "timestamp_lastchange": "2019-12-12T08:30:00",
  "number_of_items": 4,
  "amount_net": 100.0,
  "amount_gross": 119.0,
  "amount_vat": 19.0,
  "invoiced": true,
  "payed": false,
  "remarks": null,
  "detail": {
  	# Anwendungsspezifische Detail-Daten
  	"abholung": "Montag",
	"paletten": 1,
	"bio": true,
	"rechnungsnummer": 1234,
	"rechnungsdatum": "2019-12-12",
	"zahlart": "bar",
	"liter": 100,
	"manuelle_literzahl": 0,
	"calls": [
		{ ... Call ...},
		{ ... Call ...}
	]
  },
  "items" : [
    { ... CaseItem ...},
    { ... CaseItem ...}
  ],
  "payments": [
    { ... Payment ...},
    { ... Payment ...}
  ]
}
```

#### CaseItem

```json
{
  "id": 1232112,
  "vorgang": 123123,
  "position_number": 1,
  "article_number": "5er",
  "date": "2019-12-24",
  "count": 12,
  "unit": "Stk",
  "description": "Abfüllung in 5er",
  "price_net": 10.0,
  "price_gross": 11.9,
  "vat_rate": 19.0,
  "vat_amount": 1.9,
  "rebate_percent": null,
  "rebate_amount": null,
  "details": {
  	# Anwendungsspezifische Detail-Daten
	"liter_pro_einheit": 5,
	"autoupdate": true
	}
}
```

#### Call

Call ist anwendungsspezifisch und muss nicht allgemeingültig implementiert werden.

```json
{
  "id": 1234434,
  "case": "abc12345678",
  "number": "+49 7192 936434",
  "result": "missed",
  "remarks": null,
  "timestamp": "2019-12-12T12:30:00"
}
```

#### Payment

```json
{
	"id": 1234545,
	"receipt_id": "abc-123123213-asdad-1213",
	"case": "abc12345678",
	"timestamp": "2019-12-24 20:00:00",
	"method": "cash",
	"amount_net": 10.0,
	"amount_vat": 1.9,
	"amount_gross": 11.9,
	"card_number": null,
	"remarks": null,
	"details": {
	}
}
```

Bei Barzahlungen werden die TSE-Daten (Seriennummer/ID und fortlaufende Nummer der Buchung) mit abgespeichert, das findet in den Details Platz.



#### Customer

```json
{
  "id": 12123123,
  "customerno": "k123",
  "version": 1,
  "is_current_version": true,
  "timestamp_create": "2019-12-12T09:15:00",
  "company": "Mustermann und Co KG",
  "firstname": "Max",
  "lastname": "Mustermann",
  "address1": "Zuhause 14",
  "address2": null,
  "address3": null,
  "country": "de",
  "zip": "12345",
  "city": "Musterhausen",
  "remarks": null,
  "details": {
	  "bio_kontrollstelle": "DE-ÖKO-003",
	  "bio_kontrollnummer": "DE-BW-003-12345-AD"
	  }
  "contact": [
    { ... CustomerContact ...},
    { ... CustomerContact ...}
  ]
}
```

#### CustomerContact

```json
{
  "id": 12345,
  "customer": 12123123,
  "date_added": "2019-12-24",
  "type": "phone",
  "value": "+49 7192 936434",
  "details": {
  	},
  "remarks": null
}
```

Ein KundenKontakt bezieht sich immer auf einen Kunden-Datensatz in einer konkreten Version, analog zu Vorgang und Posten.


## Services

### account

Die Account-Verwaltung dient maßgeblich dazu, lokale Kassen-Nutzer zu differenzieren um Buchungen und Änderungen an Vorgängen einem Benutzer zuordnen zu können. Die Authentifizierung ist daher typischer Weise über eine kurze PIN oder ein biometrisches Merkmal. Das Backend unterstützt Passwörter alphanumerisch in beliebiger Länge. Die Übermittlung findet im Klartext/HTTPS statt, die serverseitige Speicherung sollte stets im Form von Hashwerten erfolgen.


#### account/checkPassword

Die Abfrage checkPassword dient dazu, eine lokal eingegebene PIN zu prüfen und dem Benutzer die Oberfläche freizuschalten. Der dadurch ermittelte Username wird bei kommenden Interaktionen mit der API mitgesendet.

**Request:**
```json
{
  ...
  "password": "secret"
}
```

**Response**
```json
{
  "status": "success",
  "errors": [
  ],
  "data": [
      "user": {
        "id": 1,
        "username": "bernd",
        "realname": "Bernd Wurst",
        "roles": [
          "admin", "user"
        ]
      }
    ]
}
```


#### account/changePassword

```json
{
  ...
  "oldpassword": "theoldone",
  "newpassword": "moresecretone"
}
```

oder wenn ein Administrator das Passwort eines anderen Nutzers ändern möchte:

```json
{
  ...
  "for_user": "kassenbenutzer",
  "newpassword": "moresecretone"
}
```


#### account/listUsers

Erfordert keine Parameter. Liefert eine Liste alle Benutzeraccounts für diesen API-Key.


#### account/addUser

```json
{
  ...
  "username": "bernd",
  "realname": "Bernd Wurst",
  "roles": [
    "admin", "user"
  ],
  "password": "veryspecial"
}
```

Erstellt einen neuen Benutzer.


### case

#### case/find

**Request**
```json
{
  ...
  "filter": {
    "field": "payed",
    "value": true
  },
  "limit": 10,
  "sort": {
    "field": "amount_gross",
    "order": "ASC"
  }
}
```

Die Antwort auf **find** liefert eine Liste mehrerer Vorgänge. Es ist **nicht** garantiert, dass die Vorgänge alle Detaildaten enthalten (Einzelposten, Zahlungen, Anrufe, Kundendaten). Um diese zu erhalten muss der konkrete Vorgang einzeln abgefragt werden.

#### case/load

```json
{
  ...
  "handle": "abc12345678",
  "version": 3
}
```
**handle** ist das über Versionen persistente Handle

**version** ist optional und kann verwendet werden um alte Versionen gezielt zu laden

oder

```json
{
  ...
  "id": 12345678
}
```
**id** bezeichnet eine Datenbank-ID (automatisch konkrete Version)

Die Antwort auf **load** ist ein vollständiges Objekt mit allen Einträgen und Kundendaten.


#### case/save

```json
{
  ...
  "case": {
    ... Case Objekt ...
  }
}
```
Das **Case**-Objekt muss seine Stammdaten und die Einträge enthalten. Eventuell zusätzlich enthaltene Kundendaten, Anrufe und Zahlungen werden ignoriert.

#### case/saveCall

Speichert einen Anruf zu einem Vorgang.

Leseanfragen sind unnötig, weil diese Informationen beim **case/load** schon enthalten sind.

```json
{
	...
	"case": "abc12345678",
	"anruf": {
		"number": "+49 7192 936434",
		"result": "missed",
		"remarks": null,
		"timestamp": "2019-12-12T12:30:00"
	}
}
```

#### case/savePayment

Speichert eine neue Zahlung zu einem gegebenen Vorgang. Änderungen an bestehenden Zahlungen sind nicht vorgesehen.

Leseanfragen sind unnötig, weil diese Informationen bei **case/load** schon enthalten sind. Vorgänge sollten stets aktuell aus dem Backend geladen werden bevor eine Zahlung abgewickelt wird.

```json
{
	...
	"case": "abc12345678",
	"payment": {
	  "timestamp": "2019-12-12T13:30:00",
  	  "method": "cash", 
	  "amount_net": 100.00,
	  "amount_vat": 19.0,
	  "amount_gross": 119.0,
	  "receipt_id": null,
	  "remarks": null,
	  }
}
```

### payment

#### payment/aggregate

Diese Methode dient vor allem der steuerlichen Auswertung der erhaltenen Zahlungen.

```json
{
  ...
  "filter": {
    "field": "method",
    "value": "cash"
  },
  "startdate": "2019-01-01",
  "enddate": "2019-12-31"
}
```

Liefert die Summe aller entsprechenden Buchungen im Zeitraum von **startdate** bis **enddate**, jeweils einschließlich.



### customer

#### customer/find

```json
{
  ...
  "filter": {
    "field": "lastname",
    "value": "Maier"
  },
  "limit": 10,
  "sort": {
    "field": "firstname",
    "order": "ASC"
  }
}
```

Die Antwort auf **find** liefert eine Liste mehrerer Kunden-Datensätze.

Wird der Filter mit leerem oder fehlendem **field** gestartet, wird in Name und Telefonnummern gesucht.


#### customer/load

```json
{
	...
	"customerno": "k123",
	"version": 1
}
```
oder 
```json
{
	...
	"id": 12123123
}
```

Der Parameter **version** ist optional. Ein Anfrage nach **customerno** ohne Versionsangabe liefert die aktuellste Version.

Die Antwort ist ein Kunden-Datensatz inklusive Kontaktdaten

#### customer/save

```json
{
	...
	"customer": {
		... Customer-Objekt ...
	}
}
```
Enthält das Objekt noch keine *id*, wird ein neuer Datensatz angelegt. Ansonsten wird der vorhandene aktualisiert.




