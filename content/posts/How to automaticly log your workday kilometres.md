---
title: How to automaticly log your workday kilometres
date: 2025-10-11
tags:
  - Node-Red
  - HomeAssistant
  - Obsidian
topics:
---


# Introduction  
My wife needs to fill in all kilometers she rode during workdays. Using Node-Red, Obsidian and Home Assistant I log this automatically! Saving us around 15 minutes every workday. We do this with the following steps.
1. Log the odometer of the car when my wive leaves home.
2. If she arrives at here office sometime after leaving the house, mark today as a workday.
3. When it's a workday, at arrival home. Store the odometer again. And calcualte thge difference with when she left.
4. Write the date, start KM, end Km and differnence in the table within obisidna. (of course the file will be created when it's not already there.)

# Prerequisites
  - [Home Assistant ]([Home Assistant](https://www.home-assistant.io/))
	  - [Node-Red Add-on]([hassio-addons/addon-node-red: Node-RED - Home Assistant Community Add-ons](https://github.com/hassio-addons/addon-node-red))
	  - [Mobile Companion App]([Home Assistant Companion Docs](https://companion.home-assistant.io/))
	  - [Hyundia Kia Connect - HACS Intergration]([Hyundai-Kia-Connect/kia_uvo: A Home Assistant HACS integration that supports Kia Connect(Uvo) and Hyundai Bluelink. The integration supports the EU, Canada and the USA.](https://github.com/Hyundai-Kia-Connect/kia_uvo))
  - [Obsidian]([Obsidian - Sharpen your thinking](https://obsidian.md/))
  - [Node-Red]([Low-code programming for event-driven applications : Node-RED](https://nodered.org/)) (With acces to the obsidian files)
# Walktrough  
Assuming you have evrything in the prerequisities setup. 
1. A home assisant instance with the node-red addon, a way to read your car's odometer (for me Hyundia/Kia Connect) and a way to track your location (for me the companion app).
2. An instance of obsidian and node-red running on that machine as well so it can access the files.
We can start the automation process.

## Stap 1: Setup & Installatie 

### Steps within Home Assistant
1. Use [this tutorial]([Zone - Home Assistant](https://www.home-assistant.io/integrations/zone/)) to create a zone for both your home and your work.
2. Create an boolean helper called Workday.
3. 2. Within the Home Assistant Node Red instance, create the following flows.
#### 1. Log the odometer of the car when my wive leaves home
- Create a Zone node that triggers when you leave the zone "Home".
- Create a function node that stores the current state of the odometer.

	`const kmStand = parseFloat(global.get('homeassistant').homeAssistant.states['sensor.kona_odometer'].state);`
	`flow.set("rit_km_start", kmStand);`
	`msg.payload = Vertrek opgeslagen: ${kmStand} km;`
	`return msg;`
#### 2. Log when arriving at work
- Create a Zone node that triggers when you enter the zone "Work".
- Create an Edit Action Node that sets the input boolean "Workday" to true
#### 3. Log when arrving back home
- Create a Zone node that triggers when you enter the zone "home".
- Create a function node, this node will do several things
	- Retrive earlier stored data
	- Store the odometer at the end of the workday.
	- Calculate the difference between departure and arrival.
	- Set the date of today.
	- Delete temporarly stored data

		`const kmStart = flow.get("rit_km_start") || 0;`
		`const kmEind = parseFloat(global.get('homeassistant').homeAssistant.states['sensor.kona_odometer'].state);`
		`const verschil = (kmEind - kmStart).toFixed(2);`
		`const eindTijd = new Date().toISOString();`

		`msg.payload = {`
		 `datum: eindTijd.split('T')[0],`
			`km_start: kmStart,`
			`km_eind: kmEind,`
			`km_verschil: verschil`
		`};`
 
		`flow.set("rit_km_start", null);`
		`flow.set("rit_starttijd", null);`
		`return msg;`
- Create an edit current state node to check if the workday boolean is true.
- Create a function node to create the API data 
- Create a http request node to call the Node Red instance with local file access.
- Create an edit current state node to set the workday boolean to false.
### Steps within Obsidian 
 Within Obsidian, create a Declaration folder. This is where we will store our monthly files with our logged kilometers.

### Steps within the Obsidian Node-Red instance
1. Create a edit http in node, with a post method for the API call we created before. My url is /write_declaration_km
2. Create a function node to prepare the filename. Make sure the prefix points to the declaration folder you've created earlier.
	`const info = msg.payload;`
	`const prefix = '/home/obsidian/Documents/PersonalAssistant/13. Knowledge/Declaratie/';`
	`msg.filename = ${prefix}${info.date.slice(0,7)}.md;`
	`return msg;`
3. Create a function node to prepare the row with the declaration information.
	`const info = msg.payload;`
	`const dataRow = | ${info.date} | ${info.km_start} | ${info.km_eind} | ${info.km_verschil} |;`
	`flow.set("dataRow", dataRow);`
	`return msg;`
4. Create a read file node to check if msg.filename exists.
5. Create a function node to reload the row data
	`const dataRow = flow.get("dataRow") || "";`
	`msg.payload = dataRow;`
	`return msg;`
6. Create a write file node to msg.filename and append to file. I've checked the "add newline (\n) to each payload?"
7. Create a http repsonse node with status code 200 to let our caller know everything was ok.

Next we will need some error handling, in case the check on step 4 notices that the file did not exist.

1. Create a edit catch node. And only catch errors from the "Check file exists" node.
2. Create a function node to create the header of our file.
	`const maand = msg.filename;`
	`msg.payload = Â  | Dag | KM Begin | KM Eind | Te Declareren |\n +`
	  `| --- | -------- | ------- | ------------- |\n;`
	`return msg;`
3. Create a write file node to msg.filename where we overwrite the file.

Thats all the steps we need! Now we can start testing. 
## Stap 3: Validation & Testing.
1. For testing purposes. I've created a inject node for each of the flows (Leave home, Arrive at work and Arrive home.)
2. One by one I will start these manually.
3. Open Obsidian to see your newly created file with the logged kilometers!
# Conclusion
This was a real fun project for me which acuatlly saves me quite some time. No more manual labor to find out the different states of the odometer and calcualte the kilometers driven for me. Hopefully you found it usefull as well. 

I'm aware I've overcomplicated things with two separate Node-Red instances. But that's because of the nature of my Homelab setup. If the file are stored one the same machine you can remove the API call steps.

Thanks for reading!