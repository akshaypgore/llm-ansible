**Audience:** Intermediate DevOps/Systems Engineers | **Series:** Part 1 of 4

Fun Part:- Chat with you own LLM without worrying about token expiration.
---

## Section 1 — Introduction

### 1.1 The 5 Layers of AI Ecosystem

| Layer | Role | Dev / Home Lab | Production |
|-------|------|----------------|------------|
| 5 | Applications | Simple chatbot scripts | RAG pipelines, Agents, Chatbots |
| 4 | Frameworks | LangChain, LlamaIndex | LangChain, LlamaIndex, LiteLLM |
| 3 | Model Serving | Ollama | vLLM, TGI, Triton |
| 2 | Models | phi3:mini, gemma:2b | Mistral 7B, Llama 3 70B |
| 1 | Infrastructure | VirtualBox VM,Mac Mini M-series, Local hardware | AWS/GCP/Azure, GPU servers |

> This post covers Layers 1, 2 and 3. Layers 4 and 5 will be covered in posts ahead.

### 1.2 What This Post Covers

- Setting up Ubuntu Server VM on VirtualBox. The server running the LLM.
- Installing and configuring Ollama as a systemd service. Ollama is a program which helps to manage LLM model.
- LLM model being used is phi which is superlight for homelab setup. It is similar to sonnet or gemini but on much smaller scale.
- Automating the entire setup with Ansible
- Interacting with the model via CLI, curl and Postman

> Flow of setup : Ansible running on user's system to configure Ubuntu VM (phi) to run LLM

---
## Section 2 — VM Setup

### 2.1 VM Specs

| Component | Spec | Reason |
|-----------|------|--------|
| RAM | 8GB | phi3:mini needs ~3.7GB in memory, leave headroom for OS |
| CPU | 4 cores | CPU inference benefits from multiple cores |
| Disk | 30GB | Model 2.2GB + Ubuntu OS + logs + breathing room |
| OS | Ubuntu minimal Server 22.04 LTS | Stable, well supported, no GUI overhead |
| Network | Bridged Adapter | VM gets own IP, allows Ansible and API calls from other machines/clients leveraging model |
| Hostname | phi | Named after the model running on it |


![Screenshot of virtual machine specs](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pidzsf16i9425ikxdeqb.png)

**Note** : One can leverage Mac Mini which has apple chips as it has UMA which is more capable of running models with higher transaction numbers.
Eg: Mac can run bit higher models like Llama 3 / 3.2, Mistral 7B . As I have a simple vm with no GPU, I am using basic LLM model called phi3:mini.

### 2.3 Hostname Setup
Named the vm phi. Will use this name ahead in ansible to keep things clean and simple.


![Screenshot of vm hostname](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/n33mbirdm3w622w9y5ry.png)

---
## Section 3 — Installing and Configuring Ollama manually

Walk through manual installation first so readers understand what Ansible automates in the next section.

### 3.1 Installation

- Single curl command install.
  `curl -fsSL https://ollama.com/install.sh | sh`
  **Ollama** : Is a free, open-source tool that allows you to easily download, set up, and run AI language models (like LLaMA 3, Mistral, and Gemma) directly.It acts like a "Docker for LLMs" managing the technical complexities so you can quickly run private, offline AI chat or coding assistants with a single command.
- Systemd service created automatically once the script completes successfully.

> 
![Screenshot of ollama service up and running](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ljimzk3spll613v4ykcj.png)



### 3.2 Systemd Override Configuration

- `OLLAMA_HOST=0.0.0.0` — To accept connections from all the clients in subnet which has model running.
- `OLLAMA_KEEP_ALIVE` — control model unload timeout. So if model is not queried for 5 mins, the OS will unload it from RAM automatically to free the OS.
- `StandardOutput` / `StandardError` — redirect logs to custom path. Try to put this on a separate partition other than root or entirely different disk.

**Note:** LLM models are loaded in RAM from disk before they are served. It has term called "warming up the model". In production setups something called heartbeat is used to keep model constantly warmed up and ready to serve as it affects the user experience.


