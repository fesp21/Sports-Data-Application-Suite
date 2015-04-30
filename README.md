#Sports Data Application Suite

This repository contains a suite of applications that provide web based realtime scores and updates for soccer or any other configured sport.

## Overview

The software is comprised of 4 discrete Node.js applications:

### Sports Data Provider

The Sports Data Provider (SDP) is a 'headless' Node.js application which reads new or updated FTP files, parses them and then writes JSON files. It then pushes the JSON to the Sports Faye Server (SFS).

### Sports Data Viewer

The Sports Data Viewer (SDV) is a Node.js application built with the Geddy framework which reads JSON and displays it. Each view also hooks up to the Faye server so it can receive real time data updates.

### Sports Faye Server

The Sports Faye Server (SFS), Node.js, is the real time data push server. It is a simple implementation of a Faye Server.

### Soccer Simulator

The Soccer Simulator is a Node.js application built with the Geddy Framework that builds test XML files and pushes them to the Sports Data Provider application as if it were the real data provider.

### Workflow overview
How do these applications work together? Here is a basic overview:

1. The Sports Data Provider (SDP) will receive XML file pushes from a vendor (currently Opta Sports) or from the Soccer Simulator (for testing locally or on development or production systems).
2. The SDP will then parse the XML file and generate JSON and aggregate data.
3. The SDP will then write the JSON to a directory (shared and usually an S3 bucket) and push the JSON to the Sports Faye Server (SFS) with the JSON type and Game ID as the channel name, e.g. 'gamestats-f78645'.
4. The SFS will broadcast the JSON data to any subscribers of the channel (standard Faye server behavior)
5. Before or during the above steps a web browser will render a 'view' generated from the Sports Data Viewer (SDV) which will have read JSON generated by the SDP (the latest JSON written)
6. The rendered view will also have the JavaScript registering itself to the correct Faye channel so pushes from the SFS will update the views. 

### Basic Setup

Here are some quick instructions to get the system up and going. These steps assume that you know how to launch Node.js applications and configure and launch Node.js applications built using the Geddy framework.

 
1. Configure the SDP and SDV to look to the same directory for JSON files. 
2. Configure SDP and SDV to look to the correct URL/IP for the SFS.
3. Configure the Soccer Simulator so it writes or FTPs its text XML files to the configured directory that the SDP is expecting
4. Launch the Redis Database
5. Have each application run as a separate Node.js process.
6. Launch the Soccer Simulator in a browser (defaults to port 4001)
7. Click the 'Build Match Files' link. This generates files that are time-stamped to your current local time.
8. Now click the 'Simulation Setup', select which game you desire, then submit the form (ignore the seconds delay input box, 2 seconds is a good default)
9. Now launch the Sports Data Viewer app in a browser window (defaults to port 4000)
10. You should now see game data updating in the window. 

# Individual Application Details
This section will provide more details about each individual application that comprise the suite of software.

##Sports Data Provider (SDA)

This is the main application that drives all the other applications. Its main function is to:

