
================================================================================
FILE: package.json
================================================================================
{
  "name": "adhiraj-ai-website",
  "version": "1.0.0",
  "private": true,
  "description": "Adhiraj AI - voice-enabled AI assistant with image analysis",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "node --watch server.js"
  },
  "engines": {
    "node": ">=20"
  },
  "dependencies": {
    "dotenv": "^16.4.7",
    "express": "^4.21.2",
    "express-rate-limit": "^7.5.0",
    "helmet": "^8.0.0",
    "openai": "^4.77.0"
  }
}

================================================================================
FILE: .env.example
================================================================================
# Copy this file to .env and add your own API key.
# Never commit or publish .env.
OPENAI_API_KEY=replace_with_your_secret_api_key
OPENAI_MODEL=gpt-4.1-mini
PORT=3000
NODE_ENV=development

================================================================================
FILE: server.js
================================================================================
require("dotenv").config();

const express = require("express");
const helmet = require("helmet");
const rateLimit = require("express-rate-limit");
const path = require("path");
const OpenAI = require("openai");

const app = express();
const PORT = Number(process.env.PORT || 3000);
const MODEL = process.env.OPENAI_MODEL || "gpt-4.1-mini";

app.disable("x-powered-by");
app.use(helmet({
  // The site uses inline styles only in the stylesheet and local scripts.
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'"],
      imgSrc: ["'self'", "data:", "blob:"],
      connectSrc: ["'self'"],
      mediaSrc: ["'self'", "blob:"],
      objectSrc: ["'none'"],
      frameAncestors: ["'none'"]
    }
  }
}));
app.use(express.json({ limit: "8mb" }));
app.use(express.static(path.join(__dirname, "public")));

const apiLimiter = rateLimit({
  windowMs: 60 * 1000,
  limit: 20,
  standardHeaders: "draft-7",
  legacyHeaders: false,
  message: { error: "Too many requests. Please wait a minute and try again." }
});
app.use("/api/", apiLimiter);

function getClient() {
  if (!process.env.OPENAI_API_KEY) {
    const err = new Error("The server administrator has not configured OPENAI_API_KEY.");
    err.status = 503;
    throw err;
  }
  return new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
}

function validateMessages(messages) {
  if (!Array.isArray(messages) || messages.length === 0 || messages.length > 30) {
    return "Please send between 1 and 30 messages.";
  }
  for (const item of messages) {
    if (!item || !["user", "assistant"].includes(item.role) ||
        typeof item.content !== "string" || item.content.length > 12000) {
      return "A message is invalid or too long.";
    }
  }
  return null;
}

app.get("/api/health", (_req, res) => {
  res.json({ ok: true, name: "Adhiraj AI" });
});

app.post("/api/chat", async (req, res) => {
  try {
    const { messages } = req.body || {};
    const invalid = validateMessages(messages);
    if (invalid) return res.status(400).json({ error: invalid });

    const client = getClient();
    const response = await client.responses.create({
      model: MODEL,
      instructions:
        "Your name is Adhiraj AI. Be helpful, friendly, and clear. " +
        "Answer in the user's language when practical. Do not claim you have " +
        "performed real-world actions unless a connected tool actually did them. " +
        "If uncertain, say so. Keep spoken answers reasonably concise unless asked for detail.",
      input: messages.map(m => ({ role: m.role, content: m.content }))
    });

    res.json({ answer: response.output_text || "I couldn't create an answer. Please try again." });
  } catch (err) {
    console.error("Chat API error:", err.message);
    const status = err.status && err.status >= 400 && err.status < 600 ? err.status : 500;
    res.status(status).json({
      error: status === 401 ? "The AI API key is invalid. Check the server configuration."
        : status === 429 ? "The AI service is busy or its usage limit was reached. Please try later."
        : status === 503 ? err.message
        : "Sorry, I couldn't reach the AI service. Please try again."
    });
  }
});

