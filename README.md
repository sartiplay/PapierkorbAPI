# Papierkram API

## Probleme

### Doppelte erstellung

- Bei der erstellung eines Kunden, wird nicht gecheckt ob der schon erstellt wurde
- Das selbe gilt auch für jegliche Rechnungen
- Die Papierkram API hat keine möglichkeit die Kunden, an ihrem Namen,
  durch eine `GET anfrage, zu returnen
- Es kann nur eine `ID` verwendet werden, diese `ID` ist aber eine zufällig generierte Nummer,
  die keinen zusammenhang zu irgenetwas hat. Es ist also unmöglich nach der ID zu filtern, da wir
  diese nicht wissen, und diese nur durch einen API Call bekommen könnten.

---

## Lösungen

### Lösung Doppelte erstellung

---

### Lösung 1

- Bei jeder erstellung eines neuem Dokumentes, wird eine API anfrage an Papierkram gemacht, die alle Kunden und alle deren Rechnungen returned, dann gleichen wir ab ob die schon exsitiert, und wenn nicht, dann erstellen wir die Rechnung. _Wenn benötigt auch den Kunden_

  #### Probleme Lösung 1

  1. **Teuer**
     - Bei jeder neuen Rechnung alle Kunden abzurufen kostet sehr viele Credits auf die Zukunft betrachtet ist dies nicht zu empfehlen
  2. **Langsam**
     - Am anfang wird dies kein Problem sein, wenn mann dann aber an einen Punkt kommt, wo man viele Rechnungen und Kunden hat, dauert es sehr lange nur durch alle durchzufiltern und die Rechnung / Kunden zu finden.

### Lösung 2

#### Was wird benötigt.

1. Eine Sekundäre Datenbank, die alle Kunden und Rechnungen seperat speichert, damit wir keine Credits beim prüfen benötigen.

   ##### Ablauf

   1. API Call von Shopify oder Manuel für neue Rechnungen hinzufügen.
   2. Sekundäre Datenbank wird angepingt und es werden alle gespeicherten Kunden und Rechnungen
      Abgefragt.
   3. Existiert der Kunde bereits in dieser Datenbank, wissen wir das er auch
      schon in Papierkram existiert.
   4. Danach kann auch gleich die Rechnung verglichen werden. Existiert sie schon,
      haben wir uns einen Ünnötigen Papierkram API call gespart.
   5. Existiert der Kunde jedoch **Nicht** in der Sekundären Datenbank, dann gibt es
      2 möglichkeiten.

      1. Wir fetchen nochmals alle Kunden und Rechnungen von Papierkram und
         tragen nochmals alle Kunden und Rechnungen neu ein, in die
         Sekundäre Datenbank.
      2. Wir sind uns sicher das die Datenbank _Up To Date_ ist, und wir erstellen
         den Kunden und / oder die Rechnung.

2. Dies ist jetzt Optional, wir erstellen einen eigenen API endpunkt, der jede Stunde einen API call
   an Shopify macht und überprüft, ob schon neue Bestellungen getätigt wurden.
   Danach wird alles Automatisch verrechnet, wie in schritt 1.

   **Wichtig zu Beachten**
   Für die Methode 2 ist es zwingen nötig, das die API und die Datenbank **24/7** gehostet sind.
   Dies ist theoretisch auf allen Geräten oder Hosting Services möglich.
   Um die API zu starten kann einfach `npm run dev` oder `npm run build`, verwendet werden.

---

## Aufbau

### Services

- **Papierkram API** _Required_
- **Shopify API** _Optional_
- **APP.js (API Handler)** _Required_
- **Sekundäre Datenbank** _Optional_
- **Permanent API Endpoint** _Optional_

---

## Ablauf

### Lösung 1

#### APP.js (API Handler)

|
`GET Request` -> Get All CLients
V

#### Papierkram API

|
`RESPONSE` -> Respons with a JSON Object of all Clients.
V

#### APP.js (API Hanlder)

```javascript
res.forEach((element) => {
  if (element.data.Name === currLoadedClient.Name) {
    /*
        `GET Request Papierkram API` -> Get all Inqueries from the Client
    */
  } else {
    `PUSH Request`; /* Push the New Client and Inquery */
  }
});
```

|
`GET Request` -> Get all Inqueries from Client
V

#### Papierkram API

|
`RESPONSE` -> Response with all the Inqueries of the Client.
V

#### APP.js (API Hanlder)

```javascript
res.forEach((element) => {
  if (element.data.Name === currLoadedClient.Inquery.Name) {
    return null;
  } else {
    `PUSH Request`; /* Push a new Inquery to the Client */
  }
});
```

### Lösung 1 Insgesamte API Calls an Papierkram.

#### **4**

1. `GET Request` -> All Clients
2. `GET Request` -> All Inqueries
3. `PUSH Request` -> Push New Client
4. `PUSH Request` -> Push New Inquery

---

### Lösung 2

#### APP.js (API Handler)

|
`GET Request {Client Name}` -> Get Repsonse if Client is in the Database
V

#### Sekundäre Datenbank

|
`RESPONSE` -> Response with Either Undefiend or A JSON Object.
V

#### APP.js (API Hanlder)

```javascript
if (res) {
  res.data.inqueries.forEach((element) => {
    if (element === currLoadedClient.Inquery) {
      return null;
    } else {
      `PUSH Request`; /* Push new Inquery to Client */
    }
  });
} else {
  `PUSH Request`; /* Push new Client + Inquery */
}
```

|
`PUSH Request` -> Push a new Client and / or new Inquery
V

#### Sekundäre Datenbank

|
`RESPONSE` -> Response "Succses"
V

#### APP.js (API Hanlder)

|
`PUSH Request` -> Push new Client and / or new Inquery
V

#### Papierkram API

### Lösung 2 Insgesamte API Calls an Papierkram.

#### **2**

1. `PUSH Request` -> Push New Client
2. `PUSH Request` -> Push New Inquery

---

## Zusammenfassung

### Lösung 1

Diese Lösung hat am meisten API calls and die Papierkram API, braucht aber keine zusätlichen Ressourcen.
Ist aber durch die vielen API calls auch längsämer, und bei vielen daten kann es auch zu problemen kommen, da man von Papierkram nur **50** **_Kunden_** auf einmal fetchen kann.

Bedeutet, wenn man mehr als 50 Kunden hat, muss man für nur eine Rechnung schon mehrere `GET Requests` machen. Was den Credits verbrauch, sehr stark erhöhen kann.

### Lösung 2

Diese Lésung braucht am wenigsten API Credits. Dafür braucht es aber eine zusätzliche Datenbank Ressource, die auch immer gepflegt werden muss.

Mann kann eine sogenante `GET QUERY` anwenden, bei dem man nach dem Namen oder sonstigen Daten, in der Datenbank suchen kann, und nicht immer alle Kunden auf einmal laden muss.
Da man diese `GET QUERY` benutzen kann, ist es auch um einiges schneller und hat viel weniger probleme
mit vielen Kunden und Daten.

#### Was ist jetzt das beste?

Dies müsst ihr entscheiden.

1. Falls die API nur selten verwendet wird, dann würde ich **Lösung 1** empfehlen, da sie einfacher umzusetzen ist. Und es keine 2te Datenbank benötigt.

2. Falls die API eher öfter verwendet werden soll, und auch als ein Hauptstück integriert werden soll, dann würde ich **Lösung 2** empfehlen. Da diese am meisten Credits spart, und auf längere Zeit, mit vielen Kunden auch weniger Probleme mit dem `GET Request` haben wird. Dazu ist sie auch bedeuten schneller im lesen der Daten.
