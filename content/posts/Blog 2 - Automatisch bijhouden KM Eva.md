---
title: Automatisch KM declaratie bijhouden.
date: 2025-10-09
---

# Tutorial: Automatische Kilometerregistratie voor Declaraties met Home Assistant, Node-RED en Obsidian

Met deze tutorial stel je een volledige workflow in om automatisch ritten vast te leggen en als maandelijkse Markdown-bestanden in Obsidian bij te houden, gebaseerd op je Home Assistant-data. We gebruiken:

- Home Assistant voor odometer- en locatie-data
    
- Node-RED add-on in Home Assistant voor logica en bestandsbeheer
    
- Obsidian voor gestructureerde opslag in je vault
    

Volg de onderstaande stappen zorgvuldig. Aan het eind van dit artikel kun je al je zakelijke ritten automatisch exporteren naar maandelijkse declaratiebestanden in Obsidian.

---

## 1. Benodigdheden en Voorbereiding

1. **Home Assistant setup**
    
    - Zorg dat je Home Assistant draait met de Node-RED add-on geïnstalleerd.
        
    - Je hebt een odometer-sensor, bijvoorbeeld `sensor.kona_odometer`, die de totaalstand van je auto bijhoudt.
        
    - Je hebt een presence- of zone-entiteit, bijvoorbeeld `person.eva` of `device_tracker.eva`, met minimaal de zones `home` en `work`.
        
    - Maak een helper `input_boolean.werkdag_eva` om aan te geven wanneer ritten daadwerkelijk voor werk(declaratie) zijn.
        
2. **Obsidian vault**
    
    - Kies of maak een map in je Obsidian vault, bijvoorbeeld:  
        `/home/obsidian/Documents/PersonalAssistant/13. Knowledge/Declaratie/`
        
    - Hier komen straks je maand-MD-bestanden zoals `2025-10.md`.
        

---

## 2. Flow 1: Ritgegevens berekenen en versturen

## 2.1 Helpers in Home Assistant

1. **Rit-start automation**
    
    - Trigger: `person.eva` verlaat zone `home`.
        
    - Action:
        
        - Sla de huidige odometer-stand op in een `input_number.rit_start_odometer`.
            
        - Zet `input_boolean.werkdag_eva` op `on` als het een werkdag is (optioneel via time-based automation).
            
2. **Rit-stop automation**
    
    - Trigger: `person.eva` keert terug naar zone `home`.
        
    - Condition: `input_boolean.werkdag_eva` is `on`.
        
    - Action:
        
        - Bereken eind-km, begin-km en verschil:
            
            text
            
            `km_start: "{{ states('input_number.rit_start_odometer')|float }}" km_eind: "{{ states('sensor.kona_odometer')|float }}" km_verschil: "{{ (states('sensor.kona_odometer')|float - states('input_number.rit_start_odometer')|float)|round(2) }}" date: "{{ now().strftime('%Y-%m-%d') }}"`
            
        - Verstuur deze data via een REST-command naar Node-RED endpoint `/write_declaration_km`.
            
3. **REST-command**  
    Voeg in `configuration.yaml` toe:
    
    text
    
    `rest_command:   write_declaration_km:    url: "http://<HA-IP>:1880/write_declaration_km"    method: POST    content_type: application/json    payload: >      {"date":"{{ date }}",       "km_start":{{ km_start }},       "km_eind":{{ km_eind }},       "km_verschil":{{ km_verschil }}}`
    

---

## 3. Flow 2: Node-RED – Maandbestand aanmaken of updaten

Importeer in Node-RED de volgende flow:

json