1. Receive XML files from a data provider (currently Opta Sports: http://www.optasports.com/)
2. Parse the XML to JSON
3. Calculate aggregate data after the JSON has been parsed
4. Push the JSON to the Sports Faye Server (SFS)

###Class Descriptions:

####config.js
This is the main configuration class. Below is a description of each configuration option:


	var config = {
		
		'applicationMode' : engagementConfig.applicationMode,
		'FPTDirectory' : engagementConfig.FPTDirectory,
		'JSONDirectory' :  rootPath + '/sportsdataviewer/data/json',
		// the OPTA squads file name (will be deprecated in the future)
		'squadFileName' : engagementConfig.squadFileName,
		'scheduleFileName' : engagementConfig.scheduleFileName,
		'rootPath' : rootPath,
		
		// strings to check in the XML for verification of correct XML data
		// these are set in the EngagementConfig.js file
		'competitionName' : engagementConfig.competitionName,
		'competitionID' : engagementConfig.competitionID,
		'season' : engagementConfig.season,
		'seasonName' : engagementConfig.seasonName,
		'seasonID' : engagementConfig.seasonID,
		
		// dir for aggregate data. this is not used and will be removed
		'aggregateDataDirectory' : '', 
		
		// these are strings to look at files from OPTA.
		'feedTypeStringID' : {
			'f1' : '-results',
			'f9' : '-matchresults',
			'f13': 'commentary-',
			'f30': 'seasonstats-',
			'f40': '-squads'
		},
		
		// we are using the gmail email module you might change this for your setup
		'emailSendList' : 'email@someorg.com, email2@someorg.com',
		'emailCCList' : 'ccEmail@someorg.com',
		'emailUserAccount' : 'someGoogleEmailAccount@gmail.com',
		'emailUserPassword' : 'thePassword',
		'emailSMPT' : 'smtp.gmail.com',
		
		// the application can add tweets to commentary
		// these are your twitter application settings
		'twitterOAuth' : {
			'consumer_key' : 'your key',
			'consumer_secret' : 'your secret',
			'access_token' : 'your token',
			'access_token_secret' : 'your token secret'
		},
		// these are the twitter accounts to scrape for tweets
		'twitterCommentaryAccountArray' : ['someTwitterHandle', 'anotherTwitter],
		'twitterHashTagFlag' : '#hashTagToScrapeTwitterAccountsFor',
		
		// faye server settings
		'faye' : {
		
			// these are the types of 'things' we broadcast
			'channel' : {
				'commentary' : 'commentary',
				'default' : 'faye',
				'gamestats' : 'gamestats',
				'schedule' : 'schedule',
				'scoreboard' : 'scoreboard',
				'squad' : 'squad'
			},
			'url' : engagementConfig.fayClientURL,
			// this is just a big long string to deter packet sniffers from figuring
			// out what we're doing. for total security use the faye server under
			// and SSL cert.
			'publishPassword' : '78654323MyVeryL0ngStr1ngTh4tIsC00l4ndYouC4ntT0uchThi5IfY0uTry9907654'
		}
	}

####app.js

This class provides the functionality for

1. Directory watching
2. Validation that incoming files are valid for the configured sports event
3. Transformation of XML to JSON
4. Outputs Errors when Kue fails.

If the application was to be used for a different event, e.g. Premier League vs. World Cup 2014, the configuration file would need to be updated with the correct check strings.

####Controller.js

This class is just a simple thin 'dispatcher' so the app.js class and the supporting service classed are more loosely coupled.

####CronService.js

This class handles all the Cron functionality for the application.

####Transformer.js

This class generates the properly formatted JSON used by all other applications from the OPTA XML.  

If this software was to be used for a different sport than soccer this class would be re-written to produce the proper JSON data. 

####FayeService.js

This class is responsible for all communication between the application and the Faye Sever, the SFS.

####JSONFileService.js

This class is responsible for writing JSON data to the correct directory.

####AggregateDataSevice.js

This class receives JSON data from the Transformer.js class and then calculates various required aggregate data JSON files. These files are used by any other aspects of the application that desire aggregate data.

If this software was to be used for a different sport than soccer this class would be re-written to produce the proper JSON data.

####Util.js

The utility functions used throughout the application.

####Email.js, ErrorHandler.js, Logger.js

These are supporting classes and do what their titles describe. For custom behavior for your setup crack open and modify them.

###Dependencies 
The application requires these packages:

**Chockidar:** [https://github.com/paulmillr/chokidar](https://github.com/paulmillr/chokidar)

**XMLDom:** [https://github.com/jindw/xmldom](https://github.com/jindw/xmldom)
 
**Email JS:** [https://www.npmjs.org/package/emailjs](https://www.npmjs.org/package/emailjs)

**Log:** [https://www.npmjs.org/package/log](https://www.npmjs.org/package/log)
 
**Twit:** [https://github.com/ttezel/twit](https://github.com/ttezel/twit)

**MomentJS**: [http://momentjs.com/](http://momentjs.com/)
 
**Kue Job Queue:** [https://www.npmjs.org/package/kue](https://www.npmjs.org/package/kue)

**Redis:** [http://redis.io/](http://redis.io/)
(Mac OS X Install guide: [http://jasdeep.ca/2012/05/installing-redis-on-mac-os-x/](http://jasdeep.ca/2012/05/installing-redis-on-mac-os-x/))
 
**Faye Messaging Server:** [http://faye.jcoglan.com/node.html](http://faye.jcoglan.com/node.html)

**Winston Logger:** [https://github.com/flatiron/winston](https://github.com/flatiron/winston)


##Sports Data Viewer (SDV)
This is the main view server for clients. It is a Node.js application built using the Geddy framework [http://geddyjs.org/](http://geddyjs.org/)

The application currently supports the game soccer with these views:

#####Game view with multiple widgets:
* Soccer Default View (all widgets for a game) http://127.0.0.1:4000/soccer
* Soccer Small Game Widget

#####Individual Widgets:
* Schedule
* Scoreboard
* Timeline
* Commentary
* Starting Lineup


###Configuration
	var config = {
		generatedByVersion: '0.12.12',

		// JSON directory to read from
		jsonDirectory : process.cwd() + '/data/json',
		
		// SFS URL
		'url' : engagementConfig.fayClientURL,
	};

###Suggested Setup Option
The application is blazing fast and depending on your hardware can handle LOAD, but BBG puts this application behind a Varnish server with a TTL of 1 min. Why? Because once the initial HTML is delivered to the client the SFS delivers the HTML updates so there is no more need for 'real time' delivery of information.

##EngamgementConfig.js
This file contains all the configuration options used for different servers. It is not under version control and will need to be created and configured each time the software is deployed. It should be located in the sportsdataprovider/ directory.

	var path = require('path');
	var rootPath = path.normalize(__dirname + '/..');

	module.exports = {
		'applicationMode' : 'dev',
		'competitionName' : 'English Barclays Premier League',
		'competitionID' : '8',
		'season' : 'Season 2014/2015',
		'seasonName' : 'Season 2014/2015',
		'seasonID' : '2014',
		'fayClientURL' : 'http://127.0.0.1:8000/faye',
		'FPTDirectory' : rootPath + '/sportsdataprovider/FTP',
		'squadFileName' : 'srml-8-2014-squads.xml',
		'scheduleFileName' : 'srml-8-2014-results.xml',
	}
The values for the specific 'season' data can be found in the XML set by Opta.

'competitionName' - look in a srml-8-xxxx-fxxxxx-matchresults.xml file for the node 'Name'

'competitionID' - look in a seasonstats-x-xxxx-xxxx.xml file for the node 'SeasonStatistics' and the attribute 'competition_id'.

'season' - look in a srml-8-xxxx-fxxxxx-matchresults.xml file under the node Stat that has the attribute season_name

'seasonName' - look in a srml-8-xxxx-fxxxxx-matchresults.xml file under the node Stat that has the attribute season_name

'seasonID' - look in a srml-8-xxxx-fxxxxx-matchresults.xml file under the node Stat that has the attribute season_id

The squadFileName and scheduleFileName will be the same for the entire engagement.
















