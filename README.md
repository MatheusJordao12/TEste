<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Chat Markdown com Gemini</title>

  <style>
    body { font-family: Inter, system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial; padding: 20px; background: #fafafa; }
    .container { max-width: 800px; margin: 0 auto; }
    h1 { margin-bottom: 12px; }
    #chat { display: flex; flex-direction: column; gap: 12px; margin-bottom: 12px; }
    .msg { max-width: 85%; padding: 10px 12px; border-radius: 12px; box-shadow: 0 1px 3px rgba(0,0,0,0.06); }
    .msg.user { align-self: flex-end; background: #e8f0fe; color: #0b3d91; border-top-right-radius: 4px; }
    .msg.gemini { align-self: flex-start; background: #fff; color: #1b1b1b; border-top-left-radius: 4px; }
    .meta { font-size: 12px; opacity: .7; margin-bottom: 6px; }
    #controls { display:flex; gap:8px; margin-bottom: 12px; }
    input[type="text"] { flex: 1; padding: 8px 10px; border-radius: 8px; border: 1px solid #ddd; }
    button { padding: 8px 12px; border-radius: 8px; border: none; background: #1a73e8; color: white; cursor: pointer; }
    button.secondary { background: #6c757d; }
    /* estilo do HTML interno (conteúdo convertido do markdown) */
    .msg .content p { margin: 0 0 8px 0; }
    .msg .content ul { margin: 0 0 8px 1.2em; }
    .msg .content strong { font-weight: 700; }
    .msg .content em { font-style: italic; }
    pre { background:#111; color:#eee; padding:8px; border-radius:6px; overflow:auto; }
  </style>
</head>
<body>
  <div class="container">
    <h1>Chat Markdown (renderizado)</h1>

    <div id="chat"></div>

    <div id="controls">
      <input id="pergunta" type="text" placeholder="Digite sua pergunta..." />
      <button id="enviar">Enviar</button>
      <button id="limpar" class="secondary">Limpar memória</button>
    </div>

    <small>Observação: a chave exposta no front-end é insegura — use apenas para testes locais.</small>
  </div>

  <!-- marked (Markdown -> HTML) e DOMPurify (sanitização) -->
  <script src="https://cdn.jsdelivr.net/npm/marked/marked.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/dompurify@2.4.0/dist/purify.min.js"></script>

  <script type="module">
    import { GoogleGenerativeAI } from "https://esm.run/@google/generative-ai";

    // === CONFIGURE AQUI (somente para testes locais!) ===
    const GEMINI_API_KEY = "AIzaSyCcZzQGrjyiyt8CNKV6LVSVjMSI5rc0mNo";
    const genAI = new GoogleGenerativeAI(GEMINI_API_KEY);
    // =====================================================

    const chatEl = document.getElementById("chat");
    const input = document.getElementById("pergunta");
    const btnEnviar = document.getElementById("enviar");
    const btnLimpar = document.getElementById("limpar");

    // system prompt: comando que define o estilo/voz do modelo
    const systemPrompt = { role: "system", content: "Fala como se você fosse um soldado indo para a segunda guerra mundial. Responda em estilo direto — sem explicações extras." };

    // mensagens: histórico persistido em localStorage
    let mensagens = JSON.parse(localStorage.getItem("mensagens")) || [];

    // renderiza a lista de mensagens no chat
    function renderChat() {
      chatEl.innerHTML = ""; // limpa
      if (mensagens.length === 0) {
        const info = document.createElement("div");
        info.textContent = "Nenhuma mensagem ainda. Faça uma pergunta.";
        info.style.opacity = ".7";
        chatEl.appendChild(info);
        return;
      }

      for (const m of mensagens) {
        const wrapper = document.createElement("div");
        wrapper.className = "msg " + (m.role === "user" ? "user" : "gemini");

        // meta linha (quem e hora opcional)
        const meta = document.createElement("div");
        meta.className = "meta";
        meta.textContent = m.role === "user" ? "Você" : "Gemini";
        wrapper.appendChild(meta);

        // content: aqui vamos converter Markdown -> HTML e SANITIZAR antes de inserir
        const content = document.createElement("div");
        content.className = "content";

        // se já estivermos guardando conteúdo em markdown, convertemos:
        // marked.parse transforma Markdown em HTML
        const html = marked.parse(m.content || "");
        // DOMPurify.sanitize remove scripts e atributos perigosos
        const safe = DOMPurify.sanitize(html, { USE_PROFILES: { html: true } });

        content.innerHTML = safe;
        wrapper.appendChild(content);

        chatEl.appendChild(wrapper);
      }

      // rolar para baixo
      chatEl.scrollTop = chatEl.scrollHeight;
    }

    renderChat();

    // função que envia o contexto + system prompt ao Gemini
    async function conversar(pergunta) {
      mensagens.push({ role: "user", content: pergunta });
      localStorage.setItem("mensagens", JSON.stringify(mensagens));
      renderChat();

      // montar contexto: systemPrompt + histórico (apenas role:content text)
      const contexto = [systemPrompt, ...mensagens].map(m => `${m.role}: ${m.content}`).join("\n\n");

      // pega o modelo (troque caso precise)
      const model = genAI.getGenerativeModel({ model: "gemini-2.5-flash" });

      // chamada
      const result = await model.generateContent(contexto);
      const resposta = await result.response.text();

      // empilha a resposta (assumimos que o modelo já retornou em Markdown)
      mensagens.push({ role: "model", content: resposta });
      localStorage.setItem("mensagens", JSON.stringify(mensagens));
      renderChat();
    }

    btnEnviar.addEventListener("click", async () => {
      const pergunta = input.value.trim();
      if (!pergunta) return;
      input.value = "";
      btnEnviar.disabled = true;
      btnEnviar.textContent = "Enviando...";
      try {
        await conversar(pergunta);
      } catch (err) {
        mensagens.push({ role: "model", content: `❌ Erro: ${err.message}` });
        localStorage.setItem("mensagens", JSON.stringify(mensagens));
        renderChat();
        console.error(err);
      } finally {
        btnEnviar.disabled = false;
        btnEnviar.textContent = "Enviar";
      }
    });

    btnLimpar.addEventListener("click", () => {
      mensagens = [];
      localStorage.removeItem("mensagens");
      renderChat();
    });

    // Enter para enviar
    input.addEventListener("keydown", (e) => {
      if (e.key === "Enter") {
        btnEnviar.click();
      }
    });
  </script>
</body>
</html>