app.post("/api/analyze-image", async (req, res) => {
  try {
    const { imageData, mimeType, question } = req.body || {};
    const allowedTypes = ["image/jpeg", "image/png", "image/webp", "image/gif"];
    if (typeof imageData !== "string" || imageData.length > 6_000_000) {
      return res.status(400).json({ error: "Please choose an image smaller than about 4 MB." });
    }
    if (!allowedTypes.includes(mimeType)) {
      return res.status(400).json({ error: "Use a JPG, PNG, WEBP, or GIF image." });
    }
    if (imageData.includes("base64,") || !/^[A-Za-z0-9+/=\s]+$/.test(imageData)) {
      return res.status(400).json({ error: "The image data is not valid." });
    }

    const client = getClient();
    const dataUrl = `data:${mimeType};base64,${imageData}`;
    const prompt = typeof question === "string" && question.trim()
      ? question.trim().slice(0, 2000)
      : "Describe this image and identify the visible objects. Do not guess details that cannot be seen.";

    const response = await client.responses.create({
      model: MODEL,
      instructions: "You are Adhiraj AI. Analyze only what is visible in the supplied image. Be clear about uncertainty.",
      input: [{
        role: "user",
        content: [
          { type: "input_text", text: prompt },
          { type: "input_image", image_url: dataUrl }
        ]
      }]
    });

    res.json({ answer: response.output_text || "I couldn't analyze this image. Please try again." });
  } catch (err) {
    console.error("Image API error:", err.message);
    const status = err.status && err.status >= 400 && err.status < 600 ? err.status : 500;
    res.status(status).json({
      error: status === 401 ? "The AI API key is invalid. Check the server configuration."
        : status === 429 ? "The AI service is busy or its usage limit was reached. Please try later."
        : "Sorry, I couldn't analyze that image. Please try another image."
    });
  }
});

app.get("*", (_req, res) => {
  res.sendFile(path.join(__dirname, "public", "index.html"));
});

app.listen(PORT, () => {
  console.log(`Adhiraj AI is running at http://localhost:${PORT}`);
});


================================================================================
FILE: public/index.html
================================================================================
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <meta name="theme-color" content="#ffffff">
  <meta name="description" content="Adhiraj AI — chat, talk, and ask questions about pictures.">
  <title>Adhiraj AI</title>
  <link rel="stylesheet" href="/styles.css">
  <script src="/app.js" defer></script>