``[   {     "id":"cdf733678090f59a","type":"http in","url":"/write_declaration_km","method":"post","x":140,"y":200,     "wires":[["827374a7eff8ae40"]]   },   {     "id":"827374a7eff8ae40","type":"function","name":"Prepare filename",     "func":"const info=msg.payload;\nconst prefix='/home/obsidian/Documents/PersonalAssistant/13. Knowledge/Declaratie/';\nmsg.filename=`${prefix}${info.date.slice(0,7)}.md`;\nreturn msg;","x":380,"y":200,     "wires":[["file_in_check"]]   },   {     "id":"file_in_check","type":"file in","name":"Check file exists","filename":"","filenameType":"msg",     "format":"utf8","sendError":true,"x":620,"y":200,"wires":[["file_exists_switch"]]   },   {     "id":"file_exists_switch","type":"switch","name":"Bestaat bestand?","property":"error","rules":[{"t":"undefined"},{"t":"else"}],     "outputs":2,"x":860,"y":200,     "wires":[["prepare_row"],["make_header"]]   },   {     "id":"make_header","type":"function","name":"Maak header",     "func":"const maand=msg.filename.slice(-7,-3);\nmsg.payload=`# Declaratie ${maand}\\n\\n`+\n  `| Dag | KM Begin | KM Eind | Te Declareren |\\n`+\n`| --- | -------- | ------- | ------------- |\\n`;\nreturn msg;","x":1120,"y":160,     "wires":[["append_header","prepare_row"]]   },   {     "id":"append_header","type":"file","name":"Write header",     "filename":"","filenameType":"msg","overwriteFile":"true","appendNewline":false,"encoding":"utf8","x":1400,"y":160,"wires":[]   },   {     "id":"prepare_row","type":"function","name":"Prepare row",     "func":"const info=msg.payload;\nconst dag=info.date.split('-')[2];\nmsg.payload=`| ${info.date} | ${info.km_start} | ${info.km_eind} | ${info.km_verschil} |`;\nreturn msg;","x":1120,"y":240,     "wires":[["append_row"]]   },   {     "id":"append_row","type":"file","name":"Append row","filename":"","filenameType":"msg",     "appendNewline":true,"overwriteFile":"false","encoding":"utf8","x":1400,"y":240,"wires":[["ecfd6ebf267f3d5a"]]   },   {     "id":"ecfd6ebf267f3d5a","type":"http response","statusCode":"200",     "headers":{"Content-Type":"application/json"},"x":1620,"y":200,"wires":[]   } ]``

## Uitleg per node

1. **HTTP In**: Luistert op POST `/write_declaration_km`.
    
2. **Prepare filename**: Berekent het maand-MD-bestand op basis van `date.slice(0,7)`.
    
3. **File In**: Probeert het bestand te lezen; bij fout (`sendError=true`) volgt de tweede tak.
    
4. **Switch**: Splitst in “bestaat” (geen error) en “bestaat niet” (error).
    
5. **Maak header**: Genereert de tabelkop in Markdown met de kolomtitels.
    
6. **Write header**: Schrijft de header alleen bij nieuwe bestanden (overwrite).
    
7. **Prepare row**: Maakt de rij met dag en km-waarden.
    
8. **Append row**: Voegt de rij altijd toe met een newline.
    
9. **HTTP Response**: Bevestigt de succesvolle oproep aan Home Assistant.
    

---

## 4. Testen en Validatie

1. Zet je `input_boolean.werkdag_eva` op `on`.
    
2. Simuleer vertrek en aankomst in Home Assistant of verlaat en betreed de `home` zone met je telefoon.
    
3. Controleer of in `/home/obsidian/.../Declaratie/` een nieuw bestand verschijnt of een bestaande file wordt uitgebreid.
    
4. Open het maand-MD-bestand in Obsidian; je ziet vanzelf de tabel met je ritten als rijen onder de header.
    

---

## 5. Vragen en Maatwerk

- Wil je de tabel in Obsidian automatisch herordenen op datum?
    
- Wil je extra kolommen zoals project of kostenplaats?
    
- Heb je een andere opslaglocatie of YAML-frontmatter nodig?
    

Laat het weten, dan help ik je de flow verder aan te passen. Veel succes met je geautomatiseerde kilometerdeclaratie!