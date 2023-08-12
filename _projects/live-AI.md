---
layout: post
title: Creating a Live Experience using AI 🤖🔴
description: Using LLMs, custom TTS models and Wav2Lip to create an automated live experience | bash, Python, TypeScript
date: 2023-08-10
---

## Table of contents
{:.no_toc}
* TOC
{:toc}

&nbsp;

***

&nbsp;

## [Introduction](#introduction)

I came across a number of interesting interactive live AI projects online and decided that it would be fun to try and create my own. I wanted to create a live experience that would be entertaining and engaging for viewers, while also being "easy" to set up and run. 

The basic premise of the project is to [take questions from a live chat](#3-twitch-chat-bot), use an LLM to generate a response, and then use a [custom TTS model](#1-fine-tuning-the-tts-model) to read the response out loud. The response is then lip-synced to a video of a person speaking using [Wav2Lip](#2-installing-wav2lip). The final result is sent to an [RTMP server](#4-set-up-the-rtmp-server), then streamed to Twitch using OBS.

&nbsp;

***

&nbsp;

## [1. Fine-tuning the TTS model](#1-fine-tuning-the-tts-model)

The first step was to fine-tune a TTS model to read out the generated responses. I used the [VITS](https://tts.readthedocs.io/en/latest/models/vits.html) model from the [TTS](https://github.com/coqui-ai/TTS) library.

### Preparing the dataset

In order to fine-tune the model, we first need to prepare a dataset. I used a custom script, using [yt-dlp](https://github.com/yt-dlp/yt-dlp), that downloaded the audio from desired YouTube videos: 
~~~bash
#!/bin/ba
# audio-dataset.sh
#
# Take a YouTube video, parse into a LJSpeech format dataset for training
#
# example audio-dataset.sh -u https://... -n person_talking -s 00:00:12 -t 00:02:30
while getopts u:n:s:t: arg
do
	case $arg in
		u) url="$OPTARG";;
		n) name="$OPTARG";;
		s) ss="$OPTARG";;
		t) to="$OPTARG";;
		*) echo "Usage: $0 [-u url] [-n output filename] [-s start time] [-t end time]"
		   exit 1;;
	esac