</head>
<body>
  <div class="app-shell">
    <aside class="sidebar" id="sidebar">
      <div class="brand">
        <div class="brand-mark" aria-hidden="true">A</div>
        <div><strong>Adhiraj AI</strong><span>Your AI assistant</span></div>
      </div>
      <button class="new-chat" id="newChatBtn" type="button"><span>＋</span> New chat</button>
      <div class="sidebar-label">YOUR ASSISTANT</div>
      <div class="sidebar-note"><span class="status-dot"></span> Ready to help</div>
      <div class="sidebar-bottom">
        <div class="privacy-icon">◇</div>
        <p><strong>Your conversations</strong><br>Stay in this browser tab. Avoid sharing private or sensitive information.</p>
        <button class="clear-btn" id="clearBtn" type="button">Clear chat</button>
      </div>
    </aside>

    <main class="main">
      <header class="topbar">
        <button class="menu-btn" id="menuBtn" type="button" aria-label="Toggle menu">☰</button>
        <div class="mobile-brand"><span class="mini-mark">A</span> Adhiraj AI</div>
        <div class="topbar-right"><span class="secure-dot"></span><span>AI assistant</span></div>
      </header>

      <section class="conversation" id="conversation" aria-live="polite">
        <div class="welcome" id="welcome">
          <div class="welcome-mark">A</div>
          <p class="eyebrow">MEET YOUR AI ASSISTANT</p>
          <h1>What can I help you<br><span>with today?</span></h1>
          <p class="welcome-subtitle">Ask a question, talk naturally, or upload an image to explore it together.</p>
          <div class="suggestions">
            <button class="suggestion" data-prompt="Explain artificial intelligence in simple words."><span class="suggestion-icon">✦</span><span><strong>Explain something</strong><small>Make a difficult topic simple</small></span><span class="arrow">↗</span></button>
            <button class="suggestion" data-prompt="Give me five creative ideas for a small business."><span class="suggestion-icon">↗</span><span><strong>Brainstorm ideas</strong><small>Explore new possibilities</small></span><span class="arrow">↗</span></button>
            <button class="suggestion" data-prompt="Help me make a simple plan for today."><span class="suggestion-icon">☷</span><span><strong>Plan my day</strong><small>Turn goals into next steps</small></span><span class="arrow">↗</span></button>
            <button class="suggestion" id="suggestImage"><span class="suggestion-icon">▧</span><span><strong>Understand a picture</strong><small>Upload an image to ask about it</small></span><span class="arrow">↗</span></button>
          </div>
        </div>
        <div class="messages" id="messages"></div>
        <div class="typing" id="typing" hidden><span></span><span></span><span></span><small>Adhiraj AI is thinking…</small></div>
      </section>

      <footer class="composer-area">
        <div class="image-preview" id="imagePreview" hidden>
          <img id="previewImg" alt="Selected image preview">
          <div><strong id="previewName">Image selected</strong><small>Ready to analyze</small></div>
          <button id="removeImageBtn" type="button" aria-label="Remove image">×</button>
        </div>
        <form class="composer" id="chatForm">
          <textarea id="messageInput" rows="1" placeholder="Message Adhiraj AI…" aria-label="Message Adhiraj AI"></textarea>
          <div class="composer-toolbar">
            <div class="tools">
              <button class="tool-btn" id="uploadBtn" type="button" title="Upload image"><span>＋</span><span class="tool-label">Add image</span></button>
              <input type="file" id="imageInput" accept="image/jpeg,image/png,image/webp,image/gif" hidden>
              <button class="tool-btn mic-btn" id="micBtn" type="button" title="Use voice" aria-label="Use voice"><span id="micIcon">🎙</span><span class="tool-label" id="micLabel">Voice</span></button>
            </div>
            <button class="send-btn" id="sendBtn" type="submit" aria-label="Send message"><span>↑</span></button>
          </div>
        </form>
        <p class="disclaimer">Adhiraj AI can make mistakes. Check important information. <span>·</span> Voice recognition depends on your browser.</p>
      </footer>
    </main>
  </div>
  <div class="toast" id="toast" role="status" aria-live="polite"></div>
</body>
</html>


