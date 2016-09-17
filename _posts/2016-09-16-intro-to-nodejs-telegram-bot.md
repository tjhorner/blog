---
title: "Intro to Node.js: Making a Telegram Bot"
layout: post
author: TJ Horner
summary: In this workshop, we will create a simple Telegram bot that tells us how long until CodeDay ends using Node.js, the Telegram API, and the Clear API.
permalink: /post/intro-to-node-telegram-bot
---

In this workshop, we will create a simple Telegram bot that tells us how long
until CodeDay ends using Node.js, the Telegram API, and the Clear API.

_This post is intended to serve as a guide for my CodeDay workshop, but if you’d
like to use this to make your own Telegram bot with Node.js, go right ahead :)_

# Part I: Chat bot

Telegram has two types of bots: chat bots and inline bots. Chat bots use commands
(such as **/start**, **/help**, etc.) to interact with users. Inline bots can instead be
called from anywhere and help users find and send content to any chat they wish.
This section covers chat bots.

### Step 0.1: Acquire Node.js and npm

You should have already done this before the event, but it’s a pretty small download
if you still need to get it. Here are links to the version of Node.js we will be
using for this workshop:

- **Windows**: https://nodejs.org/dist/v6.6.0/node-v6.6.0-x64.msi
- **macOS**: https://nodejs.org/dist/v6.6.0/node-v6.6.0.pkg
- **Linux**: You should already know how to get Node.js.

### Step 0.2: Get Telegram

