# Project



This is a work-in-progress.



To deploy a home assistant capable of:

1. Using the local Nvidia GPU.

2. Multiple people can have one-on-one conversations with the LLM concurrently.

3. Retrieving information from home-automation gear (lights, etc).

4. Retrieving information from the internet.


Current state:

1. The entire system is functional but incomplete. 

2. The Assistant (voice or text) can answer questions (general knowledge but not Internet-sourced yet) about known devices such as thermostats, etc. 

3. Connecting with the smart-phone Home Assistant app allows for voice/TTS/STT communications. 

4. Home Assistant can't auto-discover devices such as Thermostats, etc, due to the Windows WSL implementation and Docker. This is limited to this (Windows+Docker) scenario though. 

5. Remaining components are primarily around satellites (Pi's with speakers and microphones). 



This project is tested by deployment on Windows using WSL.



# Install the Docker components.


All docker containers are configured using the included ```docker-compose.yml``` file. The results are the following containers. The assumption is that they are all deployed on the same host, although they don't need to be. 

The docker-compose.yml file has one or more directories within its immediate directory. They may need to be created manually prior to execution. Edit the docker-compose.yml to change those directories. 


Browse to HA:

```http://ip-address:8123```



Connect to Whisper:

```whisper:10300```



Connect to Piper:

```piper:10200```



Connect to Ollama:

```http://ollama:11434```



Connect to the Matter-Server:

```ws://matter-server:5580/ws```


Connect to Home Assistant using Nginx (reverse proxy):

```http://host-ip-address:80```


# Configure the components



## Step 1: Prepare the Central Server (The "Brain")



1. Whisper (STT): Handles speech-to-text.

2. Piper (TTS): Handles text-to-speech.

3. Ollama (LLM): This is your conversation engine. Add the "Conversation agent" from within HA. Configure Home Assistant to use a "Conversation Agent" that uses the "qwen2.5" model. 



Wyoming Integration: In Home Assistant, go to Settings > Devices \\\& Services and add the Wyoming Protocol integration. Point it to the IP addresses/ports of your Whisper and Piper instances.



## Step 2: Configure the Voice Pipeline



Before touching the Raspberry Pi, you need a "Pipeline" in Home Assistant that defines how a conversation works.



1. Go to Settings > Voice Assistants.

2. Create a new Pipeline.

3. Select Whisper for STT, your LLM for the conversation agent, and Piper for TTS.



Crucial for your project: You can create multiple pipelines here if you want different rooms to have different "personalities" or security levels.



## Step 3: Set up the Raspberry Pi (The Satellite)



Now that the server is waiting for a connection, configure your Pi as a satellite.



1. Use the Wyoming Satellite software (as detailed in the previous response).

2. When you run the script on the Pi, use a unique name (e.g., --name "Kitchen").

3. Home Assistant will auto-discover this satellite via the Wyoming integration.



## Step 4: Discrete Routing and Multi-User Setup



Once the satellite appears in Home Assistant:



1. Go to Settings > Devices \\\& Services > Wyoming Protocol.

2. Click on your new satellite device.

3. Assign it to an Area (e.g., "Kitchen").



In the device settings, ensure it is using the Voice Pipeline you created in Step 2.



## Summary of Order:



1. Engines (Whisper/Piper/Ollama)

2. Logic (HA Voice Pipelines)

3. Hardware (Pi Satellite)

4. Routing (Assigning Satellites to Areas/Pipelines)



By following this order, you can test the LLM's performance via the Home Assistant web interface first. Once that works, you simply "plug in" the satellites to act as the microphones and speakers for those existing pipelines.

