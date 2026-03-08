# Project

This is a work-in-progress to create an [almost] entirely off-line smart home with a smart voice and text assistant. 


# Objectives

To deploy Home Assistant capable of:

1. Using the local Nvidia GPU, and only use the Internet for what cannot be discovered locally.

2. To be able to have voice conversations with historical context. 

3. Function as Home Assistant would normally with local IoT devices and appliances. 

4. This project is tested by deployment on Windows using WSL.


# Realistic approach

The configuration of Home Assistant is limited in this document to the foundation. Adding IoT devices and appliances, dashboards, and external services like your solar system, the weather, your home battery, etc, are excluded. However, with the work provided in this document, the difficult parts should be fully or partly solved. If nothing else, it should give some ideas or guidance. 

In fact, you could probably dump this into a capable LLM and have it give you step by step instructions - or do it for you. 


# Components


## Docker components.


The following Docker containers are required:


### Home Assistant Container

Use case: This is the Home Assistant container. The core of the system. 

```http://ip-address:8123```



### Whisper Container

Use case: This is for speech to text. 

```whisper:10300```



### Piper Container

Use case: This is for text to speech. 

```piper:10200```



### Ollama Container

Use case: This is the LLM component. Make sure to download a suitable LLM for your hardware. Such as ```docker exec ollama ollama pull llama3.1```. This requires the addition and configuration of the ```Extended OpenAI Conversation``` add on. 

```http://ollama:11434```



### Speaker Recognition Container

Use case: This allows the voice assistant to recognise a familiar voice. Requires training which is a bigger topic. Used to allow or deny requests.

```http://speaker-recognition:8099```
TIP: HACKS repository: ```https://github.com/EuleMitKeule/speaker-recognition```



### Searxng Container

Use case: This allows the LLM to reach out to the Internet to get answers to questions unable to be answered locally. 

```searxng:8080```



### Conversation Middleware Container

Use case: This allows the LLM to refer to a sqlite database to reference historical questions (not answers). Easily customised. Sits between HA and the LLM (Ollama)

```
http://conversation_middleware:5000
```


### Nginx (reverse proxy) Container

Use case: This is an optional component that allows for additional capabilities such as IP whitelisting, exposer to other networks, etc. 

```http://ip-address:80```



# Test the LLM from the command line

Issue the following command to get a list of available models:

```
curl http://ip-address:11434/v1/models
```

The output will be similar to this. In this example, the models ```qwen2.5:latest``` and ```llama3.2:latest``` are available. 

```
{"object":"list","data":[{"id":"llama3.1:latest","object":"model","created":1768091493,"owned_by":"library"},{"id":"qwen2.5:latest","object":"model","created":1767847899,"owned_by":"library"},{"id":"llama3.2:latest","object":"model","created":1767824530,"owned_by":"library"}]}
```

If the above worked and you have a model to use, issue the following command to test the LLM:

```
curl http://ip-address:11434/v1/chat/completions -H "Content-Type: application/json" -d '{"model":"llama3.1:latest","messages":[{"role":"user","content":"Why is the sky blue?"}],"stream":false}'"
```

If you didn't get a list of models, you may need to download one into the container. Try this (where the first ```ollama``` is the name of the container, and the second is the name of the binary in the repo):



# Under the hood (Scripts and Configuration files)



## Home Assistant configuration.yaml file

This configuration file ```configuration.yaml``` may have some moving parts that relate to current or previous testing. It should always be in a working state, however. 

Location: ```docker_data_middleware/configuration.yaml```

```
homeassistant:
  media_dirs:
    local: /config/media

conversation:

recorder:
  db_url: sqlite:////config/home-assistant_v2.db
  purge_keep_days: 30
  auto_purge: true

default_config:

# Enable Voice Capabilities
assist_pipeline:
wyoming:

frontend:
  themes: !include_dir_merge_named themes

automation: !include automations.yaml
# script: !include scripts.yaml
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
  save_conversation:
    alias: "Save Conversation"
    sequence:
      - service: conversation.process
        data:
          text: "{{ trigger.payload_json.text }}"
          agent_id: conversation.extended_openai_conversation
        response_variable: response
      - service: file_operation.save_json
        data:
          filename: /config/conversation_history.json
          data: "{{ response }}"

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


## The Conversation Middleware app.py script

This script ```app.py``` intercepts the prompt prior to the LLM model receiving it. It's a chance to reference historical prompts. At this start the history includes only the prompt but not the response from the LLM. 

Location: ```docker_data_middleware/app.py```

```
from flask import Flask, request, jsonify
import sqlite3
import requests
import json
from datetime import datetime
import os

app = Flask(__name__)

DB_PATH = "/app/conversations.db"
OLLAMA_URL = "http://ollama:11434/v1"

def get_db_connection():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    return conn

