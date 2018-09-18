---
layout: post
title:  "Let's have a conversation"
author: Dave Buckthorpe
tags: [Conversational, Alexa, Hackathon]
---
At the Auto Trader Hack day last year we built a prototype skill to read back stats for a retailer's stock via Alexa Voice Service API using the Amazon Echo Dot. We wanted the user to be able to ask for information about a vehicle, but also take some sort of action after finding out information.  The initial goal was to find out how we could do this and understand what sort of challenges and limitations that we might present.  This is my perspective of the journey before the hack and how we got to showcasing the prototype.

<img src="{{ site.github.url }}/images/2017-04-06/amazon-echo-family.jpg" alt="Amazon Echo and Echo Dot"  width="300" class="u-p-10 u-center-img" />

## Conversational UI
Through a talk by  [Graham Odds](https://2016.nuxconf.uk/speakers/graham-odds/ "Let's Have A Conversation") I first heard about conversational UI and the idea  of messenger apps such as Facebook Messenger, WhatsApp to initiate tasks, something you would do normally using a web interface.  In this specific example he used a bot to make a transaction whilst in the middle of talking to a friend via messenger.  This got me thinking about how this would be useful for other tasks and how we could apply this approach from a dealers point of view, could they find out information by just asking, instead of logging on to an app, would they want to?

I heard about the Amazon Echo Dot functionality through a colleague, who was also thinking about how conversational UI could be used with Auto Trader products, and how this device might change how we might use the internet to complete tasks, similar to bots in the messenger apps. From this we started to think about how we might apply this thinking with some of our products.

## Alexa Toolkit

The Amazon Echo Dot uses the Alexa Skills Kit, which is a collection of self-service API tools.  This makes things it easier to interface with the Alexa Voice Service.

Alexa is the voice service that powers Amazon Echo and Echo Dot and provides the capability for users to interact using their voice.  The skills you build can provide different types of abilities such as:

* Lookup answers to specific questions
* Challenge the user with puzzles or games
* Control lights and other devices in your home
* Provide Audio or text content


## Proof of concept

We wanted to uterlise the data on the dealer's stocklist overview page which adopts a more friendly approach in presenting information.  Initially the first step was to get Alexa to respond with some information that was hosted on an API and then craft a conversation around it so that it's more personal to the user.

I used [json-server](https://github.com/typicode/json-server "json-server") which meant we could test the skill easily with dummy data, below is what I initially started with.


```json
        {
            "reports":[
                {
                    "id":"today",
                    "enquiries":
                    {
                        "total":"50",
                        "unread":"10",
                        "totalcalls":"5",
                        "missedcalls":"4",
                        "answeredcalls":"3",
                        "unansweredcalls":"2",
                        "emails":"10",
                        "partExchangeEmails":"50",
                        "financeEnquiries":"10"
                    }
                }
            ]
        }
```

You can use Node.js, Java, Python or C# to build a skill, I used node and the [alexa app server](https://github.com/alexa-js/alexa-app-server) to be able to test slots, utterances and to check responses before uploading your skill to work with a device.

The things to consider when writing a skill included the following and are outlined in the docs.

* Intents - these represent a list of core actions that user might carry out.
* Utterances - these are words or phrases that the user can use to carry out these intents.  This basically maps to your intents and creates the interactions for the skill.
* A skill name or an 'Invocation Name' that identifies the skill, the user will need to include this name when starting the conversation with the skill.
* A cloud based service that accepts the intents and then acts upon them.
* A configuration that brings all of this together.  In this case I used I created a developer account on the developer portal.



The skill would know what intents to use based on the utterances that the user speaks. You have to setup an Amazon developer account, where you can link to the skill id provided on AWS the setup. There you input the Intent Schema, skill name and utterance combinations, which is also generated in the Alexa app server to test your responses just to makes things easier.  More information about the Intent Schema can be found [here](https://developer.amazon.com/public/solutions/alexa/alexa-skills-kit/docs/alexa-skills-kit-interaction-model-reference#custom-slot-syntax)
 
 The developer account is what allows you to publish your skill.  As this was a proof of concept it was only deployed to the staging server, but it meant we could test the skills on an Amazon Device and when it is setup you should be able to see it linked to your account via the skills tab in the Alexa app.
 
 For the first pass I managed to get the following phrase using the term 'Alexa, ask Portal Info for my report today'.  This is how it is broken down.

The phrase spoken would be something like "{% raw %}Alexa, ask {{skill name}} for my report {{id}}{% endraw %}".  In this case 'portal info' was the skill name and 'today' was the ID which meant we could reference the data easily, defined as a slot name .  The intent contained a slot name and utterances to identify the terms that the user would use to action the intent. 

The response we wanted from Alexa was something like "today, you have 10 unread enquiries and 4 missed calls".

```js
    app.intent('mostPopularVehicle', {
       'slots': {
                "time": "AT_TIME"
        },
        'utterances': [
                '{|what} {|is|was} {|my} most popular vehicle {-|time}'
        ]
        }, function(req, res) {
            //get the slot
            var time = req.slot('time', 'today');
            var reprompt = 'I didn\'t hear that';
            if (_.isEmpty(time)) {
               var prompt = 'I didn\'t hear any information, please specify a type of information you would like';
               res.say(prompt).reprompt(reprompt).shouldEndSession(true);
               return false;
            } else {
               var portalHelper = new PortalDataHelper();
               portalHelper.requestMostPopularVehicle(time).then(function(mostPopularVehicle) {
                   res.session('vehicleId', mostPopularVehicle.vehicleId)
                       .say(portalHelper.formatMostPopularVehicle(time, mostPopularVehicle))
                       .send();
               }).catch(function(err) {
                   console.log(err.statusCode);
                   var prompt = 'I couldn\'t find your most popular vehicle';
                   res.say(prompt).reprompt(reprompt).shouldEndSession(true).send();
               });
               return false;
            }
    })
```

## What we did for the hack

There was three of us on our Team, we wanted to explore the skill further by integrating more with Portal stats and to see if we could manipulate data somehow.

Using our new stock overview page as a reference we started to piece together a story for the user to follow.  First we got Alexa to read back the most popular vehicle for today and yesterday.  We changed the skill name to be "portal" and organised the data so it was easier to use in our code.

We looked at some features in the API which we could use, one feature was the ability to post a message(card) into the Alexa app is relatively straight forward but we found them to be quite plain.

<img src="{{ site.github.url }}/images/2017-04-06/alexa-app-screenshot.jpeg" alt="Alexa app screenshot"  width="300" class="u-p-10 u-center-img" />

Above shows a screenshot of the news feed in the Alexa app, when you ask Alexa about something, then it will post it into your news feed so you can see you can refer back to information if needed.  In this case the weather.

The card is displayed when the user asks for the most popular vehicle.  Not that exciting, but the possibilities to market information in this area then becomes more of an opportunity.  We tried to post an image with the text, but we found we couldn't get this to work despite hosting it an external source, so for the proof of concept we just used a simple title and text.

We also looked at how we could store sessions, so that if the user was to ask about a specific vehicle, they could keep asking about information specific to that vehicle.  It would then be useful for the user to take an action based on the information they find out related to that vehicle.  We know that dealers regularly alter the price when they don't get a lot of response from their adverts so we looked at how we alter the price.

We found we could store the vehicle in a session so we could access the details of the vehicle. So now we had the information, we looked at updating the price which proved to be successful.  So now we had a sequence where we could get the information, and then manipulate a price all through voice.

```js
    app.intent('decreasePrice', {
            'slots': {
                "decrease": "NUMBER",
                "decreasePercent": "NUMBER"
            },
            'utterances': [
                '{reduce|drop} the price by {-|decreasePercent} percent',
                '{reduce|drop} the price by {-|decrease} pounds'
            ]
        }, function(req, res){
            var vehicleId = req.session('vehicleId');
    
            if(!vehicleId){
                res.say("You haven't mentioned a vehicle.  Ask for 'help' for more information on phrases you can use").shouldEndSession(true).send();
                return;
            }
            var portalHelper = new PortalDataHelper();
            portalHelper.requestVehicle(vehicleId).then(function(response){
    
              var newPrice = parseFloat(response.price);
    
              if(req.data.request.intent.slots['decreasePercent'] && req.slot('decreasePercent')){
                newPrice *= (1 - (req.slot('decreasePercent'))/100);
              }
              if(req.data.request.intent.slots['decrease'] && req.slot('decrease')){
                newPrice -= req.slot('decrease');
              }
              var speak = "Okay the new price for your " + response.name + " is £" + newPrice + ", is this okay?";
              res.say(speak)
                .session('priceReductionJourneyStep', 1)
                .session('priceReductionValue', newPrice)
                .shouldEndSession(false)
                .send();
              return false;
            });
            return false;
        }
    );
```


We included some confirmation messages which appeared to work well in a specific session. However, we found it difficult to use as a generic confirmation because of the context and the utterances it would have to align to. We also looked to manipulate a line of text rather than just a value but we found this wouldn't work in the same way, so it needed more investigation. When we tried to update a phrase (e.g. Attention Grabber) we found it only updated it with the first word. We think this limitation was because Alexa would have to know where the end of the spoken phrase/utterance was.


To outline what we did for the hack showcase we created a script to demonstrate the features we investigated. If you would like to see/hear the demo with Alexa responses please feel free to contact me.


```text
       "Alexa, ask Portal what was my most popular vehicle yesterday?"
```

Or 
    
```text
       "Alexa, ask Portal how am I doing?"
```


We can also use the companion App for information that is too dense to speak out, like help


```text
    "Alexa, ask Portal for Help?"
```


We can also have layered conversations, so once the assistant knows which vehicle we are focused on then we can ask for additional information


```text 
    "Alexa, ask Portal what is my most popular vehicle today?"
    
    "How many enquiries on that?"
    
    "And how many daily ad views?"

    "Price?"
    
    "Lets reduce the price by 10 percent"

    "Yes"
```




## Where to next

From an Auto Trader perspective one of the next steps would be to link to either a retailer or consumer account to look at the data points we can access.

The other area we were discussing was around a notifications system, with the idea that we could push alerts and information with relevant information via voice.  We also talked about the possibility of a voice service in the browser and how we could use that to provide additional functionality that some users might find difficult to find. e.g. Alexa, What's the Auto Trader news today,  Alexa, Find me a Pink Ford Fiesta, Alexa, how much would the Tax cost for a Ford Fiesta Turbo.  What if we could just tell the user the information through a voice search.

Overall this was really insightful and opens up a different avenue. If more companies start to integrate voice APIs and if it's done cheaply, we may see a change in attitudes and behaviour and the way we use the Internet to search for information.
