---
layout: post
title: Twitch Chat Bot 🤖🟣
description: Twitch chat bot | TypeScript
date: 2023-08-13
---

## Table of contents
{:.no_toc}
* TOC
{:toc}

&nbsp;

***

&nbsp;

## [Introduction](#introduction)

I wanted to develop a Twitch chat bot with unlimited possibility when it comes to functionality, and also to put into practice the JavaScript and TypeScript skills I have been learning.

&nbsp;

***

&nbsp;

## [1. Twitch OAuth](#1-twitch-oauth)

In order to monitor the chat and send messages, we must first [authenticate with Twitch](https://dev.twitch.tv/docs/authentication/) and get our OAuth tokens. 

### Twurple

I used [Twurple](https://twurple.js.org/) as it is a JS library for interacting with the Twitch API, and it has a lot of functionality that we can use in our bot. 

Set up the AuthProvider:

~~~typescript
const tokenData = JSON.parse(await fs.readFile('./src/api/twitch/tokens.json', 'utf-8'));
        _authProvider = new RefreshingAuthProvider(
            {
                clientId,
                clientSecret,
                onRefresh: async newTokenData => await fs.writeFile('./src/api/twitch/tokens.json', JSON.stringify(newTokenData, null, 4), 'utf-8'),
            },
            tokenData
        );
        return _authProvider;
~~~

> The tokens from the OAuth stage are stored in a JSON file, and are read from and written to when the tokens are refreshed.

Set up the ChatClient:

~~~typescript
await fs.readFile('./src/api/twitch/channels.txt', 'utf8').then(data => {
            channels = data.split("\n")
        }).catch(err => console.log(err));
        let authProvider = await getAuthProvider();
        let chatClient = new ChatClient({ authProvider, channels: channels,
            logger: {
                minLevel: 'error'
            }
        });
        await chatClient.connect();
        _chatClient = chatClient;
~~~

> The channels.txt file contains a list of channels that the bot will join.

Now that we have the ChatClient, we can listen for messages:

~~~typescript
import { commands } from './api/twitch/commands.js'
import { ParseCommand } from './api/twitch/command.js';
import { getChatClient } from './api/twitch/twitch.js';
...

let chatClient = await getChatClient();

chatClient.onMessage((channel, user, text) => {
        const command = ParseCommand(text);
        if (!commands.getCommands().has(command.command)) return;
        console.log(`\nchat message:\n{ channel: '${channel}', user: '${user}', text: '${text}' }`);
        console.log(`Running command: '${command.command}' with the query: '${command.query}'.\n`);
        commands.runCommand(command.command, channel, user, command.query);
    });
~~~

> This will parse every chat message and run the command if it exists.

&nbsp;

***

&nbsp;

## [2. Commands](#2-commands)

The `Command` class is used to create a command that can be run by the bot. It has a name, optional aliases, callback function, cooldown, and minimum user level required to run the command.

~~~typescript
export class Command {
    name: string;
    aliases?: string[];
    callback: Function;
    cooldown: number;
    lastUsed: Date;
    userLevel: UserLevel;
    constructor(name: string, callback: Function, userLevel: UserLevel, aliases?: string[], cooldown: number = 1000) {
        this.name = name;
        this.aliases = aliases;
        this.callback = callback;
        this.cooldown = cooldown;
        this.lastUsed = new Date(0);
        this.userLevel = userLevel;
    }
    isOnCooldown(): boolean {
        let now = new Date();
        let diff = now.getTime() - this.lastUsed.getTime();
        if (diff > this.cooldown) {
            return false;
        } else {
            return true;
        }
    }
    use(): void {
        this.lastUsed = new Date();
    }
}
~~~

The `Commands` class is used to store all of the commands, and has functions to add, get, and run commands.

~~~typescript
export class Commands {
    commands: Map<string, Command>;
    constructor(commands?: Command[]) {
        this.commands = new Map<string, Command>();
        if (commands !== null) {
            commands?.forEach( (element, index, array) => {
                this.commands.set(element.name, element);
                if (element.aliases) {
                    element.aliases.forEach(alias => {
                        this.commands.set(alias, element);
                    });
                }
            });
        }
    }
    
    addCommand(command: Command): void {
        this.commands.set(command.name, command);
        if (command.aliases) {
            command.aliases.forEach(alias => {
                this.commands.set(alias, command);
            });
        }
    }
    getCommand(name: string): Command {
        return this.commands.get(name);
    }
    getCommands(): Map<string, Command> {
        return this.commands;
    }
    async runCommand(name: string, channel: string, user: string, query: string): Promise<boolean> {
        let command = this.getCommand(name);
        if (command) {
            if (command.isOnCooldown()) {
                console.log(`Command: '${name}' is on cooldown.`)
                return false;
            } else {
                let userLevel = await getUserLevel(channel, user);
                if (userLevel >= command.userLevel) {
                    command.use();
                    await command.callback(channel, user, query);
                    return true;
                } else {
                    console.log(`User: '${user}' (${UserLevel[userLevel]}) does not have the required user level (${UserLevel[command.userLevel]}) to run command: '${name}'.`)
                    return false;
                }
            }
        } else {
            console.log(`Command: '${name}' does not exist.`);
            return false;
        }
    }
}
~~~

Here's an example on how to create a command that echoes the query back to the chat using our command classes:

~~~typescript
commands.addCommand(new Command('!echo', async (channel: string, user: string, query: string) => {
    sendMessage(channel, query);
}, 
UserLevel.Scrub));
~~~

### User Levels

The `UserLevel` enum is used to determine what user level is required to run a command.

~~~typescript
export enum UserLevel {
    Scrub = 0,
    Subscriber,
    VIP,
    Moderator,
    Owner
}
~~~

We use our `getUserLevel` function to get the user level of a user in a channel, which uses Twurple's [API client](https://twurple.js.org/reference/api/classes/ApiClient.html) for the Twitch Helix API and other endpoints.

~~~typescript
export async function getUserLevel(channel: UserNameResolvable, user: UserNameResolvable): Promise<UserLevel> {
    if (channel.toString()[0] === '#') { channel = channel.toString().slice(1); } // channel name is scuffed when it comes from IRC
    try {
        let apiClient = await getApiClient();
        let channelId = (await apiClient.users.getUserByName(channel)).id;
        let userId = (await apiClient.users.getUserByName(user)).id;
    
        if (channelId === userId) {
            return UserLevel.Owner;
        }
        else if (await apiClient.moderation.checkUserMod(channelId, userId)) {
            return UserLevel.Moderator;
        }
        else if (await apiClient.channels.checkVipForUser(channelId, userId)) {
            return UserLevel.VIP;
        }
        else if (await apiClient.subscriptions.getSubscriptionForUser(channelId, userId)) {
            return UserLevel.Subscriber
        }
        else {
            return UserLevel.Scrub;
        }
    } catch (error) {
        console.log(error);
        return UserLevel.Scrub;
    }
}
~~~

Since we are using an enum, we can use the `UserLevel` as a number, and compare them using the greater than or equal to operator to see if a user has the required user level to run a command.

~~~typescript
let userLevel = await getUserLevel(channel, user);
if (userLevel >= command.userLevel) {
    command.use();
    await command.callback(channel, user, query);
    return true;
} else {
    console.log(`User: '${user}' (${UserLevel[userLevel]}) does not have the required user level (${UserLevel[command.userLevel]}) to run command: '${name}'.`)
    return false;
}
~~~

&nbsp;

***

&nbsp;