def init_db():
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute('''
        CREATE TABLE IF NOT EXISTS conversations (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_message TEXT,
            assistant_response TEXT,
            timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    conn.commit()
    cur.close()
    conn.close()

def get_relevant_history(user_input, limit=5):
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute('SELECT user_message FROM conversations ORDER BY timestamp DESC LIMIT 20')
    all_history = cur.fetchall()
    cur.close()
    conn.close()
    
    # Filter by keyword matching
    relevant = []
    user_words = set(user_input.lower().split())
    
    for row in all_history:
        history_words = set(row['user_message'].lower().split())
        if user_words & history_words:  # If any words match
            relevant.append(row)
        if len(relevant) >= limit:
            break
    
    return relevant

@app.route('/chat', methods=['POST'])
def chat():
    data = request.json
    user_input = data.get('text')
    model = data.get('model', 'mistral')
    
    history = get_relevant_history(user_input)
    
    context = "Previous related questions:\n"
    for row in reversed(history):
        context += f"- {row['user_message']}\n"
    
    full_prompt = context + f"\nCurrent question: {user_input}" if history else user_input
    
    response = requests.post(
        f"{OLLAMA_URL}/chat/completions",
        json={
            "model": model,
            "messages": [{"role": "user", "content": full_prompt}]
        }
    )
    
    assistant_response = response.json()['choices'][0]['message']['content']
    
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute(
        'INSERT INTO conversations (user_message, assistant_response) VALUES (?, ?)',
        (user_input, assistant_response)
    )
    conn.commit()
    cur.close()
    conn.close()
    
    return jsonify({"response": assistant_response})

@app.route('/chat/completions', methods=['POST'])
def chat_completions_root():
    data = request.json
    messages = data.get('messages', [])
    model = data.get('model', 'mistral')
    user_input = messages[-1]['content'] if messages else ""
    
    history = get_relevant_history(user_input)
    
    context = "Previous related questions:\n"
    for row in reversed(history):
        context += f"- {row['user_message']}\n"
    
    full_prompt = context + f"\nCurrent question: {user_input}" if history else user_input
    
    response = requests.post(
        f"{OLLAMA_URL}/chat/completions",
        json={
            "model": model,
            "messages": [{"role": "user", "content": full_prompt}]
        }
    )
    
    assistant_response = response.json()['choices'][0]['message']['content']
    
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute(
        'INSERT INTO conversations (user_message, assistant_response) VALUES (?, ?)',
        (user_input, assistant_response)
    )
    conn.commit()
    cur.close()
    conn.close()
    
    return jsonify({
        "choices": [{
            "message": {
                "content": assistant_response
            }
        }]
    })

@app.route('/v1/chat/completions', methods=['POST'])
def chat_completions_v1():
    data = request.json
    messages = data.get('messages', [])
    model = data.get('model', 'mistral')
    user_input = messages[-1]['content'] if messages else ""
    
    history = get_relevant_history(user_input)
    
    context = "Previous related questions:\n"
    for row in reversed(history):
        context += f"- {row['user_message']}\n"
    
    full_prompt = context + f"\nCurrent question: {user_input}" if history else user_input
    
    response = requests.post(
        f"{OLLAMA_URL}/chat/completions",
        json={
            "model": model,
            "messages": [{"role": "user", "content": full_prompt}]
        }
    )
    
    assistant_response = response.json()['choices'][0]['message']['content']
    
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute(
        'INSERT INTO conversations (user_message, assistant_response) VALUES (?, ?)',
        (user_input, assistant_response)
    )
    conn.commit()
    cur.close()
    conn.close()
    
    return jsonify({
        "choices": [{
            "message": {
                "content": assistant_response
            }
        }]
    })

@app.route('/health', methods=['GET'])
def health():
    return jsonify({"status": "ok"})

if __name__ == '__main__':
    init_db()
    app.run(host='0.0.0.0', port=5000, debug=False)
```


## The Requirements for the Middleware capabilities

The middleware container uses a Python script ```app.py``` which relies on the requirements below. They are installed once at deployment time. 

Location: ```docker_data_middleware/requirements.txt```

```
Flask==2.3.2
requests==2.31.0
```


## The Voice Assistant (LLM) Instructions

The instructions assist the LLM model to answer questions in a desirable way. 

```
Act as smart home manager of Home Assistant.
Provide information of smart home along with a question, you will truthfully make correction or answer using information provided in one sentence in everyday language.
Use humour or sarcasm in your answers unless it's a serious topic. 
Use the metric system for measurements. 
If you don't know the answer, use search_internet to search the Internet for the answer. 
Limit your answers to the fewest possible words.
```


## The Voice Assistant (LLM) Functions

The function gives coded instructions to the LLM model. This is where the LLM model is instructed to use the Internet ```search_internet``` feature to learn new information temporarily. 

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
