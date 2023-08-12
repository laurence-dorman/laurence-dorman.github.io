---
layout: post
title: Chess Game Tweet Bot 🤖♟
description: Twitter bot written in Python that posts GIFs of new games | Python
date: 2023-08-09
---

## Table of contents
{:.no_toc}
* TOC
{:toc}

&nbsp;

***

&nbsp;

## [Introduction](#introduction)

This script automates the process of fetching the latest chess games from the Chess.com API and tweeting about them with a gif representation of the game.

> Game over: @User won by checkmate, new bullet rating: XXXX.

> ![Example GIF](/assets/images/game.gif "An example of a GIF generated by the application")

&nbsp;

***

&nbsp;

## [1. Main libraries used](#1-main-libraries-used)

Import the necessary libraries for parsing PGN files, working with Twitter API, fetching data,handling environment variables, and getting the current time.

~~~python
import pgn2gif
import tweepy
import urllib.request, urllib.error, json
from datetime import datetime
from dotenv import dotenv_values
~~~

&nbsp;

***

&nbsp;

## [2. Environment variables and Twitter API setup](#2-environment-variables-and-twitter-api-setup)

Environment variables, stored in a .env file, are loaded using the dotenv library. These variables are then used to authenticate with the Twitter API using Tweepy.

~~~python
config = dotenv_values(".env")
...
api = tweepy.API(auth)
~~~

&nbsp;

***

&nbsp;

## [3. Loading users to track](#3-loading-users-to-track)

User data is stored in a JSON file **users.json**. This data contains a list of user profiles to monitor on Chess.com, along with the Twitter username to tag in the tweet.

~~~json
{
    "users": [
        {
            "chess_username": "chess_username",
            "twitter_username": "@twitter_username"
        }
    ]
}
~~~

 Extract the user data from the JSON file and store it in a list.

~~~python
with open ('users.json', 'r') as file:
    users = json.load(file)['users']
~~~

&nbsp;

***

&nbsp;

## [4. Getting the game data](#4-getting-the-game-data)

Every X seconds *(10 by default)*, we fetch the latest games from the Chess.com API for the tracked users. The API returns a list of games for the month, we check the newest game's end time to see if it is newer than the last game we fetched (or the time the application was started). If it is, we will parse the PGN file for the result and generate a GIF of the game.

~~~python
def get_recent_game(username):
    # get month and year
    now = datetime.now()
    month = now.strftime("%m")
    year = now.strftime("%Y")
    global newest_game_end_time
    try:
        with urllib.request.urlopen(f"https://api.chess.com/pub/player/{username}/games/{year}/{month}") as url:
            data = json.load(url)
            newest_game = data["games"][-1]
            if newest_game["end_time"] > newest_game_end_time:
                newest_game_end_time = newest_game["end_time"]
                return newest_game
            else:
                return False
    except Exception as err:
        log(f'Exception error: {err}')
~~~

An example of the PGN data returned by the Chess.com API.

~~~pgn
[Event "Live Chess"]
[Site "Chess.com"]
[Date "2023.08.08"]
[Round "-"]
[White "Player1"]
[Black "Player2"]
[Result "1-0"]
[CurrentPosition "rn1qkbnr/pp1bpQ2/2B4p/6N1/4p3/8/PPP2PPP/RNB1K2R b KQ -"]
[Timezone "UTC"]
[ECO "A06"]
[ECOUrl "https://www.chess.com/openings/Reti-Opening-Tennison-Gambit"]
[UTCDate "2023.08.08"]
[UTCTime "12:55:30"]
[WhiteElo "1039"]
[BlackElo "968"]
[TimeControl "60"]
[Termination "Player1 won by checkmate"]
[StartTime "12:55:30"]
[EndDate "2023.08.08"]
[EndTime "12:56:14"]
[Link "https://www.chess.com/game/live/12345678910"]

