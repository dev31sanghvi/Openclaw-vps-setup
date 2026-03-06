# OpenClaw VPS Setup — Locked Down & Private

So I wanted to self-host [OpenClaw](https://github.com/openclaw/openclaw) on my VPS without leaving anything exposed to the internet. The official docs are fine to get started, but I wasn't comfortable with how `docker.sock` gets shared — that's basically handing over root-level Docker access, and if that's open to the web, you're asking for trouble.

This repo is how I run it: everything's locked behind SSH tunnels, no public ports, no open firewall rules. If you're trying to do something similar, feel free to grab this and tweak it however you want.

## Why I went with Docker

I already have other stuff running on my VPS (a couple sites, a database, etc.) so I didn't want OpenClaw stepping on any of that. Docker keeps it in its own little box.

**What's nice about it:**
- Nothing clashes with your existing setup — node versions, dependencies, all isolated
- If the AI messes something up inside the container, just nuke it and restart. No harm done
- Even though we're sharing `docker.sock`, the container itself runs with pretty locked-down permissions

**The annoying parts:**
- Browser stuff (Puppeteer, Playwright) is a pain to get working inside a headless container
- Files the AI creates end up owned by the `node` user inside Docker, so editing them from the host can be a bit awkward
- The AI can't see anything else running on your VPS — it's blind to your other services

## What you'll need

- A Linux VPS (I'm on Ubuntu 24.04, 22.04 works too)
- SSH access
- [Docker](https://docs.docker.com/engine/install/) + [Docker Compose](https://docs.docker.com/compose/install/) installed

## Getting it running

**1. Clone this repo on your VPS:**
```bash
git clone https://github.com/dev31sanghvi/Openclaw-vps-setup.git openclaw-setup
cd openclaw-setup
```

**2. Make the data folders:**

These are where OpenClaw saves its config and workspace stuff:
```bash
mkdir openclaw-config openclaw-workspace
```

**3. Set up your `.env`:**
```bash
cp .env.example .env
nano .env
```
Put in your `OPENCLAW_GATEWAY_TOKEN` here. Don't worry about AI API keys — OpenClaw handles those separately through its own UI.

**4. Figure out your Docker group ID:**

OpenClaw needs to talk to Docker on the host to spin up sandboxes. Run this to get the group ID:
```bash
stat -c '%g' /var/run/docker.sock
```
Then add it to your `.env` like:
`DOCKER_GID=999`
(swap 999 with whatever number you got)

**5. Fire it up:**
```bash
docker compose up -d
```
Quick note — you do NOT need to open port 18789 in UFW or your firewall. The compose file binds to `127.0.0.1` only, so it's not reachable from outside.

## Getting to the web UI

Since nothing's exposed publicly, you can't just hit your VPS IP in a browser. You need an SSH tunnel.

### If you use regular SSH (password or default keys)

On your laptop, open a terminal and run:
```bash
ssh -L 18789:localhost:18789 root@your-vps-ip
```
Keep that terminal open — that's your tunnel. Now go to `http://localhost:18789` in your browser and log in with the gateway token you set earlier.

### If you use a `.pem` key file (AWS, EC2, etc.)

Same idea, just add the `-i` flag:
```bash
ssh -i /path/to/your/key.pem -L 18789:localhost:18789 root@your-vps-ip
```
Leave the terminal running, then open `http://localhost:18789`.

If you're not sure how `.pem` keys work, check your cloud provider's docs — AWS, DigitalOcean, etc. all have guides for this.

## Setting up AI models

OpenClaw keeps API keys in its own SQLite database (inside `openclaw-config`), so your `.env` stays clean.

### Paid APIs (OpenAI, Anthropic, etc.)
If you just want it to work fast and don't mind the cost:
1. Open the web UI at `http://localhost:18789`
2. Go to **Settings** > **AI Providers**
3. Drop in your API key (like your `sk-...` key from OpenAI)
4. Pick `gpt-4o` or `claude-3-5-sonnet` when you start a chat

### Free local models with Ollama
This setup comes with an Ollama container already wired up. You can pull open-source models and run them on your VPS for free — though heads up, it'll be slow without a GPU.

**Pull a model:**
```bash
docker exec -it ollama ollama run llama3:8b
```
Type `/bye` once it's done downloading to get out of the chat.

**Hook it up to OpenClaw:**

Since both containers are on the same Docker network, no API key needed.
1. In the web UI, go to **Settings** > **AI Providers** > **Ollama**
2. Set the URL to `http://ollama:11434` — don't use localhost here, `ollama` is the Docker DNS name
3. Put whatever you want in the API key field (like `ollama-local`), Ollama doesn't check it
4. Now you can pick your local model from the chat dropdown

If you decide you don't want Ollama anymore, just remove the `ollama:` block from `docker-compose.yml` and run `docker compose up -d`.

## Day-to-day commands

- Check logs: `docker compose logs -f`
- Stop everything: `docker compose down`
- Update to latest version:
  ```bash
  docker compose pull
  docker compose up -d
  ```

## Where's my data?

Everything OpenClaw saves goes into the folders you mapped — `OPENCLAW_CONFIG_DIR` and `OPENCLAW_WORKSPACE_DIR` in your `.env`. So even if you destroy the container, your data sticks around.

## Troubleshooting

### Permission denied error (`EACCES: ...node.json`)
This usually happens when you're running Docker as root. The folders you created end up owned by root, but OpenClaw runs as a non-root user called `node` (UID 1000) inside the container — so it can't write to them.

Fix it by changing ownership:
```bash
chown -R 1000:1000 ~/Tools/openclaw-config ~/Tools/openclaw-workspace
docker compose restart
```
