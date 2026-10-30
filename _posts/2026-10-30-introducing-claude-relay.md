---
title: "claude-relay: Claude Code Behind an OpenAI-Compatible API"
date: 2026-10-30 12:00:00 +0530
categories: [Open Source]
tags: [python, claude, llm, self-hosted]
render_with_liquid: false
---

I've been working on a small open-source project called [claude-relay](https://github.com/anandhakrishnanaji/claude-relay). It is a self-hosted server that puts Claude Code behind two APIs most tools already know how to talk to: the OpenAI Chat Completions API (`/v1/chat/completions`) and the Anthropic Messages API (`/v1/messages`). Both support streaming. If a library or app lets you change the base URL, you can point it at claude-relay and your requests run through Claude Code.

![claude-relay demo](/assets/img/posts/2026-10-30-claude-relay-demo.gif)
_An OpenAI SDK client talking to claude-relay_

## Why I built it

I was building apps that call an LLM through the OpenAI or Anthropic SDK, and I wanted to test them against a real endpoint on my own machine while developing. Claude Code was already installed on my laptop, so I put a small server in front of it that speaks both APIs, pointed my app's base URL at `localhost`, and kept working.

That is still the use I'd recommend for subscription login: you, testing your own code, on your own machine. For anything shared or deployed, use an API key, Bedrock or Vertex.

The project started from Richard Atkinson's [claude-code-openai-wrapper](https://github.com/RichardAtCT/claude-code-openai-wrapper) (MIT), and I've since reworked most of it around safer defaults and packaging.

## How it works

claude-relay is a FastAPI app built on the official [Claude Agent SDK](https://pypi.org/project/claude-agent-sdk/). The SDK bundles the Claude Code CLI, so you don't need Node.js installed. Each request is translated into an Agent SDK call, and the reply is translated back into the response format the client expects, including token usage and stop reasons.

For credentials, use an Anthropic API key, AWS Bedrock, or Google Vertex AI. claude-relay detects which one you've configured. It can also use a Claude subscription login, but only for your own personal use on your own machine; read the [Terms Compliance](https://github.com/anandhakrishnanaji/claude-relay#terms-compliance) section of the README before doing that.

## Quick start

You need [uv](https://docs.astral.sh/uv/) (it fetches Python 3.10+ for you) and an API key.

```console
foo@bar:~$ export ANTHROPIC_API_KEY=sk-ant-...
foo@bar:~$ uvx claude-relay-server
```

That starts the server on `http://localhost:8000`. Now take any OpenAI SDK code and change the base URL:

```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8000/v1", api_key="not-needed")

reply = client.chat.completions.create(
    model="sonnet",
    messages=[{"role": "user", "content": "Hello, Claude"}],
)
print(reply.choices[0].message.content)
```

`api_key` can be anything unless you start the server with `--api-key` or `--generate-api-key`, in which case clients send that key as a bearer token. Pass `stream=True` to get server-sent events. The Anthropic SDK works the same way with `base_url="http://localhost:8000"`.

If you want it installed instead of fetched each time, `uv tool install claude-relay-server` (or `pipx install`) gives you a `claude-relay` command. The repo also has a script that installs it as a systemd user service, and a non-root Docker image published to `ghcr.io/anandhakrishnanaji/claude-relay`.

## Security defaults

Claude Code can run shell commands and edit files. An HTTP server in front of it is an easy way to hand that ability to whoever can send it a prompt, so I made the defaults strict and made you opt in to each risky piece.

**Tools are off.** Every request runs a single turn with no tools, so text in a prompt can't run commands, even if it comes from a web page or a document you pasted in. To give Claude its Bash, Read and Write tools, two things must be true: the server is started with `ALLOW_TOOLS=true`, and the request sends `"enable_tools": true`. A request asking for tools on a server that doesn't allow them gets a 403.

**It only listens on localhost.** The server binds to `127.0.0.1` and warns at startup if you expose it on another interface without an API key.

**No browser origins.** CORS allows no origins by default, so a random web page you visit can't call your local server from JavaScript. You add origins explicitly with `CORS_ORIGINS`.

**Host allow-list.** Requests with a `Host` header outside `ALLOWED_HOSTS` are rejected. This blocks DNS rebinding, where a malicious site points its own domain at `127.0.0.1` to get around the browser's same-origin checks.

**Isolated from your own Claude setup.** Requests don't load the host's `~/.claude/CLAUDE.md`, settings, skills or plugins, and Claude Code runs in a temporary working directory. Your personal configuration doesn't leak into API responses, and the server's behaviour doesn't change depending on whose machine it runs on.

> If you need any of these relaxed, each one has a setting. The README's [Configuration](https://github.com/anandhakrishnanaji/claude-relay#configuration) section lists them.
{: .prompt-info }

## What it doesn't do

claude-relay translates request formats. It does not make Claude Code a drop-in clone of the OpenAI API, and a few things don't map:

- OpenAI-style function calling (`tools` in the request) is not supported. Claude Code's own tools are the only tools.
- `temperature`, `top_p`, `max_tokens` and the penalty parameters are ignored, and `n > 1` isn't supported.
- Images in messages are replaced with text placeholders.
- Streaming works, but Claude Code hands over whole messages, so a short answer often arrives as a single chunk.
- Every request goes through Claude Code's agent loop, which adds latency. If you already have an API key and don't need Claude Code specifically, calling the Anthropic API directly is simpler and faster.

## Try it

The code is on GitHub at [anandhakrishnanaji/claude-relay](https://github.com/anandhakrishnanaji/claude-relay) under the MIT licence, and the package is on PyPI as [claude-relay-server](https://pypi.org/project/claude-relay-server/).

If you try it, I'd like to hear how it went: which client you connected, what broke, and which defaults got in your way. Issues and pull requests are welcome.
