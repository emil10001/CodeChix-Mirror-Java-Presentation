I'm giving a few talks on the Mirror API for Google Glass, and I needed to organize some thoughts on it. I figured that writing a blog post would a good way to do that. 

Here is the slide deck for the shared content of my Mirror talks: (I'll give out the link at the talk)

Since I don't want to miss anything, I'm going to follow the structure of the slides in this post. What I am not going to do, is be consistent with languages. I'll be bouncing between Java and JavaScript, with bits of Android flavored Java thrown in for good measure. It really just comes down to what I have code for right now in which language. The goal here is to discuss the concepts, and to get a good high-level view.

## What is the Mirror API?

### What is it?

The Mirror API is one API available for creating Glassware. The main other option is the GDK. Mirror is REST based, mostly done server-side; it pushes content to Glass, and may listen for Glass to respond. The GDK is client-side, and is similar to programming for Android. The general rule of thumb, is that if you can do what you need to do with Mirror, that's the right choice. Mirror is a bit more constrained, and is likely to deliver a good user experience, without a lot of effort. It also allows Glass to manage execution in a battery friendly way.

### What does it do?

Mirror handles push messages from servers, and responds as necessary to them. The basic case is that your server pushes a message to Glass, then the message displayed on Glass as a Timeline Card.

### How does it work?

If you're coming from the Android world then you might be familiar with GCM, and push messages. Mirror works using GCM, similar to how Android uses GCM. 

You can send messages to Glass, and receive messages from Glass. Sending messages is much easier than receiving messages. On the sending side, all you need is a network connection, and the ability to do the OAuth dance. If you want to receive data from Glass, you'll also need some way for Glass to reach you, via a callback URL. 

### What’s the flow?

Glass registers with Google's servers, and then your server sends requests to Google's servers, and Google's servers communicate with Glass. You can send messages to Google's servers, and you can receive messages back from Glass via callback URLs that you provide. With that, you can get a back and forth going. 

### SO MANY QUESTIONS?!?!?!

I know, there's a lot here. We'll try to step through as much as possible in this post.

## Setting up

Before you do anything else, you'll need to head over to the [Google Developer Console](https://cloud.google.com/console/project), and register your app. Create a new app, turn on 'Mirror API', and turn off everything else. Then, go into the Credentials section, so that you can get your Client Id and Client Secret. This is outlined pretty clearly in the [Glass Quick Start documentation](https://developers.google.com/glass/develop/mirror/quickstart/java).

## OAuth

To communicate with Glass, you'll first need to do do the OAuth dance. The user visits your website, and clicks a link that will ask them to authenticate with one of their Google accounts. It then asks for permission to update their Mirror Timeline. After that's done, you'll receive a request at the callback URL, with a code. The code needs to be exchanged for a token. Once you have the token, you can begin making authenticated requests. The token will expire, so you'll need to be able to re-request it when it does.

Further reading in the [Quick Start](https://developers.google.com/glass/develop/mirror/quickstart/java).

## Timeline

The timeline is the collection of Cards that you have on Glass, starting from the latest, and going back into the past. It also includes pinned cards, and things happening 'in the future', provided by Google Now.

[Here's a video explanation from the Glass Team](http://youtu.be/qfs5d00TNrA).

### Why a Timeline?

Well, you'd have to ask the Glass team, why exactly a Timeline, but I could venture a guess. One of Glass's issues is that it is more difficult to navigate with. You can't dive into apps like you can on your phone, at least not quickly and easily. There's no 'Recent Apps' section, no home screen and no launcher (well, except from the voice commands). 

Launching apps with voice commands allows you to _do_ things, as opposed to going to an app and poking around. For example, saying `OK Glass, start a bike ride` launches Strava into a mode where it immediately begins tracking me with GPS and giving me feedback on that tracking. It does not let me look at my history there. 

Glass tends to operate on push rather than pull. Glassware surfaces information to you, as it becomes relevant. This ability allows Glass to become incredibly contextual, though we have only seen bits of this so far with Google Now. Imagine a contextual Glassware that keeps a todo list of all of your tasks, both work and home. It could surface a Card when you get to your office with the highest priority work tasks that you have for that day. On the weekend, after you're awake, it could tell you that you need to get your car into the shop to get your smog check. 

A typical category of Glassware today is news, which all work well with a push approach. Now, this is not terribly useful on Glass, as it's difficult to consume, but there are some good ones that have figured out how to deliver the content in a way that is pleasant to consume on Glass (Unamo, and CNN), or to allow you to save it off to Pocket for picking up later (Winkfeed).

The timeline is semi-ephemeral. Your cards will stick around for 7 days, and will be lost after that. Before you get all clingy and nostalgic, consider that that information is 7 days old, and will not only be stale, but difficult to even scroll to at that point. I have rarely gone back more than a day in my history to dig for something that I saw, and that was usually for a photo. Cards are usually a view into a service, so the information is rarely lost to the ether. Instead of scrolling for a solid minute trying to find something, you can pull out your phone and go to the app on your phone. Like G+ Photos, Gmail or Hangouts. Apps like Strava don't even put your recent rides in your timeline, you have to either go to the app or their website for history.

### What are Cards?

Cards are the individual items in the timeline. Messages that you send from your server to Glass users typically end up as timeline cards on the user's Glass. There are a number of basic actions, as well as custom actions, that users can take on cards, depending on what the card supports. How this works will become apparent once we look at the code.

### How do I insert Cards into the Timeline?

Using the API, of course! Let's look at how to do this in a couple of languages.

Java:

    TimelineItem timelineItem = new TimelineItem();
    timelineItem.setText("Hello World");
    Mirror service = new Mirror.Builder(new NetHttpTransport(), new JsonFactory(), null).setApplicationName(appName).build();
    service.timeline().insert(timelineItem).setOauthToken(token).execute();

JavaScript:

    client
        .mirror.timeline.insert(
        {
            "text": "Hello world",
            "menuItems": [
                {"action": "DELETE"}
            ]
        }
    )
    .withAuthClient(oauth2Client)
    .execute(function (err, data) {
        if (!!err)
            errorCallback(err);
        else
            successCallback(data);
    });

Further reading in the [Developer Guides](https://developers.google.com/glass/develop/mirror/timeline), and [Reference](https://developers.google.com/glass/v1/reference/timeline).

## Locations

Location with Glass can mean one of two things, either you're pushing a location to Glass, so that the user can use it to navigate somewhere. Or, you're requesting the user's location from Glass. The Field Trip Glassware subscribes to the user's location so that it can let the user know about interesting things in the area.

### Sending Locations to Glass

Sending locations is very simple. Basically, you send a timeline card with a location parameter, and add the `NAVIGATION` menu item.

JavaScript:

    client
        .mirror.timeline.insert(
        {
            "text": "Let's meet at the Hacker Dojo!",
            "location": {
                "kind": "mirror#location",
                "latitude": 37.4028344,
                "longitude": -122.0496017,
                "displayName": "Hacker Dojo",
                "address": "599 Fairchild Dr, Mountain View, CA"
            },
            "menuItems": [
                {"action":"NAVIGATE"},
                {"action": "REPLY"},
                {"action": "DELETE"}
            ]
        }
    )
        .withAuthClient(oauth2Client)
        .execute(function (err, data) {
            if (!!err)
                errorCallback(err);
            else
                successCallback(data);
        });

I'm using this right now for something very simple. In a Mirror/Android app that I'm working on, I can share locations from Maps to Glass, and then use Glass for navigation. Nothing is more annoying than having Glass on and then using your phone for navigation because the street is impossible to pronounce correctly.

### Getting Locations from Glass

This is a little more tricky. First, you need to add a permission to your scope, `https://www.googleapis.com/auth/glass.location`. Getting locations back from Glass is going to require you to have a real server that is publicly accessible. It should also support SSL requests if you're in production. 

Here's some code from the Java Quick Start that deals with handling location updates:

    LOG.info("Notification of updated location");
    Location location = MirrorClient.getMirror(credential).locations().get(notification.getItemId()).execute();

    LOG.info("New location is " + location.getLatitude() + ", " + location.getLongitude());
    MirrorClient.insertTimelineItem(
        credential, 
        new TimelineItem().setText("Java Quick Start says you are now at " 
            + location.getLatitude()
            + " by " + location.getLongitude())
        .setNotification(new NotificationConfig().setLevel("DEFAULT"))
        .setLocation(location).setMenuItems(Lists.newArrayList(new MenuItem().setAction("NAVIGATE"))));

This will really help enable contextual Glassware. You could imagine Starbucks Glassware that offers you a discounted coffee when you get near a Starbucks. Or a Dumb Starbucks Glassware that does the same thing, but claims to be an art installation.

Further reading in the [Developer Guides](https://developers.google.com/glass/develop/mirror/location), and [Reference](https://developers.google.com/glass/v1/reference/locations).

## Menus

Menus are the actions that a user can take on a card. They make cards interactive. The most basic one is `DELETE`, which, obviously, allows the user to delete a card. Other useful actions are `REPLY`, `READ_ALOUD`, and `NAVIGATE`. 

You can see menu items added to cards in previous examples. I will repeat the JavaScript navigation example here:

JavaScript:

    client
        .mirror.timeline.insert(
        {
            "text": "Let's meet at the Hacker Dojo!",
            "location": {
                "kind": "mirror#location",
                "latitude": 37.4028344,
                "longitude": -122.0496017,
                "displayName": "Hacker Dojo",
                "address": "599 Fairchild Dr, Mountain View, CA"
            },
            "menuItems": [
                {"action":"NAVIGATE"},
                {"action": "REPLY"},
                {"action": "DELETE"}
            ]
        })
        .withAuthClient(oauth2Client)
        .execute(function (err, data) {
            if (!!err)
                errorCallback(err);
            else
                successCallback(data);
        });

The Card generated from the example should allow you to get directions to the Hacker Dojo. You can also add custom menus, like Winkfeed's incredibly useful `Save to Pocket` menu item.

Further reading in the [Developer Guides](https://developers.google.com/glass/develop/mirror/menu-items), and [Reference](https://developers.google.com/glass/v1/reference/timeline).

## Subscriptions

Subscriptions allow you to get data back from the user. This can happen automatically, if you're subscribed to the user's locations, or when the user takes some action on a card that is going to interact with your service. E.g., you can subscribe to `INSERT`, `UPDATE`, and `DELETE` notifications. You will also get notifications if you support the user replying to a card with `REPLY`. 

In Java:

    // Subscribe to timeline updates
    MirrorClient.insertSubscription(credential, WebUtil.buildUrl(req, "/notify"), userId, "timeline");

Subscriptions require a server, and cannot be done with localhost. However, [tools like ngrok](https://ngrok.com/) may be used to get a publicly routable address for your localhost. I have heard that ngrok does not play nice with Java, but you might be able to find another tool that does. 

Among other things, you can learn from these sorts of subscriptions. You can learn which types of Cards get deleted the most, for example. Subscriptions can also listen for custom menus, voice commands and locations. With Evernote, you can say, `take a note` and it will send the text of your note to the Evernote Glassware. 

Further reading in the [Developer Guides](https://developers.google.com/glass/develop/mirror/subscriptions), and [Reference](https://developers.google.com/glass/v1/reference/subscriptions).

## Contacts

Another important type of subscription is subscribing for `SHARE` notifications. The `SHARE` action also involves Contacts.

Contacts are endpoints for content. They can be Glassware or people, though if they're people, they are probably still really some sort of Glassware that maps to people. Sharing to Contacts allow your Glassware to receive content from other Glassware. You can insert a Contact into a user's time like so (in JavaScript):

    client
        .mirror.contacts.insert(
        {
            "id": "emil10001",
            "displayName": "emil10001",
            "iconUrl": "https://secure.gravatar.com/avatar/bc6e3312f288a4d00ba25500a2c8f6d9.png",
            "priority": 7,
            "acceptCommands": [
                {"type": "POST_AN_UPDATE"},
                {"type": "TAKE_A_NOTE"}
            ]
        })
        .withAuthClient(oauth2Client)
        .execute(function (err, data) {
            if (!!err)
                errorCallback(err);
            else
                successCallback(data);
        });

It's obvious what you want people Contacts for, but what about Glassware contacts? Well, imagine sharing an email with Evernote, or a news item with Pocket. Now, consider that as the provider of the email or news item, you don't need to provide the Evernote or Pocket integration yourself, you just need to allow sharing of that content. It's a very powerful idea, analogous to Android's powerful sharing functionality.

Further reading in the [Developer Guides](https://developers.google.com/glass/develop/mirror/contacts), and [Reference](https://developers.google.com/glass/v1/reference/contacts).

## Code Samples

I have three sample projects that I have been pulling code from for this post. 

* [A slightly modified Java Quick Start](https://github.com/emil10001/mirror-quickstart-java) provided by the Glass Team
* My [Android Mirror demo](https://github.com/emil10001/GlassFromAndroid)
* My [node.js Mirror demo](https://github.com/emil10001/glass-mirror-nodejs-auth-demo)