1. e4 {[%clk 0:01:00]} 1... d5 {[%clk 0:01:00]} 2. Nf3 {[%clk 0:00:59.9]} 2... dxe4 {[%clk 0:00:59.2]} 3. Ng5 {[%clk 0:00:59.7]} 3... f5 {[%clk 0:00:58.5]} 4. d3 {[%clk 0:00:59.3]} 4... h6 {[%clk 0:00:57.8]} 5. Qh5+ {[%clk 0:00:53.6]} 5... g6 {[%clk 0:00:55.9]} 6. Qxg6+ {[%clk 0:00:51.6]} 6... Kd7 {[%clk 0:00:55.2]} 7. dxe4 {[%clk 0:00:43]} 7... fxe4 {[%clk 0:00:52.3]} 8. Bb5+ {[%clk 0:00:40.7]} 8... c6 {[%clk 0:00:51.1]} 9. Qe6+ {[%clk 0:00:40.3]} 9... Ke8 {[%clk 0:00:49.7]} 10. Bxc6+ {[%clk 0:00:37.8]} 10... Bd7 {[%clk 0:00:46.7]} 11. Qf7# {[%clk 0:00:35.2]} 1-0
~~~

Parse the PGN file, extracting key information, and generate a GIF of the game.

~~~python
def get_result(user, termination):
    way_of = termination.split("won ")[1]
    if termination.lower().find(user["chess_username"].lower()) != -1:
        return (user["twitter_username"] + " won " + way_of)
    else:
        return (user["twitter_username"] + " lost " + way_of)

def new_game(game, user):
    reverse = False
    pgn = game["pgn"]
    with open ("game.pgn", "w") as f:
        f.write(pgn)
    sides = pgn[pgn.find("[White"):pgn.find("\n[Result")].split("\n")
    termination = pgn[pgn.find("[Termination \"")+len("[Termination \""):pgn.find("\"]\n[StartTime")]
    result = get_result(user, termination)
    time_control = game["time_class"]
    if sides[0].lower().find(user["chess_username"].lower()) == -1:
        reverse = True;
        new_rating = game["black"]["rating"]
    else:
        new_rating = game["white"]["rating"]
    media_id = make_gif(reverse)
    result = f"{result}, new {time_control} rating: {new_rating}."
    return result, media_id
~~~

&nbsp;

***

&nbsp;

## [5. The main loop](#5-the-main-loop)

The main logic of the script is in the check_for_new_game() function, which iterates over each user, checks for new games, and tweets if a new game is found. The main() function runs this logic in an infinite loop, periodically checking for new games.

~~~python
def check_for_new_game():
    for user in users:
        game = get_recent_game(user["chess_username"])
        if game:
            log("New game found.")
            result, media_id = new_game(game, user)
            make_tweet(result, media_id)
            clean_up()
        else:
            log("No new game found.")
    time.sleep(refresh_rate)


def main():
    while True:
        check_for_new_game()
~~~

&nbsp;

***

&nbsp;

## [6. Deploying to a free EC2 instance](#6-deploying-to-a-free-ec2-instance)

The script is simple, and can be deployed to a free EC2 instance on AWS. The script can be run in the background as a daemon process.

### Systemd service

This is the `.service` daemon file that runs the python script as a systemd process. The bot is designed to run as a `simple` service that restarts automatically if it crashes. 
> Note: If you are using a virtual environment, make sure to use the full path to the python executable in the `ExecStart` command as shown below.
~~~
[Unit]
Description=Twitter chess.com bot
After=multi-user.target
StartLimitBurst=0
StartLimitIntervalSec=0
[Service]
Type=simple
Restart=always
WorkingDirectory=/path/to/working/directory
ExecStart=/path/to/working/directory/my_venv/bin/python3 /path/to/working/directory/script.py
[Install]
WantedBy=multi-user.target
~~~

With the service file now placed in the `/etc/systemd/system` directory, we can start the service using the following commands:

```
sudo systemctl daemon-reload
sudo systemctl start chessbot.service
```

To check the status of the service, use the following command:
```
sudo systemctl status chessbot.service
```
> Active: **active (running)** ...

&nbsp;

***

&nbsp;

# [Conclusion](#conclusion)

This project was a fun way to learn about the Chess.com and Twitter APIs, integrating them into a simple Python script. Such a bot can be useful for chess enthusiasts who want to keep an eye on their favourite players or for community managers to engage their audience with dynamic, up to date content.

> The full source code for this project can be found on [GitHub](https://github.com/laurence-dorman/chess-twitter-bot).