================================================================================
FILE: public/styles.css
================================================================================
:root{font-family:Inter,-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,Arial,sans-serif;color:#24252b;background:#fff;font-synthesis:none;text-rendering:optimizeLegibility;-webkit-font-smoothing:antialiased;--ink:#25262c;--muted:#777984;--line:#e9e9ed;--soft:#f7f7f9;--accent:#5957d5;--accent-dark:#4643bf}
*{box-sizing:border-box}body{margin:0;min-height:100vh}.app-shell{display:flex;min-height:100vh}.sidebar{width:258px;flex:0 0 258px;border-right:1px solid var(--line);background:#fcfcfd;padding:25px 17px 18px;display:flex;flex-direction:column}.brand{display:flex;align-items:center;gap:11px;padding:0 7px 29px}.brand-mark,.welcome-mark,.mini-mark{display:grid;place-items:center;background:linear-gradient(140deg,#7774ee,#4d4ac8);color:#fff;font-weight:750}.brand-mark{width:39px;height:39px;border-radius:13px;font-size:21px;box-shadow:0 5px 12px #5957d52b}.brand strong{display:block;font-size:15px;letter-spacing:-.2px}.brand span{display:block;color:var(--muted);font-size:11px;margin-top:4px}.new-chat{width:100%;height:43px;border:1px solid #e4e3f9;border-radius:11px;background:#f2f1ff;color:#4d4ac8;font-weight:650;text-align:left;padding:0 13px;cursor:pointer}.new-chat span{font-size:20px;vertical-align:-1px;margin-right:8px}.sidebar-label{margin:31px 9px 12px;font-size:10px;letter-spacing:1.3px;color:#999aa4;font-weight:750}.sidebar-note{font-size:12px;color:#666873;padding:10px 9px;display:flex;align-items:center;gap:8px}.status-dot,.secure-dot{display:inline-block;width:7px;height:7px;background:#38b980;border-radius:50%}.sidebar-bottom{margin-top:auto;border-top:1px solid var(--line);padding:19px 8px 0;position:relative}.privacy-icon{width:29px;height:29px;border-radius:9px;display:grid;place-items:center;background:#f0efff;color:#5d59d4;font-size:21px}.sidebar-bottom p{font-size:11px;line-height:1.7;color:#777984;margin:10px 0 13px}.sidebar-bottom p strong{color:#40414a;font-size:12px}.clear-btn{border:0;background:none;color:#777984;padding:4px 0;font-size:11px;cursor:pointer}.clear-btn:hover{color:#4d4ac8}.main{display:flex;flex-direction:column;min-width:0;flex:1;height:100vh}.topbar{height:66px;border-bottom:1px solid #f0f0f2;display:flex;align-items:center;justify-content:space-between;padding:0 34px;flex-shrink:0}.topbar-right{display:flex;align-items:center;gap:8px;color:#777984;font-size:12px}.menu-btn,.mobile-brand{display:none}.conversation{flex:1;overflow-y:auto;padding:44px 24px 20px}.welcome{max-width:760px;margin:22px auto 0;text-align:center}.welcome-mark{width:56px;height:56px;border-radius:19px;font-size:30px;margin:0 auto 24px;box-shadow:0 8px 20px #5957d526}.eyebrow{font-size:10px;letter-spacing:1.8px;color:#8583db;font-weight:750;margin:0 0 15px}.welcome h1{font-size:clamp(30px,4vw,43px);letter-spacing:-1.6px;line-height:1.16;margin:0;color:#282932;font-weight:730}.welcome h1 span{color:#6663d8}.welcome-subtitle{font-size:14px;line-height:1.7;color:#858691;max-width:440px;margin:16px auto 31px}.suggestions{display:grid;grid-template-columns:1fr 1fr;gap:12px;text-align:left;max-width:650px;margin:0 auto}.suggestion{display:flex;align-items:center;gap:12px;min-height:76px;border:1px solid #eaeaf0;background:#fff;border-radius:13px;padding:13px 14px;text-align:left;cursor:pointer;transition:border .15s,transform .15s,box-shadow .15s}.suggestion:hover{border-color:#c9c7f5;transform:translateY(-1px);box-shadow:0 6px 18px #25255009}.suggestion-icon{width:34px;height:34px;display:grid;place-items:center;flex-shrink:0;border-radius:10px;background:#f2f1ff;color:#6663d8;font-size:18px}.suggestion strong{display:block;font-size:12px;color:#3c3d46;font-weight:650}.suggestion small{display:block;font-size:10px;color:#92939d;margin-top:5px}.arrow{margin-left:auto;color:#9b9ca6;font-size:15px}.messages{max-width:780px;margin:0 auto;display:flex;flex-direction:column;gap:24px}.message{display:flex;gap:12px;align-items:flex-start;max-width:100%;animation:fadein .2s ease}.message.user{justify-content:flex-end}.avatar{width:30px;height:30px;flex:0 0 30px;border-radius:10px;background:#efeeff;color:#5957d5;display:grid;place-items:center;font-weight:750;font-size:14px}.message-content{min-width:0;max-width:calc(100% - 42px);font-size:14px;line-height:1.75;overflow-wrap:anywhere}.message.user .message-content{background:#f3f3f7;border-radius:17px 17px 5px 17px;padding:11px 15px;max-width:78%;white-space:pre-wrap}.message.assistant .message-content{padding:2px 0;color:#33343c;white-space:pre-wrap}.message-content img{max-width:min(100%,300px);max-height:260px;object-fit:contain;border-radius:10px;display:block;margin-bottom:8px}.message-actions{display:flex;gap:10px;margin-top:7px}.message-actions button{background:transparent;border:0;color:#898a95;font-size:11px;cursor:pointer;padding:3px 0}.message-actions button:hover{color:#5653ca}.typing{max-width:780px;margin:16px auto;display:flex;align-items:center;gap:5px;color:#92939d;font-size:12px}.typing span{width:6px;height:6px;background:#7774dc;border-radius:50%;animation:bounce 1s infinite}.typing span:nth-child(2){animation-delay:.15s}.typing span:nth-child(3){animation-delay:.3s}.typing small{margin-left:6px}.composer-area{width:min(820px,calc(100% - 48px));margin:0 auto;padding:12px 0 16px;flex-shrink:0}.composer{border:1px solid #e3e3e9;border-radius:17px;background:#fff;padding:12px 13px 10px;box-shadow:0 6px 28px #24244808;transition:border .15s,box-shadow .15s}.composer:focus-within{border-color:#c4c2f3;box-shadow:0 0 0 3px #6b68db0b}textarea{font:inherit;font-size:14px;color:#33343c;resize:none;border:0;outline:0;width:100%;min-height:30px;max-height:160px;line-height:1.55;padding:3px 4px 9px;background:transparent}textarea::placeholder{color:#a0a1aa}.composer-toolbar{display:flex;align-items:center;justify-content:space-between}.tools{display:flex;gap:8px}.tool-btn{border:0;background:transparent;color:#777984;display:flex;align-items:center;gap:6px;padding:7px 9px;border-radius:8px;font-size:11px;cursor:pointer}.tool-btn:hover,.tool-btn.listening{background:#f1f0ff;color:#5754d1}.tool-btn>span:first-child{font-size:16px}.send-btn{width:34px;height:34px;border:0;border-radius:11px;background:#5957d5;color:#fff;display:grid;place-items:center;cursor:pointer;font-size:22px;transition:background .15s}.send-btn:hover{background:var(--accent-dark)}.send-btn:disabled{opacity:.5;cursor:not-allowed}.disclaimer{text-align:center;font-size:10px;color:#a0a1a9;margin:10px 0 0}.disclaimer span{padding:0 4px}.image-preview{display:flex;align-items:center;gap:10px;padding:9px 12px;border:1px solid var(--line);border-radius:12px;margin-bottom:8px;background:#fff}.image-preview img{width:43px;height:43px;object-fit:cover;border-radius:7px}.image-preview strong,.image-preview small{display:block;font-size:11px}.image-preview small{color:#92939d;margin-top:4px}.image-preview button{margin-left:auto;border:0;background:#f1f1f4;border-radius:50%;width:26px;height:26px;font-size:18px;color:#666;cursor:pointer}.toast{position:fixed;bottom:24px;left:50%;transform:translate(-50%,15px);opacity:0;pointer-events:none;background:#292a31;color:#fff;padding:11px 16px;border-radius:10px;font-size:12px;box-shadow:0 8px 25px #0002;transition:all .2s;max-width:calc(100% - 30px);z-index:10}.toast.show{opacity:1;transform:translate(-50%,0)}[hidden]{display:none!important}@keyframes bounce{0%,60%,100%{transform:translateY(0);opacity:.6}30%{transform:translateY(-4px);opacity:1}}@keyframes fadein{from{opacity:0;transform:translateY(4px)}to{opacity:1;transform:translateY(0)}}
@media(max-width:760px){.sidebar{position:fixed;z-index:8;inset:0 auto 0 0;width:270px;transform:translateX(-102%);transition:transform .2s;box-shadow:none}.sidebar.open{transform:translateX(0);box-shadow:10px 0 40px #0001}.main{height:100dvh}.topbar{height:58px;padding:0 16px}.menu-btn{display:block;border:0;background:transparent;color:#60616b;font-size:20px;padding:6px 9px}.mobile-brand{display:flex;align-items:center;gap:8px;font-size:14px;font-weight:700;margin-right:auto;margin-left:12px}.mini-mark{width:26px;height:26px;border-radius:8px;font-size:14px}.topbar-right{font-size:11px}.conversation{padding:28px 16px 12px}.welcome{margin:15px auto 0}.welcome-mark{margin-bottom:20px}.welcome h1{font-size:32px;letter-spacing:-1px}.welcome-subtitle{font-size:13px;margin:14px auto 24px}.suggestions{grid-template-columns:1fr;gap:9px}.suggestion{min-height:66px;padding:10px 12px}.composer-area{width:calc(100% - 24px);padding:8px 0 10px}.disclaimer{font-size:9px;line-height:1.5}.message-content{font-size:13px}.message.user .message-content{max-width:88%}}@media(min-width:761px) and (max-height:700px){.conversation{padding-top:20px}.welcome{margin-top:4px}.welcome-mark{width:45px;height:45px;margin-bottom:15px}.welcome h1{font-size:34px}.welcome-subtitle{margin:10px auto 18px}.suggestion{min-height:65px}}


================================================================================
FILE: public/app.js
================================================================================
const conversationEl = document.getElementById("conversation");
const welcomeEl = document.getElementById("welcome");
const messagesEl = document.getElementById("messages");
const typingEl = document.getElementById("typing");
const form = document.getElementById("chatForm");
const input = document.getElementById("messageInput");
const sendBtn = document.getElementById("sendBtn");
const imageInput = document.getElementById("imageInput");
const imagePreview = document.getElementById("imagePreview");
const previewImg = document.getElementById("previewImg");
const previewName = document.getElementById("previewName");
const toastEl = document.getElementById("toast");
const micBtn = document.getElementById("micBtn");
const micLabel = document.getElementById("micLabel");

let history = [];
let selectedImage = null;
let isBusy = false;
let recognition = null;
let isListening = false;
let toastTimer = null;

function toast(message) {
  toastEl.textContent = message;
  toastEl.classList.add("show");
  clearTimeout(toastTimer);
  toastTimer = setTimeout(() => toastEl.classList.remove("show"), 3500);
}

function scrollToBottom() {
  conversationEl.scrollTop = conversationEl.scrollHeight;
}

function setBusy(value) {
  isBusy = value;
  sendBtn.disabled = value;
  typingEl.hidden = !value;
  if (value) scrollToBottom();
}

function showConversation() {
  welcomeEl.hidden = true;
}

function addMessage(role, text, imageUrl = null) {
  showConversation();
  const row = document.createElement("div");
  row.className = `message ${role}`;

  if (role === "assistant") {
    const avatar = document.createElement("div");
    avatar.className = "avatar";
    avatar.textContent = "A";
    row.appendChild(avatar);
  }

  const body = document.createElement("div");
  body.className = "message-content";

  if (imageUrl) {
    const img = document.createElement("img");
    img.src = imageUrl;
    img.alt = "Image sent for analysis";
    body.appendChild(img);
  }

  const textNode = document.createElement("div");
  textNode.textContent = text;
  body.appendChild(textNode);

  if (role === "assistant") {
    const actions = document.createElement("div");
    actions.className = "message-actions";
    const speakButton = document.createElement("button");
    speakButton.type = "button";
    speakButton.textContent = "🔊 Read aloud";
    speakButton.addEventListener("click", () => speakText(text));
    const copyButton = document.createElement("button");
    copyButton.type = "button";
    copyButton.textContent = "Copy";
    copyButton.addEventListener("click", async () => {
      try {
        await navigator.clipboard.writeText(text);
        toast("Answer copied.");
      } catch {
        toast("Copy isn't available in this browser.");
      }
    });
    actions.append(speakButton, copyButton);
    body.appendChild(actions);
  }

  row.appendChild(body);
  messagesEl.appendChild(row);
  scrollToBottom();
}

function speakText(text) {
  if (!("speechSynthesis" in window)) {
    toast("Speech playback is not supported in this browser.");
    return;
  }
  window.speechSynthesis.cancel();
  const utterance = new SpeechSynthesisUtterance(text);
  utterance.lang = "en-IN";
  utterance.rate = 1;
  window.speechSynthesis.speak(utterance);
}

function resizeInput() {
  input.style.height = "auto";
  input.style.height = `${Math.min(input.scrollHeight, 160)}px`;
}
input.addEventListener("input", resizeInput);
input.addEventListener("keydown", event => {
  if (event.key === "Enter" && !event.shiftKey) {
    event.preventDefault();
    form.requestSubmit();
  }
});

async function sendChat(text) {
  const clean = text.trim();
  if (!clean || isBusy) return;

  const image = selectedImage;
  addMessage("user", clean, image ? URL.createObjectURL(image) : null);
  input.value = "";
  resizeInput();
  clearSelectedImage();
  setBusy(true);

  const updatedHistory = [...history, { role: "user", content: clean }].slice(-20);
  try {
    const response = await fetch("/api/chat", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ messages: updatedHistory })
    });
    const data = await response.json();
    if (!response.ok) throw new Error(data.error || "The request failed.");
    const answer = data.answer || "I couldn't create an answer.";
    history = [...updatedHistory, { role: "assistant", content: answer }].slice(-20);
    addMessage("assistant", answer);
  } catch (error) {
    addMessage("assistant", `Sorry — ${error.message || "something went wrong."}`);
  } finally {
    setBusy(false);
    input.focus();
  }
}

async function analyzeSelectedImage(question) {
  if (!selectedImage || isBusy) return;
  const image = selectedImage;
  const previewUrl = URL.createObjectURL(image);
  addMessage("user", question || "Please describe this picture.", previewUrl);
  clearSelectedImage();
  setBusy(true);

  try {
    const base64 = await fileToBase64(image);
    const response = await fetch("/api/analyze-image", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        imageData: base64,
        mimeType: image.type,
        question: question || "Describe this image and identify the visible objects."
      })
    });
    const data = await response.json();
    if (!response.ok) throw new Error(data.error || "Image analysis failed.");
    addMessage("assistant", data.answer || "I couldn't analyze this image.");
  } catch (error) {
    addMessage("assistant", `Sorry — ${error.message || "I couldn't analyze that image."}`);
  } finally {
    setBusy(false);
    URL.revokeObjectURL(previewUrl);
  }
}