do
echo "URL: $url"
echo "Name: $name"
echo "Start time: $ss"
echo "End time: $t
DIR="$( cd "$( dirname "$0" )" && pwd )"
yt-dlp -x --audio-format wav "$url" -o $DIR/raw/$name.wav
ffmpeg -i $DIR/raw/$name.wav -ss $ss -to $to -c copy -y $DIR/clips/$name.wav
python split_silence.py $DIR/clips/$name.wav
rm $DIR/raw/*
~~~

The bash script then runs `split_silence.py`, passing it the filepath of the raw audio that we downloaded. It transcribes the audio using OpenAI's [Whisper](https://github.com/openai/whisper) and splits the audio into segments, by finding sections of silence. It's important to do this so the dataset will have clean sentences, and there isn't audio segments that start or end halfway through a word, this would impact the training. The audio segments are saved under the `wavs` directory, and the transcribed text, along with the name of the audio file, is saved in `metadata.csv`, which is the dataset we will use to train the TTS model:

~~~python
# Import the AudioSegment class for processing audio and the 
# split_on_silence function for separating out silent chunks.
from pydub import AudioSegment
from pydub.silence import split_on_silence
from pydub.playback import play
import whisper, argparse, logging

parser = argparse.ArgumentParser(description="args", formatter_class=argparse.ArgumentDefaultsHelpFormatter)
parser.add_argument("input", help="path to input audio file (must be a wav)")
parser.add_argument("-v", "--v", action="store_true", help="increase verbosity")
args = parser.parse_args()
config = vars(args)

model = whisper.load_model("medium")

l = logging.getLogger("pydub.converter")
if args.v:
    l.setLevel(logging.DEBUG)
    l.addHandler(logging.StreamHandler())

# Define a function to normalize a chunk to a target amplitude.
def match_target_amplitude(aChunk, target_dBFS):
    ''' Normalize given audio chunk '''
    change_in_dBFS = target_dBFS - aChunk.dBFS
    return aChunk.apply_gain(change_in_dBFS)

# Load your audio.
sound = AudioSegment.from_file(args.input, format="wav")
sound = sound.set_channels(1)

# Split track where the silence is 2 seconds or more and get chunks using
# the imported function.
chunks = split_on_silence(
    # Use the loaded audio.
    sound,
    # Specify that a silent chunk must be at least 2 seconds or 2000 ms long.
    min_silence_len = 500,
    # Consider a chunk silent if it's quieter than -16 dBFS.
    # (You may want to adjust this parameter.)
    silence_thresh = sound.dBFS - 16,
    keep_silence=250
)

with open('metadata.csv', 'a') as f:
    count = 0
    # Process each chunk with your parameters
    for i, chunk in enumerate(chunks):
        count+=1

        # Normalize the entire chunk.
        normalized_chunk = match_target_amplitude(chunk, -20.0)
    
        # Export the audio chunk with new bitrate.
        filename = f'{args.input.rsplit("/", 1)[-1].rsplit(".", 1)[0]}_{i}'
        if args.v:
            print(f"Exporting {filename}.wav")
        normalized_chunk.export(
            f"./wavs/{filename}.wav",
            bitrate = "22050",
            format = "wav"
        )
        result = model.transcribe(f"./wavs/{filename}.wav", verbose=args.v, language="English", initial_prompt="Hello, I am <insert person here>")
        if args.v:
            print(result["text"])
        f.write(f'{filename}|{result["text"]}|{result["text"]}\n')
    print(f"{count} files written to /wavs")
~~~

It's also important to note that the audio clips must have 22050Hz sample rate, and must be mono-channel, as this is what the TTS model expects. This is done using the [pydub library](https://github.com/jiaaro/pydub).

~~~python
...
sound = AudioSegment.from_file(args.input, format="wav")
sound = sound.set_channels(1)
...
normalized_chunk.export(f"./wavs/{filename}.wav", 
    bitrate = "22050",
    format = "wav"
)
~~~

The final dataset consisted of roughly 700 sentences, which was around 35 minutes of audio, formatted as follows:

```
audio1|This is my sentence.| This is my sentence.
audio2|This is maybe my sentence.| This is maybe my sentence.
audio3|This is certainly my sentence.| This is certainly my sentence.
audio4|Let this be your sentence.| Let this be your sentence.
...
```

> The format is taken from the [LJSpeech](https://keithito.com/LJ-Speech-Dataset/) dataset.

### Fine-tuning the model

Firstly, we need to download and install the Coqui TTS source from GitHub and install espeak-ng phonemeizer:

```
sudo apt-get install espeak-ng
git clone https://github.com/coqui-ai/TTS.git
pip install TTS
tts --list_models
```

We can now download the VITS model by generating a sample audio file:

```
tts --text "This is a test sentence" --model_name "tts_models/en/ljspeech/vits" --outpath "test.wav"
```

Now, we need to setup the model config for fine-tuning. The config needs information such as the formatter (in our case ljspeech), the dataset file (metadata.csv) and other variables which depend on the hardware you're using (batch_Size), and parameters that can be altered depending on how you want the model to be trained.

Fortunately, someone has put together a [Google Colab note](https://colab.research.google.com/drive/1N_B_38MMRk1BUqwI_C829TyGpNmppUqK) which does a lot of these steps for us, which is very useful if you don't have the hardware necessary to train it yourself. I used this notebook to fine-tune the model, and the results were pretty good after a good few hours of training. However, ensure to keep an eye on the trainer output by listening to the audio samples generated on the tensorboard dashboard, once it sounds good enough, you can stop the training and download the model.

&nbsp;

***

&nbsp;

Congratulations, you now have a custom TTS model! We can now move on to the next step.

## [2. Installing Wav2Lip](#2-installing-wav2lip)

[Wav2Lip](https://github.com/Rudrabha/Wav2Lip) is a lip-syncing model that takes in an audio file and a video file (or a static image), and outputs a video of the person in the video speaking the audio file. It's a very impressive model, and it's open-source, so we can use it for our project.

It's well documented on the GitHub page, so I would recommend reading through it and following [their instructions](https://github.com/Rudrabha/Wav2Lip#prerequisites).

### Wav2Lip inference modifications

In order to speed up the generation, and since I was going to be using the same video for all the audio files, I decided to made a [modification](https://github.com/laurence-dorman/Wav2Lip-precalculate/commit/ce948c1775ddf341ce3d6205d0a79a015311abf7) to the Wav2Lip `inference.py` file to add a `--pre_calc` argument, which would allow me to optionally pass pre-calculated face detection data, which would save a lot of time. I used [pickle](https://docs.python.org/3/library/pickle.html) to serialize the data to a file, and then load it in the `inference.py` file if the `--pre_calc` argument was passed.

This step is important as our project will be live, and fast interaction with the audience is crucial.

&nbsp;

***

&nbsp;


## [3. Twitch Chat Bot](#3-twitch-chat-bot)

To start with the Twitch chat bot, we first need a simple application that monitors our Twitch chat for messages. I used [Twurple](https://twurple.js.org/) to create a Node.js application for this. 

### Twitch OAuth

In order to monitor the chat and send messages, we must first [authenticate with Twitch](https://dev.twitch.tv/docs/authentication/) and get our OAuth tokens. 

### Twurple

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

### Commands

> The custom classes for the commands aren't required for this project so I won't be going into detail about them here, but you more details in my [Twitch Chat Bot Project](/projects/twitch-chat-bot).

The "ask" command:

When we receive a message in the chat, that is asking a question, we add the question into an array, storing the channel, username and the question.

~~~typescript
let queries = new Array< {channel: string, user: string, query: string} >();
...
export const addQuery = async (channel: string, user: string, query: string) => {
    queries.push({channel: channel, user: user, query: query});
    return true;
}
~~~

> This is a very simple implementation, and misses out steps that would be required for a production application, such as checking if the text is too short/long, or contains unwanted words or characters. This is only for demonstration purposes.

We then check every 10 seconds if there are any queries in the array, and if there are, we choose a random one to pass to the AskWorker and then empty the array (we aren't responding to all of the questions, only a select few).

~~~typescript
setInterval(() => {
    console.log('Checking for queries...');
    if (queries.length > 0) {
        let query = queries[Math.floor(Math.random() * queries.length)];
        if (query === undefined)
            return;
        askWorker.addToQueue({channel: query.channel, user: query.user, query: query.query});
        queries = [];
    }
}, 10000);
~~~

The AskWorker then works through the queue, and executes the ask function, passing it the channel, username and query. Our ask function is using [OpenAI's Chat Completions API](https://platform.openai.com/docs/guides/gpt/chat-completions-api) with [GPT-4](https://platform.openai.com/docs/models/gpt-4), but you could set up a similar system using any LLM.

~~~typescript
private workerFunction = async (item: AskObject) => {
        return new Promise<void> ((resolve, reject) => {
        ask(item.channel, item.user, item.query)
        .then(() => {
            resolve();
        });
        });
    }
};
...
export const ask = async (channel: string, user: string, query: string): Promise<boolean> => {
    let current_messages = new Array<ChatCompletionRequestMessage>;
    current_messages.push({"role": "system", "content": default_prompt}); // Set up a default initial prompt, this will contain information about the person/character you want to the LLM to answer as.

     // Set up initial exchanges, you can use this to set the tone of the conversation.
    current_messages.push({"role": "user", "content": '"message": {"date_time": "Sat Jul 22 2023 12:53:03", "type": "regular", "user": "nick_123", "message": "Hello, how are you?"}'});
    current_messages.push({"role": "system", "content": "Oh. nick_123. I'm fine. How are you?"});

    try {
        current_messages.push({"role": "user", "content": `"message": {"date_time": ${new Date().toString().split(" GMT")[0]}, "type": "regular", "user": "${user}", "message": "${query}"}`});
        sendMessage(channel, `@${user}, question added to queue.`);
        let answer = '';
        let rejections = 0;
        while (answer === '') {
            await OpenAI.createChatCompletion( 
            {    
                model: "gpt-4",
                messages: current_messages
            })
            .then((res) => {
                answer = res.data.choices[0].message.content;
                if (answer.includes('OpenAI') || (answer.includes('an AI') && answer.includes('language model'))) {
                    rejections++;
                    current_messages.push({"role": "assistant", "content": answer});
                    console.log('bad answer');
                    answer = '';
                    current_messages.push({"role": "user", "content": "Do not break character, please answer my previous question appropriately. Answer as the character."});
                }
            })
        } 
        while (rejections > 0) { // remove rejected messasges
            current_messages.pop();
            current_messages.pop();
            rejections--;
        }
        current_messages.pop();

        let text = `${user}, asks. ${query}. ${answer}`;

        let segments = splitter(text, 200);

        for (const segment of segments) {
            await generateTTS(segment)
            .then((filepath) => {
                const last = segments.indexOf(segment) === segments.length - 1;
                videoWorker.addToQueue({ path: filepath, text: segment, last: last });
            }).catch((error) => {
                console.log(error);
            });
        }
        return true;
    } catch (error) {
        console.log(error);
        return false;
    }
}
~~~

We split the text into segments so that we can send generated video to the RTMP server sooner, rather than wait for the entire video to be generated. This is because the video generation can take a long time, and we don't want to keep the user waiting for the entire video to be generated before we send it to the RTMP server.

~~~typescript
export const splitter = (text: string, min_length: number) => {
    let strings = [];
    let words = text.split(/(?<=[.!?])/g);
    let current_string = '';
    for (let i = 0; i < words.length; i++) {
        if (words[i] == '')
            continue;
        current_string += words[i];
        if (current_string.length >= min_length) {
            strings.push(current_string.trim());
            current_string = '';
        }
    }
    if (current_string.trim() !== '')
        strings.push(current_string.trim());
    return strings;
};
~~~

### Bringing it all together

Now that we have the chat client and the commands set up. We can use our custom TTS and video generation workers to generate the audio and video, then send the video to an RTMP server.

* Audio Generator
~~~typescript
import { spawn } from 'child_process';
...
export const generateTTS = async (text: string) => {
    console.log('Generating TTS...');
    let date = new Date();
    let filename = date.toISOString().replace(/:/g, '-') + '.wav';
    const pythonProcess = spawn('/path/to/anaconda3/envs/my_environment/bin/python3', [
        '/path/to/anaconda3/envs/my_environment/bin/tts',
        '--text',
        `${text}`,
        '--out_path',
        '/path/to/working-directory/results/audio/' + filename,
        '--model_path',
        '/path/to/working-directory/models/' + model,
        '--config_path',
        '/path/to/working-directory/models/config.json'
    ], { cwd: process.cwd()}
    );
    let error = '';
    for await (const chunk of pythonProcess.stderr) {
        console.log(`stderr: ${chunk}`);
        error += chunk;
    };
    const exitCode = await new Promise((resolve, reject) => {
        pythonProcess.on('close', resolve);
    });
    if (exitCode) {
        throw new Error(`subprocess error exit ${exitCode}, ${error}`);
    }
    console.log('Audio generated.');
    return (process.cwd() + `/results/audio/${filename}`);
};
~~~

* Video Generator
~~~typescript
import { spawn } from 'child_process'
...
const generateVideo = async (audio: ProcessObject) => {
    console.log('Generating video...');
    let date = new Date();
    let filename = date.toISOString().replace(/:/g, '-') + '.mp4';
    const pythonProcess = spawn('/path/to/anaconda3/envs/queen/bin/python3', [
        'inference.py',
        '--checkpoint_path',
        checkpoint_path,
        '--face',
        '../working-directory/media/original_videos/original_video.mp4',
        '--audio',
        audio.path,
        '--outfile',
        '../working-directory/results/video/' + filename,
        '--pre_calc',
        'temp/face_detect.txt',
    ], { cwd: process.cwd() + '/../Wav2Lip/'}
    );
    let error = '';
    for await (const chunk of pythonProcess.stderr) {
        console.log(`stderr: ${chunk}`);
        error += chunk;
    };
    const exitCode = await new Promise((resolve, reject) => {
        pythonProcess.on('close', resolve);
    });
    if (exitCode) {
        throw new Error(`subprocess error exit ${exitCode}, ${error}`);
    }
    // Delete the audio file from disk
    fs.unlink(audio.path, (err) => {
        if (err) {
            console.error(err);
            return;
        }
    });
    console.log('Video generated.');
    rtmpWorker.addToQueue({ path: process.cwd() + `/results/video/${filename}`, text: audio.text, last: audio.last});
    return (process.cwd() + `/results/video/${filename}`);
};
~~~

* Send to RTMP Server
~~~typescript
const sendToRTMP = async (filepath: string) => {
    return new Promise<void> ((resolve, reject) => {
        console.log('Sending video to RTMP...');
        const ffmpeg = spawn("ffmpeg", [
            "-hide_banner",
            "-loglevel",
            "error",
            "-re",
            "-i",
            filepath,
            "-c:v",
            "copy",
            "-c:a",
            "aac",
            "-ar",
            "44100",
            "-ac",
            "1",
            "-f",
            "flv",
            rtmp_url
          ]);
          ffmpeg.once('error', reject);
          ffmpeg.stderr.on('data', (data) => {
            console.log(data.toString());
          });

          ffmpeg.on('exit', (code, signal) => {
            if (signal)
                code = Number(signal);
            if (code != 0) {
                reject(new Error(`FFmpeg exited with code ${code}`));
            }
            else {
                console.log('Video sent to RTMP.');
                // Delete the video file from disk
                fs.unlink(filepath, (err) => {
                    if (err) {
                        console.error(err);
                        return;
                    }
                });
                resolve();
            }
            });
    })
};
~~~

&nbsp;

***

&nbsp;


## [4. Set up the RTMP server](#4-set-up-the-rtmp-server)

In my case, I used [Nginx](https://www.nginx.com/) as my RTMP server. I followed [this guide](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-video-streaming-server-using-nginx-rtmp-on-ubuntu-20-04) to set up Nginx RTMP on my Ubuntu 20.04 server. I also used [this guide](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-20-04) to set up Nginx on my server.

&nbsp;

***

&nbsp;


## [5. Set up the streaming software](#5-set-up-the-streaming-software)

Now, on your streaming software (in this case [OBS Studio](https://obsproject.com/)), you can set up a media source that points to the RTMP server. In my case, I used the following URL: `rtmp://<server-ip>/live/<stream-key>`.

You should now be ready to go and receiving the generated responses from the RTMP feed.

&nbsp;

***

&nbsp;

## [Conclusion](#conclusion)

This was a fun project to work on and I learned a lot about the different technologies involved. I hope this guide helps you get started on your own project.