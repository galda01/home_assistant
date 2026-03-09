# Project

This is a work-in-progress to create an [almost] entirely off-line smart home with a smart voice and text assistant.



# Objectives

To deploy Home Assistant capable of:

1. Using the local Nvidia GPU, and only use the Internet for what cannot be discovered locally.
2. To be able to have voice conversations with historical context.
3. Function as Home Assistant would normally with local IoT devices and appliances.
4. This project is tested by deployment on Windows using WSL.

We're using the "dolphin-mixtral" model throughout this project. However, use the most suitable model for your resources and objectives.

# Realistic approach

The configuration of Home Assistant is limited in this document to the foundation. Adding IoT devices and appliances, dashboards, and external services like your solar system, the weather, your home battery, etc, are excluded. However, with the work provided in this document, the difficult parts should be fully or partly solved. If nothing else, it should give some ideas or guidance.

In fact, you could probably dump this into a capable LLM and have it give you step by step instructions - or do it for you.



# Components



## Docker components



The following Docker containers are required:



### Home Assistant Container

Use case: This is the Home Assistant container. The core of the system.

Location: ```http://ip-address:8123```



### Whisper Container

Use case: This is for speech to text.

Location: ```whisper:10300```



### Piper Container

Use case: This is for text to speech.

Location: ```piper:10200```



### Ollama Container

Use case: This is the LLM component. Make sure to download a suitable LLM for your hardware. Such as `docker exec ollama ollama pull llama3.1`. This requires the addition and configuration of the `Extended OpenAI Conversation` add on.

Load a model (such as dolphin-mixtral) into the Ollama container.

Example: ```docker exec ollama ollama pull dolphin-mixtral```

Location: ```ollama:11434```



### Speaker Recognition Container

Use case: This allows the voice assistant to recognise a familiar voice. Requires training which is a bigger topic. Used to allow or deny requests.

Location: ```http://speaker-recognition:8099```

TIP: HACS repository: `https://github.com/EuleMitKeule/speaker-recognition`



### Searxng Container

Use case: This allows the LLM to reach out to the Internet to get answers to questions unable to be answered locally.

Location: ```http://searxng:8080```



### Conversation Middleware Container

Use case: This allows the LLM to refer to a sqlite database to reference historical questions (not answers). Easily customised. Sits between HA and the LLM (Ollama)

Location: ```http://conversation_middleware:5000```



### Nginx (reverse proxy) Container

Use case: This is an optional component that allows for additional capabilities such as IP whitelisting, exposer to other networks, etc.

Location: ```http://ip-address:80```



# Test the LLM from the command line

Issue the following command to get a list of available models from the Ollama container:

Example: ```curl http://ip-address:11434/v1/models```

Issue the following command to submit a simple prompt directly to the Ollama model:

Example: ```curl http://ip-address:11434/v1/chat/completions -H "Content-Type: application/json" -d '{"model":"dolphin-mixtral:latest","messages":[{"role":"user","content":"Why is the sky blue?"}],"stream":false}'```

Issue the following command to test the web search feature directly:

Example: ```curl "http://ip-addresst:8080/search?q=what+is+today+date&format=json"```


# Under the hood (Scripts and Configuration files)



## Docker Compose file

The `docker-compose.yml` file looks like this.

Location: The root of your project (most likely):

```
version: '3.8'
services:

  conversation _middleware:
    image: python:3.11-slim
    container _name: conversation _middleware
    restart: unless-stopped
    working _dir: /app
    volumes:
      - . \docker _data _middleware:/app
    ports:
      - "5000:5000"
    environment:
      OLLAMA _URL: http://ollama:11434/v1
    networks:
      - ai _network
    entrypoint: sh -c "pip install --upgrade pip  & & pip install -r requirements.txt  & & python app.py"

  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    container _name: homeassistant
    restart: unless-stopped
    volumes:
      - . \docker _data _homeassistant:/config
    ports:
      - "8123:8123"
    networks:
      - ai _network
    entrypoint: >
      sh -c "if  [ ! -d '/config/custom _components/hacs' ]; then
               wget -O - https://get.hacs.xyz | bash -;
             fi;
             python3 -m homeassistant --config /config"

  whisper:
    image: rhasspy/wyoming-whisper
    container _name: whisper
    restart: unless-stopped
    command: --model base --language en
    ports:
      - "10300:10300"
    networks:
      - ai _network

  piper:
    image: rhasspy/wyoming-piper
    container _name: piper
    restart: unless-stopped
    command: --voice en _US-lessac-medium
    ports:
      - "10200:10200"
    networks:
      - ai _network

  ollama:
    image: ollama/ollama:latest
    container _name: ollama
    deploy:
      resources:
        reservations:
          devices:
            - capabilities:  ["gpu"]
    ports:
      - 11434:11434
    volumes:
      - . \docker _data _ollama:/root/.ollama
    restart: unless-stopped
    networks:
      - ai _network

  searxng:
    container _name: searxng
    image: searxng/searxng:latest
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - SEARXNG _BASE _URL=http://localhost:8080/
    volumes:
      - . \docker _data _searxng:/etc/searxng
    networks:
      - ai _network

  nginx:
    image: nginx:latest
    container _name: nginx _proxy
    restart: unless-stopped
    ports:
      - "0.0.0.0:80:80"
    volumes:
      - . \docker _data _nginx:/etc/nginx/conf.d
    networks:
      - ai _network

  speaker-recognition:
    image: ghcr.io/eulemitkeule/speaker-recognition:latest
    container _name: speaker-recognition
    restart: unless-stopped
    ports:
      - "8099:8099"
    environment:
      - TZ=Australia/Adelaide
      - LOG _LEVEL=info
      - EMBEDDINGS _DIR=/app/embeddings
    volumes:
      - . \docker _data _speaker-recognition:/app/embeddings
    networks:
      - ai _network

networks:
  ai _network:
    driver: bridge
```