function fileToBase64(file) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => resolve(String(reader.result).split(",")[1]);
    reader.onerror = () => reject(new Error("Could not read that image."));
    reader.readAsDataURL(file);
  });
}

function setSelectedImage(file) {
  if (!file) return;
  const allowed = ["image/jpeg", "image/png", "image/webp", "image/gif"];
  if (!allowed.includes(file.type)) {
    toast("Please choose a JPG, PNG, WEBP, or GIF image.");
    return;
  }
  if (file.size > 4 * 1024 * 1024) {
    toast("Please choose an image smaller than 4 MB.");
    return;
  }
  selectedImage = file;
  previewImg.src = URL.createObjectURL(file);
  previewName.textContent = file.name || "Selected image";
  imagePreview.hidden = false;
  input.placeholder = "Ask a question about the image…";
  input.focus();
}

function clearSelectedImage() {
  selectedImage = null;
  imagePreview.hidden = true;
  previewImg.removeAttribute("src");
  imageInput.value = "";
  input.placeholder = "Message Adhiraj AI…";
}

form.addEventListener("submit", async event => {
  event.preventDefault();
  const text = input.value.trim();
  if (selectedImage) {
    await analyzeSelectedImage(text || "Describe this image and tell me what is visible.");
    input.value = "";
    resizeInput();
  } else {
    await sendChat(text);
  }
});