![Screenshot of ollama config](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/lnm5nu1nv8o6azbr59im.png)
---
### 3.3 Log Configuration

- Create dir `/var/log/ollama` with correct `ollama:ollama` ownership
- Using custom log location to get all the logs as there is difference in verbosity. Journactl will filter the logs but we would need all the logs from stdout and stderr

### 3.4 Logrotate

- Config file at `/etc/logrotate.d/ollama`


![Screenshot of logroatate config](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5iar9m98vjlkw6gfffa1.png)


![Screenshot of logrotate service working as expected](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xxluakwrzgsislwobb61.png)
Few commands to use logrotate:
1. `logrotate --debug /etc/logrotate.d/ollama` - dry run
2. `logrotate --force /etc/logrotate.d/ollama` - force run
3. `ls -lh /var/log/ollama/` - check if logs rotated


![Screenshot of logrotate commands](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gd19nml5bw1rochmqx7n.png)

In above ss we can see the logs got rotated
---
### 3.5 Pull and Test Model

- `ollama pull phi3:mini`
- `ollama list` — verify download
- `ollama run phi3:mini` — quick interactive test

Your privale llm model is up and running. Ready to answer your queries.
![Screenshot of ollama basic commands](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/e9ojv465sco6zq73vy2h.png)
---
## Section 4 — Automating with Ansible

Now that we understand every manual step, lets automate it all.

### 4.1 Repository Structure


![Screenshot of directory structure of llm ansible repo](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/w8jaz3gk8tuyqlj4cilh.png)

### 4.2 Running the Playbook

1. Dry run 
`ansible-playbook -i inventory.ini playbook.yml --check`

![Screenshot of ansible dry run](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/rvpozm0lmzoqzx9r0y55.png)

Faced an error here because the service was not installed yet. This is handled in playbook.

2. Run the playbook
`ansible-playbook -i inventory.ini playbook.yml`

![Screenshot of ansible being executed](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/rupwhd801fgvywhw9yg8.png)

### 4.3 GitHub Repo
---
{% github https://github.com/akshaypgore/llm-ansible %}

---
## Section 5 — Interacting with the Model
Two ways to interact — CLI and curl. Each progressively more useful for building applications.

### 5.1 CLI — ollama run
```
ollama --version
ollama list
ollama ps
ollama run phi3:mini
ollama show phi3:mini
```

![Screenshot of ollama commands executed](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/y3thg0gfyosz7tz7dkro.png)

In the above image we can see that ollama unloaded the model as it was not being used. We had to run `ollama run phi3:mini` to reload the model in RAM which is also called warming up.

### 5.2 REST API via curl
This is the important part — how applications actually talk to Ollama. Below are few endpoints which are exposed

- `/api/generate` — single prompt
- `/api/chat` — conversation with history and roles
```
1. curl http://localhost:11434 #check if model is running
2. curl http://localhost:11434/api/generate #prompt like experience. Ask a question, model answers
3. #interaction with LLM like a chat. Question and Anser
curl http://localhost:11434/api/chat \
  -d '{
    "model": "phi3:mini",
    "stream": false,
    "messages": [
      {
        "role": "user",
        "content": "What is Linux?"
      },
      {
        "role": "assistant",
        "content": "Linux is an open source operating system..."
      },
      {
        "role": "user",
        "content": "Who created it?"
      }
    ]
  }' | python3 -c "import sys,json; print(json.load(sys.stdin)['message']['content'])"
```

![Screenshot of interaction with ollama](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/993iyaegthatssjoimiw.png)

##### Some interaction with our own LLM model.

![Screenshot of interaction with LLM](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/2d3k57r9qor3yvc1texr.png)

There are many more important elemets to discuss ahead in future posts. Like
#### Performance Metrics of LLM
1. eval_count : Number of tokens (words) generated
2. eval_duration : Time required to generate tokens
3. total_duration : Time required to execute the query
#### Context and cost tracking
1. prompt_eval_count : Tokens consumed in input along with tokens in chat history
2. load_duration : Time to load model in memory of server
