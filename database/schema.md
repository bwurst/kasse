# Datenbank-Schema für Kasse

In diesen Datenbanken können Kunden, Vorgänge und Zahlungen gespeichert werden.

In der Praxis wird das Datenbanksystem umfangreicher ausfallen.

Daher sollte dies als gemeinsamer Nenner verschiedener Software benutzt werden um von gemeinsamer Entwicklung profitieren zu können.

## Grundlegende Überlegungen

#### Detail-Daten als JSON-Feld

Die Datenbank-Struktur muss alle Felder abdecken, die für eine nachvollziehbare Kassenführung relevant sind und die potenziell für die Datenbank zur Filterung und Sortierung benutzt werden sollen.

Weitere Daten, die nur zum internen Gebrauch sind und niemals Filterkriterium sein werden (Beispiel: Signatur-Hash auf dem Kassenzettel), werden im Daten-Feld als JSON-Datensatz hinterlegt. Hier muss die Anwendung festlegen, welche Werte erforderlich sind und verarbeitet werden. Dies muss dann konsequent im Programm so umgesetzt werden.

Da die API auch im JSON-Format arbeitet, kann der Inhalt des Daten-Felds einfach in den Datensatz eingehängt werden.


#### Freie Suchfelder in den Schemata

In den Datenbank-Tabellen gibt es Felder der Form **free_int_1** oder **free_string_1**. Diese werden benutzt um für die jeweilige Anwendung relevante Daten Datenbank-suchbar und -filterbar zu hinterlegen. Die Werte in diesem Feld sollten von der Anwendung von den jeweiligen Detail-Feldern in diese Felder kopiert werden. Das Daten-Feld im JSON-Format hat immer vorrangig Gültigkeit. 

#### remarks-Felder

Nahezu alle Tabellen enthalten ein Feld **remarks**. Das kann für Freitext-Notizen zu allem benutzt werden. In den meisten Fällen kann das einfach ignoriert werden, aber manchmal ist es wichtig, Zusatzinfos irgendwo speichern zu können und dann kann die jeweilige Kasse das implementieren.


## Tabellen

Die Datenbank-Tabellen sind in Pseudo-MySQL angegeben. Die Datentypen sind nicht einheitlich, WIP.

#### Case

Die Tabelle **Case** führt alle Vorgänge (Bestellung, Lieferung, Beleg) auf. Dabei wird ein **handle** vergeben (UUID oder ähnliches) um den Vorgang von der Bestellung bis zur Zahlung immer wieder zu identifizieren. Die Datenbank-ID bezieht sich auf eine konkrete Version, das **handle** bleibt auch bei Änderungen (also neuen Versionen) immer gleich.

Rabatte können als Prozentwert (**rebate_percent**) oder als Preisabzug (**rebate_amount**) hinterlegt werden. Rabatte sind als Abzug vom gesamten Rechnungspreis zu verstehen. Diese Definition geht vom Abzug vom Bruttopreis aus, das kann ein Kassensystem selbst auch abweichend verwenden.

```sql
CREATE TABLE `Case` (
    `ID` int NOT NULL AUTO_INCREMENT,
    `handle` VARCHAR(50) NOT NULL,
    `version` int unsigned NOT NULL,
    `is_current_version` bool NOT NULL,
    `customerno` VARCHAR(50) NULL ,
    `author` int unsigned NOT NULL ,
    `timestamp_create` datetime NOT NULL ,
    `timestamp_lastchange` datetime NOT NULL ,
    `number_of_items` int  NOT NULL ,
    `amount_net` decimal  NOT NULL ,
    `amount_gross` decimal  NOT NULL ,
    `amount_vat` decimal  NOT NULL ,
    `rebate_percent` decimal NULL,
    `rebate_amount` decimal NULL,
    `invoiced` bool  NOT NULL ,
    `payed` bool  NOT NULL ,
    `free_int_1` int  NULL ,
    `free_int_2` int  NULL ,
    `free_int_3` int  NULL ,
    `free_string_1` string  NULL ,
    `free_string_2` string  NULL ,
    `free_string_3` string  NULL ,
    `case_data` longtext  NOT NULL ,
    `remarks` text  NOT NULL ,
    PRIMARY KEY (
        `ID`
    )
);
```

##### Anmerkungen
**is_current_version** wird vom Programm beim Anlegen einer neuen Version bei nur genau der aktuellsten Version auf *true* gesetzt (transaktionssicher). So kann bei einer späteren Abfrage gefiltert werden, dass nur aktuelle Versionen beachtet werden.
**customerno** kann NULL sein, wenn sich der Vorgang nicht auf einen gespeicherten Kunden bezieht (Laufkundschaft).
**author** ist der Benutzer des Kassensystems ("Kassierer").
**invoiced** bezeichnet den Zustand der erfolgten Abrechnung ohne sofortige Bezahlung, also z.B. "wird überwiesen". Der Beleg kann dann im Tagesgeschäft ausgeblendet werden ohne dass er als bezahlt gilt.


#### CaseItem

Die Tabelle **CaseItem** speichert die einzelnen Rechnungsposten. Jede Zeile bezieht sich auf einen konkreten Beleg (in einer konkreten Version), so dass bei einer neuen Version alle Einträge kopiert werden.

Zur Speicherung des Einzelpreises stehen Felder für brutto und für netto zur Verfügung. Damit werden Rundungsfehler im späteren Verlauf vermieden. Die Preise können mit höherer Genauigkeit gespeichert werden.