document.getElementById("uploadBtn").addEventListener("click", () => imageInput.click());
document.getElementById("suggestImage").addEventListener("click", () => imageInput.click());
imageInput.addEventListener("change", () => setSelectedImage(imageInput.files[0]));
document.getElementById("removeImageBtn").addEventListener("click", clearSelectedImage);

document.querySelectorAll("[data-prompt]").forEach(button => {
  button.addEventListener("click", () => sendChat(button.dataset.prompt));
});

function resetChat() {
  if (isBusy) return toast("Please wait for the current answer to finish.");
  history = [];
  messagesEl.replaceChildren();
  welcomeEl.hidden = false;
  clearSelectedImage();
  input.value = "";
  resizeInput();
  if ("speechSynthesis" in window) window.speechSynthesis.cancel();
}
document.getElementById("newChatBtn").addEventListener("click", resetChat);
document.getElementById("clearBtn").addEventListener("click", resetChat);

const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
if (SpeechRecognition) {
  recognition = new SpeechRecognition();
  recognition.lang = "en-IN";
  recognition.interimResults = true;
  recognition.continuous = false;

  recognition.onstart = () => {
    isListening = true;
    micBtn.classList.add("listening");
    micLabel.textContent = "Listening…";
    document.getElementById("micIcon").textContent = "●";
  };
  recognition.onresult = event => {
    let transcript = "";
    for (let i = event.resultIndex; i < event.results.length; i++) {
      transcript += event.results[i][0].transcript;
    }
    input.value = transcript;
    resizeInput();
  };
  recognition.onerror = event => {
    if (event.error === "not-allowed") toast("Please allow microphone access in your browser.");
    else if (event.error !== "no-speech" && event.error !== "aborted") toast("Voice recognition error: " + event.error);
  };
  recognition.onend = () => {
    isListening = false;
    micBtn.classList.remove("listening");
    micLabel.textContent = "Voice";
    document.getElementById("micIcon").textContent = "🎙";
    if (input.value.trim() && !isBusy) form.requestSubmit();
  };
  micBtn.addEventListener("click", () => {
    if (isListening) recognition.stop();
    else {
      try { recognition.start(); }
      catch { toast("Voice recognition is starting. Please try again."); }
    }
  });
} else {
  micBtn.addEventListener("click", () => toast("Voice recognition works best in Chrome or Edge. You can type instead."));
  micBtn.title = "Voice recognition not supported in this browser";
}

document.getElementById("menuBtn").addEventListener("click", () => {
  document.getElementById("sidebar").classList.toggle("open");
});
document.querySelector(".main").addEventListener("click", event => {
  if (event.target.closest("button") && window.innerWidth <= 760) {
    // Keep the menu open only while interacting with its own controls.
    if (!event.target.closest("#menuBtn")) document.getElementById("sidebar").classList.remove("open");
  }
});

fetch("/api/health").catch(() => toast("Cannot reach the server. Check your connection."));


================================================================================
FILE: render.yaml
================================================================================
services:
  - type: web
    name: adhiraj-ai
    runtime: node
    plan: starter
    buildCommand: npm install
    startCommand: npm start
    healthCheckPath: /api/health
    envVars:
      - key: NODE_ENV
        value: production
      - key: OPENAI_API_KEY
        sync: false
      - key: OPENAI_MODEL
        value: gpt-4.1-mini