Since we’re making a Telegram bot, you’ll need a Telegram account. Head over to
[telegram.org/dl](https://telegram.org/dl) and create your account.

### Step 1: Set up your project directory

Now that we have Node.js and npm installed and our Telegram account is ready, we
can start with the project. Open up your preferred terminal and run these commands:

```bash
mkdir codeday-telegram-bot
cd codeday-telegram-bot
```

Now we have a directory for our Telegram bot, but we need a package.json, this
tells npm information about our project. To do this, (in the same terminal window)
we need to run this:

```bash
npm init
```

This command will tell npm to initialize a pre-made package.json in our project
directory. Just hit enter at all of the prompts, we can worry about those later.

### Step 2: Install necessary packages

For this project, we will only need one npm package. An npm package is essentially
a piece of code (“package”) in the npm registry that we can download with a simple
command, `npm install`.

We’re going to use a package called `node-telegram-bot-api`, we can install it like so:

```bash
npm install --save node-telegram-bot-api
```

The `--save` part signals npm to save this package to our package.json so it is
easily installed later, along with any other dependencies that our package may need.

### Step 3: Get a Telegram bot token

Using Telegram, contact [@botfather](https://telegram.me/Botfather) and use the **/newbot**
command to create a new bot. Name it whatever you’d like with any username, it doesn’t matter.

### Step 4: Write some code!

Now that we have our bot token, we can use it to interface with the Telegram Bot API.

Create an index.js file in your project directory, then open it in your favorite
text editor. Here’s some simple “Hello world” boilerplate code I’ve written to
assist with this step:

```javascript
var TelegramBot = require('node-telegram-bot-api'),
    // Be sure to replace YOUR_BOT_TOKEN with your actual bot token on this line.
    telegram = new TelegramBot("YOUR_BOT_TOKEN", { polling: true });

telegram.on("text", (message) => {
  telegram.sendMessage(message.chat.id, "Hello world");
});
```

Essentially, what this code does is include the library we downloaded from npm earlier,
then instantiated a new `TelegramBot` object with our token and told it to poll for
updates from Telegram. Then it will listen for the “text” event (which, in this library,
is a simple text message) and then send a message to the chat the original message was
saved saying “Hello world”.

Now, run the index.js file with this command:

```bash
node index
```

Send your bot a message. It should reply with “Hello world”. If it does, awesome!
If it doesn’t, ask me (if you're attending this workshop in person, that is).

![It works!]({{ site.url }}/assets/telegram-bot/hello-world.png)

### Step 5: Implement a simple command

In its current state, this bot is not very useful. Let’s make it useful, how about
by adding a **/codeday** command that tells us when CodeDay ends! We will be utilizing
two more libraries here, **codeday-clear** and **moment**. **codeday-clear** is the official
Node.js library for interacting with the Clear API. Clear is our internal CodeDay
management system and has a very nice API. We will be using this API to get the
ending date of CodeDay and then use **moment** to convert it into a human-readable format.

To install these libraries:

```bash
npm install --save codeday-clear moment
```

In our code, we then need to include these two libraries and instantiate a `Clear`
instance. Since the Clear API requires access to Clear and an API token/secret
pair, I have created a sample app with limited permissions for the purpose of this
workshop:

- **Token**: 1YZiGaj3baaLU8IKVsASRIWaNF2oJNg0
- **Secret**: 1COMnWyGnGBsNqkhaZ6WMBWB9UWZw6QZ

Here’s what it should look like:

```javascript
var Clear = require('codeday-clear'),
    // Our sample app token and secret
    clear = new Clear("1YZiGaj3baaLU8IKVsASRIWaNF2oJNg0", "1COMnWyGnGBsNqkhaZ6WMBWB9UWZw6QZ");

// moment is not a class, just a simple function
var moment = require('moment');
```

This code should go right after where we declare the `telegram` instance (this should be line 3 if you are following along). Now that we have our Clear and moment instances declared, we can use them in a simple command. Let’s replace the code inside of the “text” event with something like this:

```javascript
telegram.on("text", (message) => {
  if(message.text.toLowerCase().indexOf("/codeday") === 0){
    clear.getEventById("oo4QIuKQQTYA", (codedayEvent) => {
      var endsAt = moment(codedayEvent.ends_at * 1000);
      telegram.sendMessage(message.chat.id, "CodeDay ends " + endsAt.fromNow() + "!");
    });
  }
});
```

Here’s a breakdown of what’s happening here:

- We check with the JavaScript `indexOf` function if the message starts with /codeday
- If it does, we tell the Clear library to grab the event with ID oo4QIuKQQTYA (that’s San Diego Fall 2016’s ID)
- We use a callback function once it gets the event, and then parse the `ends_at` date with moment. (We multiply it by 1000 to convert it to milliseconds (which moment understands), since Clear sends this date in a UNIX timestamp format.)
- We then use moment’s `fromNow` function to get the relative time (e.g. “in 10 hours”) of the `endsAt` date and then send that all through the Telegram API.

![Since I wrote this post two months before CodeDay, this happened…]({{ site.url }}/assets/telegram-bot/simple-command.png)

### Step 6: Getting fancy with Markdown

Telegram supports messages sent with **Markdown**, a simple way to format text. Let’s
make the end date bold as an example.

In Telegram’s version of Markdown, we will surround the text we want to bold with
asterisks, so the raw message we send to Telegram should look something like this:

> CodeDay ends \*in 2 months\*!

To implement this, it’s rather simple; all we need to do is surround the end date
in asterisks and then pass the `parse_mode` parameter to the Telegram API and set
it to “Markdown”, this will tell the Telegram API that we want the text to render
as Markdown.

The modified code could look something like this:

```javascript
telegram.sendMessage(message.chat.id, "CodeDay ends *" + endsAt.fromNow() + "*!", {
  parse_mode: "Markdown"
});
```

Now when we run the **/codeday** command, it will look like this instead:

![Fancy!]({{ site.url }}/assets/telegram-bot/markdown.png)

Telegram also supports italics, inline code, code blocks, and inline URLs along with the bold option we just used.

# Part II: Inline bot

This section will cover how to turn this bot into an inline bot, which can be called
from anywhere and help users find and send content from your bot.

### Step 0: Enable inline mode in Botfather

Bots don’t have inline mode enabled by default, so you need to send the **/setinline**
command to Botfather to enable it. You can use anything as your placeholder text,
but I used “Search CodeDays…” as mine.

![]({{ site.url }}/assets/telegram-bot/botfather-setinline.png)

After you enable inline mode, you may need to restart whatever Telegram client you’re
using for the changes to take effect (since they’re cached). Once you restart,
type your bot’s username then a space. If you see the placeholder text you set,
you’re ready for the next step.

### Step 1: Implement inline queries

Now we need to tell our script how to react to inline queries. To do this, here’s
some boilerplate “Hello world” code I’ve written to simplify the process:

```javascript
telegram.on("inline_query", (query) => {
  telegram.answerInlineQuery(query.id, [
    {
      type: "article",
      id: "testarticle",
      title: "Hello world",
      input_message_content: {
        message_text: "Hello, world! This was sent from my super cool inline bot."
      }
    }
  ]);
});
```

Here’s how it works:

- Upon an inline query…
- Respond with an array with 1 “article” result…
- Which has a title of “Hello world” and sends the message “Hello, world! This was sent from my super cool inline bot.”

### Step 2: Implement Clear API

Now let’s actually make it show the CodeDay events. We need to iterate over every
event and turn them into an object the Telegram API understands, so you could have
something like this:

```javascript
telegram.on("inline_query", (query) => {
  var searchTerm = query.query.trim();

  clear.getRegions((regions) => {
    var queryResults = [ ];

    regions.forEach((region) => {
      if(region.name.toLowerCase().indexOf(searchTerm.toLowerCase()) !== -1 && region.current_event && region.current_event.venue){
        queryResults.push({
          type: "article",
          id: region.id,
          title: "CodeDay " + region.name,
          description: "Hosted at " + region.current_event.venue.full_address,
          input_message_content: {
            latitude: region.location.lat,
            longitude: region.location.lng,
            title: "CodeDay " + region.name,
            address: region.current_event.venue.full_address
          }
        });
      }
    });

    telegram.answerInlineQuery(query.id, queryResults);
  });
});
```

So what we’re doing here is…

- Upon an inline query…
- Ask Clear for a list of events…
- Then turn those events into objects the Telegram API can understand…
- And make the message content the venue the CodeDay is at.
- Add the result to the results array.
- Send the results back to the API.

# Part III: Conclusion

Now that you know the basics of bot creation, I encourage you to play around with
Telegram’s bot API and make something awesome with it! You can make games, tools,
calculators(?)… anything, really. Go check out the docs and see what you can build!
https://core.telegram.org/bots/api

**And remember, have fun!**
