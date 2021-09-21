Entwicklung Masterportal Addon TrafficCount (gfiTheme)
======================================================

[Masterportal](https://bitbucket.org/geowerkstatt-hamburg/masterportal/src/dev/) TrafficCount Addon zur Verwendung in der "Digitale Plattform Stadtverkehr - Berlin"-Plattform.
Basiert auf dem [Traffic Count Addon der Geowerkstatt Hamburg](https://bitbucket.org/geowerkstatt-hamburg/addons/src/dev/gfiThemes/trafficCount/).

main branch = Masterportal 2.6.x
dev branch = Masterportal 2.11.x

## Development

1. Clone the masterportal repo and enter the directory
```
git clone https://bitbucket.org/geowerkstatt-hamburg/masterportal.git
cd masterportal
```

2. checkout the branch which works with the addon (see above). As the time of this writing the stable branch is 2.6.3, which works with the main branch of this repo
```
git checkout stable
```

* You can also just use a copy of the masterportal source in the desired Version.


3. delete the (masterportal default empty) _addons_ folder and clone this branch in the  _addons_ directory
```
rmdir addons
git clone https://gitlab.wemove.com/wemove/masterportal-addon-trafficCount addons
```

4. still in the masterportal directory install the node modules required
```
npm install
```

5. Start the Server in Development
```
npm install
```

You can view the running Masterportal at https://localhost:9001

* At https://localhost:9001/portal/basic you can see the basic portal.

6. put your Portals Config in the _portal_ dir of the root folder. To see the functionality of the TrafficCount Addon, you have to use it in your Portal Config.


[siehe Setup Masterportal](https://bitbucket.org/geowerkstatt-hamburg/masterportal/src/dev/doc/setupDev.md)

[siehe Addon Entwicklung mit dem Masterportal](https://bitbucket.org/geowerkstatt-hamburg/masterportal/src/dev/doc/addOnsVue.md)

## Tests

Executes unit tests. This also includes the unit tests of add-ons.
```
npm run test
```


## Deployment

```
npm run buid
```

gebaute Version aus _dist/mastercode_ in Deployment-Umgebung kopieren.
Bsp:

```
dist/mastercode/2_6_3_DEV_2021-09-18__22-01-29/
```

ACHTUNG:
Die gebaute Masterportal-Javascript-Datei ist hierbei von dem Namen des Ordners abhängig.

Ist die Version des Masterportals im Deployment-Zielverzeichnis bisher im Unterordner _2_6_3_, so muessen alle Vorkomnisse von
```
2_6_3_DEV_2021-09-18__22-01-29
```
durch
```
2_6_3
```
in
```
dist/mastercode/2_6_3_DEV_2021-09-18__22-01-29/js/masterportal.js
```
ersetzt werden.


## TrafficCount Parameter

Werden an das TrafficCount Plugin als gfi Theme Parameter angegeben.

**meansOfTransports**

Array der anzuzeigenden Means of Transports aus dem Stream. Werden untereinander angezeigt. Können beliebig viele sein.
Pflichtparameter. Bestehen jeweils aus:

* **id**: String: Id des Datensatzes in der Quelle des SensorThings Stream bsp.: "Anzahl_Kfz".
* **type**: String: Art der Auswertung des Datensatzes, nur "counting" oder "speed" möglich.
* **label**: String: zugehöriges Label, z.B. "KFZ Anzahl",
* **dayInterval**: String: eingesetztes MessIntervall für die Tagesmessung bsp: "1-Min" oder "5-min". Muss in der Quelle auch so angegeben sein.


**archiveStartDate**

String, der den Anfang der Anzeige von archivierten Daten für die Woche und die Jahr Akkumulationen angibt.
Kein Pflichtparameter. Default "2020-01-01",


**headerProperties**

Array aus Strings, die die Header Informationen, die aus dem Stream entnommen werden sollen referenzieren.
Jeder Eintrag sollte der Key einer Übersetzung aus _/locales/XX/common.json_ Bereich "trafficCountProperties" sein.
Dann erscheint das Label des Headerfelds lokalisiert.

Beispiel _richtung_ entspricht richtung aus Stream. Im englischen übersetzungsfile _/locales/en/common.json_ sollte im Traffic Count Properties das Snippet
```
"trafficCountProperties": {
    "richtung": "Direction",
},
```
zu finden sein.


### Vollständiges Beispiel Parameter:

```
"gfiTheme": {
  "name": "trafficCount",
  "params": {
    "headerProperties": [
      "operation_start", "position", "richtung", "lane"
    ],
    "archiveStartDate": "2015-01-01",
    "meansOfTransports": [
      { "id": "Anzahl_Kfz", "type": "counting", "label": "KFZ Anzahl",
        "dayInterval": "5-Min" },
      { "id": "Geschwindigkeit_Kfz", "type": "speed", "label": "KFZ Geschwindigkeit",
        "dayInterval": "5-Min" },
      { "id": "Anzahl_Pkw", "type": "counting", "label": "PKW Anzahl",
        "dayInterval": "5-Min" },
      { "id": "Geschwindigkeit_Pkw", "type": "speed", "label": "PKW Geschwindigkeit",
        "dayInterval": "5-Min" },
      { "id": "Anzahl_Lkw", "type": "counting", "label": "LKW Anzahl",
        "dayInterval": "5-Min" },
      { "id": "Geschwindigkeit_Lkw", "type": "speed", "label": "LKW Geschwindigkeit",
        "dayInterval": "5-Min" }
    ]
  }
}
```

## Entwicklung Komoot Suche

In die Suchleiste ist eine Suche mit [Komoot Photon](https://photon.komoot.io) integriert.

Siehe auch die Dokumentation der [config.json](./doc/config.json.md#markdown-header-portalconfigsearchbarkomoot)