## Home Assistant configuration.yaml file

This configuration file `configuration.yaml` may have some moving parts that relate to current or previous testing. It should always be in a working state, however.

Location: `docker _data _middleware/configuration.yaml`

```
homeassistant:
  media _dirs:
    local: /config/media

conversation:

recorder:
  db _url: sqlite:////config/home-assistant _v2.db
  purge _keep _days: 30
  auto _purge: true

default _config:

# Enable Voice Capabilities
assist _pipeline:
wyoming:

frontend:
  themes: !include _dir _merge _named themes

automation: !include automations.yaml
# script: !include scripts.yaml
scene: !include scenes.yaml

http:
  use _x _forwarded _for: true
  trusted _proxies:
    - 172.16.0.0/12  # Standard Docker network range
    - 127.0.0.1      # Localhost

template: !include templates.yaml

rest _command:
  searxng _search:
    url: "http://searxng:8080/search?q={{ query | urlencode }} &format=json &apikey=mysecret"
    method: GET
    timeout: 30

script:
  save _conversation:
    alias: "Save Conversation"
    sequence:
      - service: conversation.process
        data:
          text: "{{ trigger.payload _json.text }}"
          agent _id: conversation.extended _openai _conversation
        response _variable: response
      - service: file _operation.save _json
        data:
          filename: /config/conversation _history.json
          data: "{{ response }}"

  searxng _search _script:
    alias: "SearXNG Search"
    fields:
      query:
        description: "The search query"
    sequence:
      - action: rest _command.searxng _search
        data:
          query: "{{ query }}"
        response _variable: searx _output
      - variables:
          final _response:
            results: >-
              {% if searx _output.content.results is defined and searx _output.content.results|length > -1 %}
                {{ searx _output.content.results [0].title }}: {{ searx _output.content.results [0].content | truncate(1500) }}
              {% else %}
                No results found for "{{ query }}".
              {% endif %}
      - stop: "Success"
        response _variable: final _response
```



## The Conversation Middleware app.py script

This script `app.py` intercepts the prompt prior to the LLM model receiving it. It's a chance to reference historical prompts. At this start the history includes only the prompt but not the response from the LLM.

Location: `docker _data _middleware/app.py`

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
DEBUG_LOG = "/tmp/debug.log"

def log_debug(message):
    with open(DEBUG_LOG, 'a') as f:
        f.write(f"{datetime.now()} - {message}\n")

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
    
    relevant = []
    user_words = set(user_input.lower().split())
    
    for row in all_history:
        history_words = set(row['user_message'].lower().split())
        if user_words & history_words:
            relevant.append(row)
        if len(relevant) >= limit:
            break
    
    return relevant

