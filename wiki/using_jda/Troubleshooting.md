# This wiki page is migrating to [jda.wiki/using-jda/troubleshooting](https://jda.wiki/using-jda/troubleshooting/)

***

# Troubleshooting

This is a collection of common issues and recommended solutions.

## Table of Contents

- [Shutdown but the process doesn't exit](#shutdown-but-the-process-doesnt-exit)
- [NoClassDefFoundError or ClassNotFoundException on startup](#noclassdeffounderror-or-classnotfoundexception-on-startup)

### Discord Issues and API Limitations

- [The provided token is invalid!](#the-provided-token-is-invalid)
- [Can't get emoji from message](#cant-get-emoji-from-message)

### Event Handling and RestActions

- [RestAction queue returned failure](#restaction-queue-returned-failure)
- [Nothing happens when using X](#nothing-happens-when-using-x)
- [My event listener code is not executed](#my-event-listener-code-is-not-executed)
- [Missed 2 heartbeats! Trying to reconnect...](#missed-2-heartbeats-trying-to-reconnect)
- [Listener must implement EventListener](#listener-must-implement-eventlistener)
- [I can't get the previous message content from delete/update](#i-cant-get-the-previous-message-content-from-deleteupdate)
- [Preventing use of complete() in callback threads](#preventing-use-of-complete-in-callback-threads)

### RateLimits

- [Hit the WebSocket RateLimit](#hit-the-websocket-ratelimit)
- [Encountered 429 or Encountered global rate limit](#encountered-429-or-encountered-global-rate-limit)

### Intents and Caching

- [Cannot get reference as it has already been Garbage Collected](#cannot-get-reference-as-it-has-already-been-garbage-collected)
- [Users/Members not in cache](#usersmembers-not-in-cache)
- [I'm getting CloseCode(4014 / Disallowed intents...)](#im-getting-closecode4014--disallowed-intents)

### Interactions and Slash Commands

- [This interaction failed / Unknown Interaction](#this-interaction-failed--unknown-interaction)
- [Interaction Followup Messages Timed out](#interaction-followup-messages-timed-out)

Not find an answer? Try asking in [our discord server](https://discord.gg/0hMr4ce0tIl3SLv5)

-----------------

### Shutdown but the process doesn't exit

When you call `JDA.shutdown()` or `JDA.shutdownNow()` the JDA instance will stop all of its threads. However, if HTTP/2 was used by the `OkHttpClient` instance it will keep the JVM running due to a timeout thread for http connections. This can be terminated by shutting it down manually:

```java
OkHttpClient client = jda.getHttpClient();
client.connectionPool().evictAll();
client.dispatcher().executorService().shutdown();
```

### The provided token is invalid!

```
javax.security.auth.login.LoginException: The provided token is invalid!
```

This exception indicates that the token you have used in your `JDABuilder` is not a valid bot token.
Usually, this means you tried using the **secret** instead of the bot token. To get your token, follow these steps:

1. Open the [Application Dashboard](https://discord.com/developers/applications)
1. Select your application
1. On the left side, click the **Bot** tab
1. If you don't have a bot yet, you must create one
1. Once you have a bot, there is a token section. Click **COPY**.
1. The token is now in your clipboard and you can paste it into your code

If you follow these steps and you still get the same exception, it could be due to one of these problems:

- You included excess whitespace in your string. The token string should not include any newlines or spaces.
- You were banned from the API or your server is hosted on a public hosting platform like Glitch or Heroku.
- The token is not for a bot account, we do not support client accounts.

A valid token looks like this:

```
NDkyNzQ3NzY5MDM2MDEzNTc4.Xw2cUA.LLslVBE1tfFK20sGsNm-FVFYdsA
```

**NEVER SHARE YOUR TOKEN WITH ANYONE. DO NOT COMMIT IT AND PUSH IT TO GITHUB. DO NOT SHOW IT TO ANYONE UNDER ANY CIRCUMSTANCES.**

### Nothing happens when using X

In JDA we make use of async rate-limit handling through the use of the common [[RestAction|7)-Using-RestAction]] class.
<br>When you have code such as `channel.sendMessage("hello");` or `message.delete();` nothing actually happens.
This is because both `sendMessage(...)` as well as `delete()` return a `RestAction` instance. You are not done here since that class is only an intermediate step to executing your request. Here you can decide to use async `queue()` (recommended) or `submit()` or the blocking `complete()` (not recommended).

You might notice that `queue()` returns `void`. This is because it's **async** and uses callbacks instead.
[[Read More|7)-Using-RestAction]]

If you *do* have a `queue()` then maybe your code doesn't even run? Try putting a `System.out.println("debug")` right before and after your code and see if it prints. If not, then read this [My event listener code is not executed](#my-event-listener-code-is-not-executed).

### RestAction queue returned failure

When JDA encounters an issue while executing a `RestAction` it will emit an error through the **failure callback**. You can handle this by adding a second callback to `queue()`, for example: `message.delete().queue(v -> System.out.println("success"), ContextException.herePrintingTrace​());`.

Example:

```java
public void deleteMessage(Message message) {
    message.delete().queue(null, (exception) -> {
        message.getChannel().sendMessage("There was an error " + exception).queue();
    });
}
```

You can use [ErrorHandler](https://ci.dv8tion.net/job/JDA/javadoc/net/dv8tion/jda/api/exceptions/ErrorHandler.html) to handle or ignore specific [ErrorResponse](https://ci.dv8tion.net/job/JDA/javadoc/net/dv8tion/jda/api/requests/ErrorResponse.html) failures.

### My event listener code is not executed

There are many reasons why your event listener might not be executed but here are the most common issues:

1. You are using a deprecated part of JDA? Such as `new JDABuilder(...)`
    <br>Use the replacement that is documented. For example `createDefault(token)`
1. You are using the wrong login token?
    <br>If the token is for another bot which doesn't have access to the desired guilds then the event listener code cannot run.
1. Your bot is not actually in the guild?
    <br>Make sure your bot is online and has access to the resource you are trying to interact with.
1. You never registered your listener? 
    <br>Use `jda.addEventListener(new MyListener())` on either the `JDABuilder` or `JDA` instance
1. You did not override the correct method?
    <br>Use `@Override` and see if it fails. Your method has to use the correct name and parameter list defined in `ListenerAdapter`. [[Read More|1)-Events]].
1. You don't actually extend `EventListener` or `ListenerAdapter`.
    <br>Your class should **either** use `extends ListenerAdapter` or `implements EventListener`.
1. You are missing a required [`GatewayIntent`](https://github.com/DV8FromTheWorld/JDA/wiki/Gateway-Intents-and-Member-Cache-Policy) for this event.
    <br>Make sure that you `enableIntents(...)` on the `JDABuilder` to allow the events to be received.
1. The event has other requirements that might not be satisfied such as the cache not being enabled.
    <br>Please check the requirements on the event documentation.

If none of the above apply to you then you might have an issue in your listener's code, at that point you should use a debugger.

### This interaction failed / Unknown Interaction

This means you didn't acknowledge or reply to an interaction in time. You only have **3 seconds** to reply or acknowledge.
You have to use `event.deferReply().queue()`, `event.deferEdit().queue()`, `event.editMessage(...).queue()`, or `event.reply(...).queue()`. (If you don't `queue()` it won't do it)
<br>**Make sure your event listener code is executed.**

### Interaction Followup Messages Timed out 

This means you sent followup messages through `InteractionHook.sendMessage(...)` or similar but never acknowledged the interaction.


### Cannot get reference as it has already been Garbage Collected

Due to how we structure cache we sometimes have to invalidate our entire cache (that's just how discord works).
When you store references to JDA entities for a long period of time such as a field you will suffer with the error `java.lang.IllegalStateException: Cannot get reference as it has already been Garbage Collected` once the entity was removed from the JDA cache. We highly recommend to store only the parts you actually need of the specific entity such as `id` and use something like `event.getJDA().getRoleById(id)`.

Entities that should not be stored for a long period of time include:

- Role
- Channel (any type of channel)
- Guild
- Emote
- User
- Message

Instead store IDs of the entities, or for messages simply the parts you need such as content.

### Hit the WebSocket RateLimit

When you update your game or online status you emit a socket message to discord. If you do that often enough you hit a limit and JDA has to backoff for 60 seconds.

Things that contribute to the WebSocket RateLimit include:

- `AudioManager.openAudioConnection(...)`
- `AudioManager.closeAudioConnection()`
- `AudioManager.setSelfMuted(...)`
- `AudioManager.setSelfDeafened(...)`
- Any setter method on `Presence`.

It is also possible that you get spammed by this warning if you use `ChunkingFilter.ALL` (this is done when using `create(token, intents)`). If your bot is in more than 120 guilds then this warning is unavoidable when using member chunking. It is recommended to use `setChunkingFilter(ChunkingFilter.NONE)` to reduce the startup time and get rid of this warning. If chunking on startup is absolutely necessary, you have to accept this warning.

There are many ways to retrieve members: [Loading Members](https://github.com/DV8FromTheWorld/JDA/wiki/Gateway-Intents-and-Member-Cache-Policy#loading-members)

I explained this in a bit more detail in issue [#1290](https://github.com/DV8FromTheWorld/JDA/issues/1290)

### Missed 2 heartbeats! Trying to reconnect...

This warning implies your event thread is too busy and will block critical events from being received. You should try to limit blocking calls and make sure your event handlers don't take up too much time. Do profiling to figure out what takes so long or create a [thread dump](https://github.com/DV8FromTheWorld/JDA/wiki/10\)-FAQ#how-do-i-make-a-thread-dump) when you get this warning to see where the issue is.

By default, all events are handled on the same thread they get received and handled on. If you block this thread for too long then JDA cannot keep up with important lifecycle events sent by discord. Either you start writing non-blocking code (replace `complete()` with `queue()` etc.) or you use a thread pool for your event handling.

### Users/Members not in cache

The default behavior in `createDefault` is to only cache members connected to voice channels.
If you need members to be cached, for example to lookup users by roles, then you have to enable this explicitly.

I explained this in [this wiki page](https://github.com/DV8FromTheWorld/JDA/wiki/Gateway-Intents-and-Member-Cache-Policy) and [this stackoverflow answer](https://stackoverflow.com/a/61229594/10630900).

There are many ways you can retrieve members dynamically: [Loading Members](https://github.com/DV8FromTheWorld/JDA/wiki/Gateway-Intents-and-Member-Cache-Policy#loading-members)

### Listener must implement EventListener

```none
Exception in thread "main" java.lang.IllegalArgumentException: Listener must implement EventListener
        at net.dv8tion.jda.api.hooks.InterfacedEventManager.register(InterfacedEventManager.java:62)
        at net.dv8tion.jda.internal.hooks.EventManagerProxy.register(EventManagerProxy.java:52)
        at net.dv8tion.jda.internal.JDAImpl.addEventListener(JDAImpl.java:810)
        at net.dv8tion.jda.api.JDABuilder.lambda$build$0(JDABuilder.java:1841)
        at java.base/java.lang.Iterable.forEach(Iterable.java:75)
        at net.dv8tion.jda.api.JDABuilder.build(JDABuilder.java:1841)
```

When you get an exception like this, that means one of the event listeners you registered does not implement the `EventListener` interface provided by JDA.

This is not a valid event listener class:

```java
public class MyListener {
   ...
}
```

You can either use **ListenerAdapter** or **EventListener**:

```java
import net.dv8tion.jda.api.hooks.ListenerAdapter;

public class MyListener extends ListenerAdapter {
  ...
}
```

When using **EventListner** make sure you actually imported the correct interface from JDA and **not** `java.util.EventListner`:

```java
import net.dv8tion.jda.api.events.GenericEvent;
import net.dv8tion.jda.api.hooks.EventListener;

public class MyListener extends EventListener {
  @Override
  public void onEvent(GenericEvent event) {
    ...
  }
}
```

[Read More](https://github.com/DV8FromTheWorld/JDA/wiki/1%29-Events)


### I can't get the previous message content from delete/update

When discord emits a `message_delete` or `message_update` they only provide the new content of the message. Since JDA does not keep a cache of messages it is unable to provide the previous content. Instead you will have to track content of messages yourself.

### Can't get emoji from message

Methods such as [`Message.getEmotes()`](https://ci.dv8tion.net/job/JDA/javadoc/net/dv8tion/jda/api/entities/Message.html#getEmotes()) and [`Message.getEmotesBag()`](https://ci.dv8tion.net/job/JDA/javadoc/net/dv8tion/jda/api/entities/Message.html#getEmotesBag()) only include custom emoji which have to be uploaded to a guild by a moderator. Unicode emoji such as 👍 are not included and require using a 3rd party library to be located in a string. You can use [emoji-java](https://github.com/vdurmont/emoji-java) to extract unicode emoji from a message.

An example use-case including a code sample can be found in my answer to a related question on StackOverflow: https://stackoverflow.com/a/58353912/10630900

### Preventing use of complete() in callback threads

The following code will illustrate an issue where callbacks might cause a deadlock

```java
class Main {
    public static main(String[] args) {
        JDA api = JDABuilder.createDefault(BOT_TOKEN)
                .setCallbackPool(Executors.newSingleThreadScheduledExecutor()) // (1)
                .build().awaitReady();
        TextChannel channel = api.getTextChannelById(CHANNEL_ID);
        channel.sendMessage("hello there").queue((message) -> { // (2)
            System.out.println("Hello");
            message.editMessage("general kenobi").complete(); // (3) deadlock
            System.out.println("World!!!!"); // never printed
        });
   }
}
```
> You can test this yourself on **3.8.0** and see it fail.

Since we decided to use a single-thread pool (1) we only have one thread to execute callbacks.
This thread is used by the first callback (2) and cannot be used for the second callback (3).

Due to this reason we simply don't allow using `complete()` in any callback threads at all. If you use callbacks you should use `queue()`.

### NoClassDefFoundError or ClassNotFoundException on startup

An error like `java.lang.NoClassDefFoundError: net/dv8tion/jda/api/JDABuilder` or similar means you are not including your dependencies or transitive dependencies in the archive.

1. Gradle (build.gradle)
    <br>With gradle this can be fixed by using the [shadow plugin](https://github.com/johnrengelman/shadow) and building your jar with `shadowJar` instead. The jar will then be present in the `build/libs` directory with a name like `example-1.0-all.jar`

2. Maven (pom.xml)
    <br>With maven you need the [shade plugin](https://maven.apache.org/plugins/maven-shade-plugin/) in your pom to add dependencies to your package task. You can see the shade plugin being applied in this [example pom.xml](https://gist.github.com/MinnDevelopment/5d8c5965043bbe5315d47b690cd7a4d9)

3. Jar
    <br>You need to use the `-withDependencies.jar` rather than the normal one.

### Encountered 429 or Encountered global rate limit

When the internal jda rate-limiter fails to predict a rate limit bucket the HTTP response is `429: TOO MANY REQUESTS`. This means the request has to be retried. If you see this a lot (many times per minute), then JDA might have an issue with the rate limit handling of that route. If you use `setRelativeRateLimit(false)` it could also mean that your clock is not properly synchronizing with NTP.

Encountering the global rate-limit is something JDA cannot predict or prevent. This rate-limit implies you sent too many requests in total across all routes. Discord limits how much HTTP traffic a client is allowed to do and will tell us to limit all requests for a specified time interval. You should avoid hitting this too often.

### I'm getting CloseCode(4014 / Disallowed intents...)

This means you tried to use `GatewayIntent.GUILD_MEMBERS` or `GatewayIntent.GUILD_PRESENCES` without enabling it in your application dashboard. To use these privileged intents you first have to enable them.

1. Open the [application dashboard](https://discord.com/developers/applications)
2. Select your bot application
3. Open the **Bot** tab
4. Under the **Privileged Gateway Intents** section, enable either **SERVER MEMBERS INTENT** or **PRESENCE INTENT** depending on your needs.

If you use these intents you are limited to 100 guilds on your bot. To allow the bot to join more guilds while using this intent you have to [verify your bot](https://blog.discord.com/the-future-of-bots-on-discord-4e6e050ab52e). This will be available in your application dashboard when the bot joins at least 75 guilds.
