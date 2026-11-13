# SPEC — Diapasão, Afinador Cromático & Metrônomo (PWA)

PWA estático para GitHub Pages. Sem build, sem dependências. **Todos os caminhos relativos (`./`)**.
Publicado em `https://ederdandan.github.io/diapasao-app/`. Versão atual: **v9** (ver `CACHE` em `sw.js`).

## Regras para quem for alterar
1. **Toda mudança publicada → incrementar `CACHE` (`diapason-vN`) em `sw.js`.** É a fonte única da versão (o menu ⚙ lê `./sw.js` e exibe `vN`) e é o que faz o app instalado atualizar.
2. Manter tudo em `index.html` (HTML+CSS+JS embutidos). Sem frameworks/CDNs (precisa funcionar offline).
3. Antes de commitar: checar sintaxe dos `<script>` e, se mexer em layout, conferir os 4 temas e as 2 abas em tela baixa (~667 px), sem scroll.
4. IDs são únicos no documento (já houve bug com `#sub` duplicado).

## Arquivos
| Arquivo | Papel |
|---|---|
| `index.html` | App inteiro (UI, áudio, temas, metrônomo) |
| `manifest.json` | `display: standalone`, `start_url`/`scope` `./`, cores, ícones SVG + PNG |
| `sw.js` | Service Worker cache-first, cache versionado |
| `icon.svg` | Ícone vetorial — favicon e manifest |
| `icon-192.png`, `icon-512.png` | PNG Android/Chrome (512 também `maskable`) |
| `apple-touch-icon.png` | 180×180 — iOS não aceita SVG na tela inicial |
| `.github/workflows/pages.yml` | Deploy automático no GitHub Pages a cada push na `main` |
| `SPEC.md` | Este documento |

## Interface
- Topo: abas **Afinador | Metrônomo** (aba persistida) + botão ⚙.
- Menu ⚙ (bottom sheet `<dialog>`, único lugar que rola): Tema, Calibração A4, Instrumento, Timbre, Oitava, dicas, **Instalar app**, **Versão**.
- Tela fixa: sem scroll (`body` fixo, `overflow:hidden`, `overscroll-behavior:none`), sem zoom (viewport `maximum-scale=1,user-scalable=no`, `touch-action:manipulation`, bloqueio de `gesturestart` no iOS e Ctrl+roda), sem seleção de texto / menu de toque longo.
- Medidores dimensionados por `dvh`; `@media (max-height:720px)` reduz fontes/espaços.
- **Temas** (tokens CSS em `html[data-theme]`, aplicado no `<head>` antes do render; `theme-color` acompanha):
  | Tema | Visual | Medidor |
  |---|---|---|
  | `arco` (padrão) | escuro, verde | ponteiro semicircular ±50 cents |
  | `regua` | claro (papel) | régua linear com marcador |
  | `anel` | preto OLED | ponto num anel, nota no centro |
  | `leds` | azul-grafite | 21 LEDs de 5 cents |
- Cores semânticas em todos: verde = afinado, âmbar = baixo, vermelho/coral = alto.

## Afinador
- `getUserMedia` com `echoCancellation`, `noiseSuppression`, `autoGainControl` = `false`.
- `AudioContext` criado/retomado só após gesto do usuário (iOS). Analyser ligado ao destino via ganho 0 (Safari).
- **YIN** (limiar 0,15) + interpolação parabólica; `fftSize 4096` (E2 = 82 Hz); faixa ~55–1200 Hz.
- Gate de volume (RMS > 0,008), mediana das últimas 5 leituras, ~30 detecções/s; ponteiro suavizado por frame.
- Mostra nota+oitava, solfejo (Dó…Si), Hz e cents. Sem sinal por 1,5 s → limpa a leitura.
- Estados com **histerese** (entra em ±3, sai em ±4 cents): **AFINADO** · **♭ BAIXO — SUBIR AFINAÇÃO** · **♯ ALTO — BAIXAR AFINAÇÃO**.
- Screen Wake Lock enquanto ligado.

## Calibração A4
- 415–466 Hz (padrão 440), slider, −1/+1, atalhos 432 e 440.
- **Tudo deriva do A4**: `f = A4 · 2^((midi−69)/12)`.

## Diapasão (gerador de tom)
- `OscillatorNode` com fade-in/out (`setTargetAtTime`), glide curto ao trocar nota. Toque na nota toca; de novo para.
- Timbre **Puro** (senoide) ou **Rico** (`PeriodicWave` com harmônicos — audível em alto-falante de celular).
- **Oitava acima** opcional.
- **Detecção do afinador pausada enquanto o tom toca** (mic captaria o alto-falante).
- Notas (MIDI), cada coluna = uma corda/par (corda de oitava embaixo):
  - Violão: E2 A2 D3 G3 B3 E4
  - Viola **Cebolão Mi** (5ª→1ª): B2+B3, E3+E4, G#3+G#4, B3, E4
  - Viola **Cebolão Ré**: A2+A3, D3+D4, F#3+F#4, A3, D4
  - Botão Lá A4.
- Aviso: no iPhone a chave de silencioso corta o áudio.

## Metrônomo
- 30–250 BPM: −/+, slider, atalhos 60/80/100/120/140, **Tap** (média dos últimos toques, reinicia após 2 s), nome do andamento (Largo…Prestissimo).
- Compassos 2/4, 3/4, 4/4, 6/8 (6/8 com acento secundário no 4º); subdivisão semínima, colcheia, tercina, semicolcheia.
- Agendamento por lookahead (`setTimeout` 25 ms + relógio do `AudioContext`, janela 120 ms) — sem drift.
- Cliques com envelope: acento 1600 Hz, acento secundário 1300 Hz, tempo 1000 Hz, subdivisão 750 Hz.
- Bolinhas de tempo sincronizadas ao áudio; Wake Lock enquanto toca.
- Limitação: com app em segundo plano/tela bloqueada o navegador pode irregularizar o `setTimeout`.

## Persistência (`localStorage`, sempre em try/catch)
`a4`, `inst`, `rich`, `oct`, `theme`, `view`, `bpm`, `sig`, `sub`.

## Instalação (PWA)
- Chrome/Edge/Android: `beforeinstallprompt` → botão "Instalar app" no menu ⚙ (some após instalar).
- iOS/iPadOS: instrução Compartilhar → Adicionar à Tela de Início.
- Já instalado (`display-mode: standalone`) ou sem suporte: seção oculta.

## Service Worker
- Pré-cache: `./`, `./index.html`, `./manifest.json`, `./icon.svg`, PNGs — com `cache: 'reload'` (ignora o cache HTTP de 10 min do GitHub Pages).
- `install` → `skipWaiting`; `activate` → apaga caches antigos + `clients.claim`.
- Fetch: só GET mesma origem; cache-first, senão rede (e guarda); navegação offline → `./index.html`.
- Registro `register('./sw.js')` (sem `/` inicial) + `update()`; no `controllerchange` a página recarrega uma vez → app instalado aplica a versão nova ao abrir.

## Deploy
- Push na `main` → workflow `pages.yml` publica (Settings → Pages → Source: **GitHub Actions**).
- HTTPS do GitHub Pages é obrigatório para microfone e Service Worker.
