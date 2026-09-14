# Google Translate TTS em páginas estáticas

## Problema

O endpoint usado pelo Google Translate TTS é:

```text
https://translate.google.com/translate_tts?ie=UTF-8&client=tw-ob&tl=da&q=...
```

Quando o áudio é solicitado a partir de uma página publicada em um domínio como
`github.io`, o navegador normalmente envia o cabeçalho `Referer` da página. O
endpoint pode responder com `404 text/html` para essa requisição, em vez de
retornar o áudio `audio/mpeg`.

Como a resposta não é um áudio válido, navegadores baseados em Chromium podem
mostrar:

```text
(failed) net::ERR_BLOCKED_BY_ORB
```

O erro não significa necessariamente que o endereço do áudio está incorreto.
Neste caso, é uma combinação da validação do Google com o bloqueio de
`Opaque Response Blocking` do navegador.

## Implementação recomendada

### 1. Definir `no-referrer` no HTML

Adicione esta tag no `<head>` de todas as páginas que podem executar o TTS:

```html
<meta name="referrer" content="no-referrer">
```

Em sites estáticos, essa é a proteção mais importante, porque também cobre
scripts inline e páginas que criam o áudio diretamente.

### 2. Definir a política antes do `src`

Ao criar o elemento de áudio, defina `referrerPolicy` antes de atribuir a URL:

```javascript
function googleTtsUrl(text, language = "da") {
  const params = new URLSearchParams({
    ie: "UTF-8",
    client: "tw-ob",
    tl: language,
    q: text,
  });
  return `https://translate.google.com/translate_tts?${params}`;
}

function playGoogleTts(text) {
  const audio = new Audio();
  audio.referrerPolicy = "no-referrer";
  audio.src = googleTtsUrl(text);
  audio.play().catch(() => {
    // Exibir uma mensagem e oferecer o TTS do navegador como alternativa.
  });
}
```

Evite depender somente de:

```javascript
new Audio(url);
audio.referrerPolicy = "no-referrer";
```

Alguns navegadores ou versões podem considerar a política tarde demais quando a
URL já foi informada no construtor. Criar o áudio vazio e configurar a política
antes do `src` é mais seguro.

### 3. Tratar falhas e oferecer fallback

O endpoint `translate_tts` não é uma API oficial documentada para uso em
aplicações. Ele pode mudar, limitar requisições ou deixar de responder. O TTS
deve tratar `error` e rejeições de `play()`:

```javascript
audio.onerror = () => {
  showMessage("Google TTS não respondeu. Tente o TTS do navegador.");
};

audio.play().catch(() => {
  showMessage("A reprodução do Google TTS foi bloqueada.");
});
```

Para o fallback local:

```javascript
const utterance = new SpeechSynthesisUtterance(text);
utterance.lang = "da-DK";
speechSynthesis.speak(utterance);
```

## Exemplo completo com frases longas

O endpoint pode falhar ou truncar textos muito longos. Divida o conteúdo em
partes antes de reproduzir:

```javascript
function splitText(text, maxLength = 180) {
  const parts = [];
  while (text.length > maxLength) {
    let breakAt = text.lastIndexOf(" ", maxLength);
    if (breakAt < 40) breakAt = maxLength;
    parts.push(text.slice(0, breakAt));
    text = text.slice(breakAt).trim();
  }
  if (text) parts.push(text);
  return parts;
}

async function speakGoogle(text) {
  for (const part of splitText(text)) {
    const audio = new Audio();
    audio.referrerPolicy = "no-referrer";
    audio.src = googleTtsUrl(part);
    await new Promise((resolve, reject) => {
      audio.onended = resolve;
      audio.onerror = reject;
      audio.play().catch(reject);
    });
  }
}
```

## Checklist para novos projetos

- Adicionar `meta name="referrer" content="no-referrer"` em todas as páginas
  que usam Google TTS.
- Criar o `Audio` sem URL.
- Definir `audio.referrerPolicy = "no-referrer"` antes de definir `audio.src`.
- Codificar o texto com `URLSearchParams` ou `encodeURIComponent`.
- Dividir textos longos em partes curtas.
- Tratar `onerror` e a rejeição de `audio.play()`.
- Implementar fallback com `speechSynthesis`.
- Testar o site publicado em `github.io`, não apenas localmente ou no
  repositório do GitHub.
- Verificar no DevTools que a resposta do Google possui:
  `Content-Type: audio/mpeg` e status `200`.

## Diagnóstico rápido

Teste o endpoint com e sem `Referer`:

```powershell
$url = "https://translate.google.com/translate_tts?ie=UTF-8&client=tw-ob&tl=da&q=Teste"

curl.exe -sS -D - -o NUL `
  -H "Referer: https://seu-usuario.github.io/seu-projeto/pagina.html" `
  $url

curl.exe -sS -D - -o NUL `
  -H "Referer:" `
  $url
```

O caso problemático normalmente retorna `404` com `Content-Type: text/html`
quando há referenciador e `200` com `Content-Type: audio/mpeg` sem
referenciador.

## Observação sobre produção

`translate_tts` é um endpoint interno e não possui garantia de estabilidade.
Para um produto que dependa de disponibilidade, privacidade, volume elevado ou
suporte oficial, prefira uma API de síntese de voz documentada, como Google
Cloud Text-to-Speech, Azure Speech ou Amazon Polly.
