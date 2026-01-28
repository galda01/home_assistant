# Project



This is a work-in-progress.



To deploy a home assistant capable of:

1. Using the local Nvidia GPU.

2. Multiple people can have one-on-one conversations with the LLM concurrently.

3. Retrieving information from home-automation gear (lights, etc).

4. Retrieving information from the internet.


Current state:

1. The entire system is functional but incomplete. 

2. The Assistant (voice or text) can answer questions about HA concerns, and with the healp of SearXNG it can search the Internet to assist.

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



Connect to the Searxng:

```searxng:8080```



Connect to Home Assistant using Nginx (reverse proxy):

```http://host-ip-address:80```


# Test the LLM from the command line

Issue the following command to get a list of available models:

```
curl http://127.0.0.1:11434/v1/models
```

The output will be similar to this. In this example, the models ```qwen2.5:latest``` and ```llama3.2:latest``` are available. 

```
{"object":"list","data":[{"id":"llama3.1:latest","object":"model","created":1768091493,"owned_by":"library"},{"id":"qwen2.5:latest","object":"model","created":1767847899,"owned_by":"library"},{"id":"llama3.2:latest","object":"model","created":1767824530,"owned_by":"library"}]}
```

If the above worked and you have a model to use, issue the following command to test the LLM:

```
curl http://10.0.0.43:11434/v1/chat/completions -H "Content-Type: application/json" -d '{"model":"llama3.1:latest","messages":[{"role":"user","content":"Why is the sky blue?"}],"stream":false}'"
```

If you didn't get a list of models, you may need to download one into the container. Try this (where the first ```ollama``` is the name of the container, and the second is the name of the binary in the repo):

```
docker exec -it ollama ollama pull llama3.1
```


# Configure the components



## Step 1: Prepare the Central Server (The "Brain")



1. Whisper (STT): Handles speech-to-text.

2. Piper (TTS): Handles text-to-speech.

3. For the LLM (```llama3.1:latest``` in our case), make these settings:
- Settings, Extended OpenAI Conversation, Add Service
- Name: Local LLM
- API Key: none
- Base URL: https://ollama:11434/v1
- API Version: <blank>
- Organisation: <blank>
- Skip Auth: TRUE
- API Provider: OpenAI
- Save
- Reconfigure Conversation Agent
- Chat_Model: llama3.1:latest

Add the following to the ```prompt template``` but customise it accordingly:
```
I want you to act as smart home manager of Home Assistant.
I will provide information of smart home along with a question, you will truthfully make correction or answer using information provided in one sentence in everyday language.
Use humour or sarcasm in your answers. 
Use the metric system for measurements. 
If you don't know the answer, use search_internet to search the Internet for the answer. 
The current state of devices is provided in available devices.
Use execute_services function only for requested action, not for current states.
Do not execute service without user's confirmation.
Do not restate or appreciate what user says, rather make a quick inquiry.
```

Set the ```function``` to be the following:

```
- spec:
    name: search_internet
    description: "Search for current events, news"
    parameters:
      type: object
      properties:
        query:
          type: string
      required:
        - query
  function:
    type: script
    sequence:
      - action: script.searxng_search_script
        data:
          query: "{{query}}"
        response_variable: _function_result
```

4. Edit the ```configuration.yml``` file and restart the ```homeassistant``` container. 

These settings are a guide:

```
default_config:

frontend:
  themes: !include_dir_merge_named themes

automation: !include automations.yaml
script: !include scripts.yaml
scene: !include scenes.yaml

http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 172.16.0.0/12  # Standard Docker network range
    - 127.0.0.1      # Localhost

template: !include templates.yaml

rest_command:
  searxng_search:
    url: "http://searxng:8080/search?q={{ query | urlencode }}&format=json&apikey=mysecret"
    method: GET
    timeout: 30

script:
  searxng_search_script:
    alias: "SearXNG Search"
    fields:
      query:
        description: "The search query"
    sequence:
      - action: rest_command.searxng_search
        data:
          query: "{{ query }}"
        response_variable: searx_output
      - variables:
          final_response:
            results: >-
              {% if searx_output.content.results is defined and searx_output.content.results|length > -1 %}
                {{ searx_output.content.results[0].title }}: {{ searx_output.content.results[0].content | truncate(1500) }}
              {% else %}
                No results found for "{{ query }}".
              {% endif %}
      - stop: "Success"
        response_variable: final_response

```

5. Wyoming Integration: In Home Assistant, go to Settings > Devices \\\& Services and add the Wyoming Protocol integration. Point it to the IP addresses/ports of your Whisper and Piper instances.



## Step 2: Configure the Voice Pipeline



Before touching the Raspberry Pi, you need a ```pipeline``` in Home Assistant that defines how a conversation works.



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

