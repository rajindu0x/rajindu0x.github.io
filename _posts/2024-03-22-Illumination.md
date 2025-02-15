---
title: Illumination - HTB Forensics
date: 2024-03-10 23.13 +0530
categories: [Digital Forensics, HackTheBox]
tags: [Forensics, git, discord]
---

## Challenge Description:
A Junior Developer just switched to a new source control platform. Can you find the secret token?

## Enumeration:
We are given a zip file. Upon unzipping the file we can see the given folder is a git repository by .git folder.
![listing the files](/assets/img/posts/illumination/ss1.png)

As this is a git folder we should be able to see the git log.

````
$git log
commit edc5aabf933f6bb161ceca6cf7d0d2160ce333ec (HEAD -> master)
Author: SherlockSec <dan@lights.htb>
Date:   Fri May 31 14:16:43 2019 +0100

    Added some whitespace for readability!

commit 47241a47f62ada864ec74bd6dedc4d33f4374699
Author: SherlockSec <dan@lights.htb>
Date:   Fri May 31 12:00:54 2019 +0100

    Thanks to contributors, I removed the unique token as it was a security risk. Thanks for reporting responsibly!

commit ddc606f8fa05c363ea4de20f31834e97dd527381
Author: SherlockSec <dan@lights.htb>
Date:   Fri May 31 09:14:04 2019 +0100

    Added some more comments for the lovely contributors! Thanks for helping out!

commit 335d6cfe3cdc25b89cae81c50ffb957b86bf5a4a
Author: SherlockSec <dan@lights.htb>
Date:   Thu May 30 22:16:02 2019 +0100

    Moving to Git, first time using it. First Commit!

````
By reading the log we can see, the author accidentally did a commit with a unique token on 30th May(token not visible as it's only the log), but removed the unique token on 31st May. We can inspect each commit with $git show {commit-id}.

But before that, let's examine the two files starting with config.json
````json
{

	"token": "Replace me with token when in use! Security Risk!",
	"prefix": "~",
	"lightNum": "1337",
	"username": "UmVkIEhlcnJpbmcsIHJlYWQgdGhlIEpTIGNhcmVmdWxseQ==",
	"host": "127.0.0.1"

}

````
The file contains a username encoded in base64, hostname and LightNum and two other values for token and prefix. The token value says that we need to replace the token with it's real value. But where is it?

Upon decoding the username, we are given a little hint!
![decoded base64 username](/assets/img/posts/illumination/ss4.png)

Now, let's examine bot.js file to see how we can leverage the information we already found.

````js
cat bot.js 
//Discord.JS
const Discord = require("discord.js");
const client = new Discord.Client();
const fs = require("fs");
var config = JSON.parse(fs.readFileSync("./config.json"));


//Node-Hue-API
var hue = require("node-hue-api"),
	HueApi = hue.HueApi,
	lightState = hue.lightState;

//Display command results in console
var displayResult = function(result) {

	console.log(JSON.stringify(result, null, 2));
	
};

//Function taken from campushippo.com
var rgbToHex = function (rgb) { 

  var hex = Number(rgb).toString(16);
  if (hex.length < 2) {

       hex = "0" + hex;
  }

  return hex;
};

//Function taken from campushippo.com
var fullColorHex = function(r,g,b) {   
  var header = "0x"
  var red = rgbToHex(r);
  var green = rgbToHex(g);
  var blue = rgbToHex(b);
  return header+red+green+blue;
};

//Declarations
var host = config.host,
	username = config.username,
	api = new HueApi(host, username),
	state = lightState.create(),
	prefix = config.prefix,
	lightNum = config.lightNum;

//Bot code
client.on("ready", () => {
	console.log(`Logged in as ${client.user.tag}!`);
});

client.on("message", message => {
	if (message.author.bot) return; //Ignore bot messages
	if (message.content.indexOf(prefix) !== 0) return; //Ensure prefix is at the beginning

	const args = message.content.slice(prefix.length).trim().split(/ +/g); //Split command into arguments
	const command = args.shift().toLowerCase(); 

	switch (command) {

		case "light.off" : //Turn light off
			api.setLightState(lightNum, state.off())
		       .then(displayResult)
		       .done();
			message.channel.send("Light Off!");
			break;

		case "light.on" : //Turn light on
			api.setLightState(lightNum, state.on())
			   .then(displayResult)
			   .done();
			message.channel.send("Light On!");
			break;

		case "light.rgb" : //Change light colour
			let r = args[0];
			let g = args[1];
			let b = args[2];
			api.setLightState(lightNum, state.on().rgb(r, g, b))
			   .then(displayResult)
			   .done();
			const embed = new Discord.RichEmbed()
				.setTitle('Light Colour Change')
				.setColor(fullColorHex(r, g, b))
				.setDescription(`Red Value: ${r}. Green Value: ${g}. Blue Value: ${b}`);
			message.channel.send(embed);
			break;

		case "light.switch" : //Switch to different light
			let num = args[0];
			lightNum = num;
			//fs.writeFile("./config.json", JSON.stringify(config))
			message.channel.send(`Light Number switched to ${lightNum}`);
	}
});

client.login(Buffer.from(config.token, 'base64').toString('ascii')) //Login with secret token

````

## Solution: 
Perfect! Upon reading the script we can see that the script is used to connect to a discord bot, implemented using discord.js library. The last line of the script implies that the client needs to login to the discord bot by supplying the base64 encoded token fetched from config.json file. The script will then decode the token and logs the client in.

Now it's clear that we need to find the working token. For this lets examine the git commits. Specifically, the commit done on 'Date: Fri May 31 12:00:54 2019 +0100', as the author said he removed the unique token in that commit.

![token found](/assets/img/posts/illumination/ss2.png)

We found the token. We can leverage this token to connect to sherlocksec's discord bot. Upon decoding the token we get our flag!
![token found](/assets/img/posts/illumination/ss3.png)

## Lessons learned: 
This scenario underscores the need for developers to handle sensitive information securely, avoiding hardcoded tokens in source code and opting for safer storage methods like environment variables. Regular review of commit history helps catch unintended leaks. 