Rabatte können als Prozentwert (**rebate_percent**) oder als Preisabzug (**rebate_amount**) hinterlegt werden. Ein Festbetrag ist als Abzug vom Einzelpreis zu verstehen. Diese Definition geht vom Abzug vom Bruttopreis aus, das kann ein Kassensystem selbst auch abweichend verwenden.

```sql
CREATE TABLE `CaseItem` (
    `ID` int  NOT NULL ,
    `case` int  NOT NULL ,
    `position_number` int  NULL ,
    `article_number` VARCHAR(50) NOT NULL ,
    `date` date  NOT NULL ,
    `count` decimal  NOT NULL ,
    `unit` string  NULL ,
    `description` string  NOT NULL ,
    `price_net` decimal(12,4)  NULL ,
    `price_gross` decimal(12,4)  NULL ,
    `vat_rate` decimal(12,4)  NOT NULL ,
    `vat_amount` decimal(12,4)  NOT NULL ,
    `rebate_percent` decimal(12,4)  NULL ,
    `rebate_amount` decimal(12,4)  NULL ,
    `free_int_1` int  NULL ,
    `free_int_2` int  NULL ,
    `free_int_3` int  NULL ,
    `free_string_1` string  NULL ,
    `free_string_2` string  NULL ,
    `free_string_3` string  NULL ,
    `item_data` text  NULL ,
    `remarks` text  NULL ,
    PRIMARY KEY (
        `ID`
    )
);
```

##### Anmerkungen
**position_number** enthält eine fortlaufende Nummer auf dem Beleg.
**article_number** kann für eine Artikelnummer oder einen Artikel-Code benuzt werden. Die zugehörige Datenbank ist nicht Teil dieses Schemas.

#### Payment

In dieser Tabelle sind die eigentlichen Zahlungsvorgänge erfasst. Werte in dieser Tabelle sind unveränderlich zu speichern und dürfen daher nur angefügt werden. Sollte ein Vorgang in der Tabelle Case verändert werden müssen (oder in einer neueren Version nach der Zahlung andere Beträge enthalten), so sind diese Werte in der payment-Tabelle steuerlich relevant. Solche Veränderungen an der Case-Tabelle dürfen in der Anwendung nicht vorgesehen sein (bezahlte Vorgänge dürfen nicht mehr verändert werden).

```sql
CREATE TABLE `Payment` (
    `ID` int  NOT NULL ,
    `receipt_id` string  NULL,
    `case` string  NULL ,
    `timestamp` timestamp  NOT NULL ,
    `method` string  NOT NULL ,
    `amount_net` decimal  NOT NULL ,
    `amount_vat` decimal  NOT NULL ,
    `amount_gross` decimal  NOT NULL ,
    `card_number` string  NULL ,
    `free_int_1` int  NULL ,
    `free_int_2` int  NULL ,
    `free_int_3` int  NULL ,
    `free_string_1` string  NULL ,
    `free_string_2` string  NULL ,
    `free_string_3` string  NULL ,
    `payment_data` text  NULL ,
    `remarks` text  NULL 
);
```

**receipt_id** kann auf eine andere Tabelle verweisen, kann aber auch einfach eine Seriennummer des Kassenbons sein. NULL bedeutet, dass kein Kassenbon erstellt wurde.
**case** referenziert einen Vorgang in der anderen Tabelle anhand seines Handles (also keine konkrete Version). Ist der Wert an dieser Stelle NULL, dann wurde der Vorgang nicht erfasst oder bereits gelöscht aber die Zahlung muss als Aufzeichnung erhalten bleiben.
**method** kann "cash", "ec", "banktransfer" oder ähnliches enthalten. Die möglichen Werte sind von der Anwendung festzulegen.
**card_number** wird verwendet um eine EC- oder Kreditkarte zu indentifizieren. Nach der Nummer könnte man suchen wollen. Weitere Details die eventuell notwendig sind, müssen in **payment_data** hinterlegt werden.


### Customer

Die Einträge dieser Datenbank sollten in der Regel von anderen Tabellen mittels der **customerno** verlinkt werden, die über mehrer Versionen hinweg persistent ist.

Ist es für die Kassenführung unschädlich, wenn alte Versionen aus der Kundendatenbank entfernt werden?

```sql
CREATE TABLE `Customer` (
    `ID` int  NOT NULL ,
    `customerno` string  NOT NULL ,
    `version` int  NOT NULL ,
    `is_current_version` bool  NOT NULL ,
    `timestamp_create` datetime  NOT NULL ,
    `company` text  NULL ,
    `firstname` string NULL ,
    `lastname` string NULL ,
    `address1` string NULL ,
    `address2` string NULL ,
    `address3` string NULL ,
    `country` string NULL,
    `zip` string NULL ,
    `city` string NULL ,
    `free_int_1` int  NULL ,
    `free_int_2` int  NULL ,
    `free_int_3` int  NULL ,
    `free_string_1` string  NULL ,
    `free_string_2` string  NULL ,
    `free_string_3` string  NULL ,
    `customer_data` text  NULL ,
    `remarks` text  NULL ,
    PRIMARY KEY (
        `ID`
    )
);
```


### CustomerContact

```sql
CREATE TABLE `CustomerContact` (
    `id` int  NOT NULL ,
    `customer` int  NOT NULL ,
    `date_added` date  NOT NULL ,
    `type` string  NOT NULL ,
    `value` string  NOT NULL ,
    `contact_data` text  NULL ,
    `remarks` text  NULL ,
    PRIMARY KEY (
        `id`
    )
);
```
**customer** bezieht sich auf eine ID der Kundentabelle, somit müssen bei einer neuen Version eines Kunden auch alle zugehörigen Kontakte neu dupliziert werden.