@app.route('/chat', methods=['POST'])
def chat():
    data = request.json
    log_debug(f"[/chat] Received request")
    user_input = data.get('text')
    model = data.get('model', 'neural-chat')
    tools = data.get('tools', [])
    
    log_debug(f"[/chat] Tools/Functions in request: {tools}")
    log_debug(f"[/chat] Model: {model}, Input: {user_input[:50]}")
    
    history = get_relevant_history(user_input)
    
    context = "Previous related questions:\n"
    for row in reversed(history):
        context += f"- {row['user_message']}\n"
    
    full_prompt = context + f"\nCurrent question: {user_input}" if history else user_input
    
    log_debug(f"[/chat] Sending to Ollama")
    
    ollama_request = {
        "model": model,
        "messages": [{"role": "user", "content": full_prompt}]
    }
    
    if tools:
        ollama_request["tools"] = tools
        log_debug(f"[/chat] Including {len(tools)} tools in Ollama request")
    
    response = requests.post(
        f"{OLLAMA_URL}/chat/completions",
        json=ollama_request
    )
    
    log_debug(f"[/chat] Response status: {response.status_code}")
    
    try:
        response_json = response.json()
        log_debug(f"[/chat] Full response: {response_json}")
        log_debug(f"[/chat] Response keys: {list(response_json.keys())}")
    except Exception as e:
        log_debug(f"[/chat] ERROR parsing JSON: {e}")
        log_debug(f"[/chat] Raw response text: {response.text}")
        response_json = {}
    
    if 'choices' in response_json:
        assistant_response = response_json['choices'][0]['message']['content']
    else:
        log_debug(f"[/chat] ERROR: 'choices' not in response")
        assistant_response = "Error: Invalid response from Ollama"
    
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
    log_debug(f"[/chat/completions] Received request")
    messages = data.get('messages', [])
    model = data.get('model', 'neural-chat')
    tools = data.get('tools', [])
    user_input = messages[-1]['content'] if messages else ""
    
    log_debug(f"[/chat/completions] Tools/Functions in request: {tools}")
    log_debug(f"[/chat/completions] Model: {model}, Input: {user_input[:50]}")
    
    history = get_relevant_history(user_input)
    
    context = "Previous related questions:\n"
    for row in reversed(history):
        context += f"- {row['user_message']}\n"
    
    full_prompt = context + f"\nCurrent question: {user_input}" if history else user_input
    
    log_debug(f"[/chat/completions] Sending to Ollama")
    
    ollama_request = {
        "model": model,
        "messages": [{"role": "user", "content": full_prompt}]
    }
    
    if tools:
        ollama_request["tools"] = tools
        log_debug(f"[/chat/completions] Including {len(tools)} tools in Ollama request")
    
    response = requests.post(
        f"{OLLAMA_URL}/chat/completions",
        json=ollama_request
    )
    
    log_debug(f"[/chat/completions] Response status: {response.status_code}")
    
    try:
        response_json = response.json()
        log_debug(f"[/chat/completions] Full response: {response_json}")
        log_debug(f"[/chat/completions] Response keys: {list(response_json.keys())}")
    except Exception as e:
        log_debug(f"[/chat/completions] ERROR parsing JSON: {e}")
        log_debug(f"[/chat/completions] Raw response text: {response.text}")
        response_json = {}
    
    if 'choices' in response_json:
        assistant_response = response_json['choices'][0]['message']['content']
    else:
        log_debug(f"[/chat/completions] ERROR: 'choices' not in response")
        assistant_response = "Error: Invalid response from Ollama"
    
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
    log_debug(f"[/v1/chat/completions] Received request")
    messages = data.get('messages', [])
    model = data.get('model', 'neural-chat')
    tools = data.get('tools', [])
    user_input = messages[-1]['content'] if messages else ""
    
    log_debug(f"[/v1/chat/completions] Tools/Functions in request: {tools}")
    log_debug(f"[/v1/chat/completions] Model: {model}, Input: {user_input[:50]}")
    
    history = get_relevant_history(user_input)
    
    context = "Previous related questions:\n"
    for row in reversed(history):
        context += f"- {row['user_message']}\n"
    
    full_prompt = context + f"\nCurrent question: {user_input}" if history else user_input
    
    log_debug(f"[/v1/chat/completions] Sending to Ollama")
    
    ollama_request = {
        "model": model,
        "messages": [{"role": "user", "content": full_prompt}]
    }
    
    if tools:
        ollama_request["tools"] = tools
        log_debug(f"[/v1/chat/completions] Including {len(tools)} tools in Ollama request")
    
    response = requests.post(
        f"{OLLAMA_URL}/chat/completions",
        json=ollama_request
    )
    
    log_debug(f"[/v1/chat/completions] Response status: {response.status_code}")
    
    try:
        response_json = response.json()
        log_debug(f"[/v1/chat/completions] Full response: {response_json}")
        log_debug(f"[/v1/chat/completions] Response keys: {list(response_json.keys())}")
    except Exception as e:
        log_debug(f"[/v1/chat/completions] ERROR parsing JSON: {e}")
        log_debug(f"[/v1/chat/completions] Raw response text: {response.text}")
        response_json = {}
    
    if 'choices' in response_json:
        assistant_response = response_json['choices'][0]['message']['content']
    else:
        log_debug(f"[/v1/chat/completions] ERROR: 'choices' not in response")
        assistant_response = "Error: Invalid response from Ollama"
    
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
    log_debug("[STARTUP] Flask app starting")
    app.run(host='0.0.0.0', port=5000, debug=False)
```



## The Requirements for the Middleware capabilities

The middleware container uses a Python script `app.py` which relies on the requirements below. They are installed once at deployment time.

Location: `docker _data _middleware/requirements.txt`

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

The function gives coded instructions to the LLM model. This is where the LLM model is instructed to use the Internet `search_internet` feature to learn new information temporarily.

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

