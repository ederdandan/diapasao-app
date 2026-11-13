# SPEC — Diapasão & Afinador Cromático (PWA)

PWA estático para GitHub Pages. Sem build, sem dependências. **Todos os caminhos relativos (`./`)**.

## Arquivos (raiz)
| Arquivo | Papel |
|---|---|
| `index.html` | UI única (HTML+CSS+JS embutidos), mobile-first, visual escuro "equipamento de estúdio" |
| `manifest.json` | `display: standalone`, `start_url: "./"`, `scope: "./"`, cores tema, ícones SVG + PNG |
| `sw.js` | Service Worker Cache-First, cache versionado `diapason-vN` |
| `icon.svg` | Ícone vetorial (diapasão) — favicon e manifest |
| `icon-192.png`, `icon-512.png` | Ícones PNG (Android/Chrome; o 512 também como `maskable`) |
| `apple-touch-icon.png` | 180×180 — iOS não aceita SVG na tela inicial |

## Interface (visual clean)
- Tela principal: medidor, nota, status, botão do afinador e cordas do instrumento atual (toque toca/para o tom) + botão A4.
- Todas as opções (calibração A4, instrumento, timbre, oitava, dicas) ficam no menu ⚙ (bottom sheet com `<dialog>`).
- 4 temas minimalistas (menu ⚙, salvo em `localStorage`), cada um com cores (tokens CSS em `[data-theme]`) e medidor próprios:
  `arco` (escuro, ponteiro semicircular), `regua` (claro, régua linear), `anel` (preto OLED, ponto num anel com a nota no centro), `leds` (azul, 21 LEDs de 5 cents).
- Tela fixa sem scroll (só o menu ⚙ rola internamente) e sem zoom (viewport + `touch-action` + bloqueio de gesture/ctrl+roda); medidores dimensionados por `dvh` para caber em telas baixas.
- Viola: cada par é uma coluna (nota principal + corda de oitava abaixo).

## Áudio
- `getUserMedia` com `echoCancellation`, `noiseSuppression`, `autoGainControl` = `false`.
- `AudioContext` criado/retomado só após gesto do usuário (exigência iOS).
- Analyser conectado ao destino via ganho 0 (garante processamento no Safari).

## Detecção de pitch
- **YIN** (diferença cumulativa normalizada, limiar 0,15) + interpolação parabólica.
- `fftSize = 4096` (necessário p/ E2 = 82 Hz); faixa ~55–1200 Hz.
- Gate de volume (RMS mínimo); mediana das últimas 5 leituras; detecção ~30×/s.
- Nota cromática mais próxima, relativa ao A4 calibrado. Mostra nota+oitava, solfejo, Hz e cents.

## Calibração A4
- 415–466 Hz (padrão 440), slider, `-1`/`+1`, atalhos `432` e `440`. Persistido em `localStorage`.
- **Todas as frequências (afinador e gerador) derivam do A4**: `f = A4 · 2^((midi−69)/12)`.

## Medidor
- Gauge semicircular −50…+50 cents, ponteiro suavizado (lerp por frame), zona verde central.
- Estados, com **histerese** (entra em ±3, sai em ±4) para não piscar:
  - **AFINADO** (verde) |c| ≤ 3
  - **BAIXO — SUBIR AFINAÇÃO ♭** (âmbar) c < −3
  - **ALTO — BAIXAR AFINAÇÃO ♯** (vermelho/coral) c > +3

## Gerador de tom
- `OscillatorNode` com fade-in/out (`setTargetAtTime`) — sem estalos; troca de nota com glide curto.
- Timbre **Puro** (senoide) ou **Rico** (`PeriodicWave` com harmônicos — audível em alto-falante de celular, que não reproduz bem < ~150 Hz).
- Opção **Oitava acima**.
- **Detecção pausada enquanto o tom toca** (sem cancelamento de eco o mic captaria o alto-falante).
- Predefinições (MIDI):
  - A4 (69)
  - Violão: E2 A2 D3 G3 B3 E4
  - Viola **Cebolão em Mi** (5ª→1ª): B2+B3, E3+E4, G#3+G#4, B3, E4 (pares 5–3 em oitava, 2–1 em uníssono)
  - Viola **Cebolão em Ré**: A2+A3, D3+D4, F#3+F#4, A3, D4

## Extras
- Screen Wake Lock enquanto o afinador está ligado (re-solicita ao voltar à aba).
- Aviso: no iPhone, a chave de silencioso corta o áudio do gerador.

## Service Worker
- Pré-cache: `./`, `./index.html`, `./manifest.json`, `./icon.svg`, PNGs.
- `install` → `skipWaiting`; `activate` → apaga caches ≠ atual + `clients.claim`.
- Fetch: só GET mesma origem, cache-first, fallback rede (e guarda), navegação offline → `./index.html`.
- **A cada deploy com mudança, incrementar `CACHE` em `sw.js`** (senão usuários ficam na versão antiga).
- Essa é a fonte única da versão: o menu ⚙ lê `./sw.js` e exibe o `vN`.
- Registro: `navigator.serviceWorker.register('./sw.js')` (sem `/` inicial) + `update()`; no `controllerchange` a página recarrega uma vez (app instalado aplica a versão nova ao abrir).
- Pré-cache com `cache: 'reload'` (ignora o cache HTTP de 10 min do GitHub Pages, senão grava index.html velho).

## Deploy (GitHub Pages)
1. Criar repositório no GitHub e enviar os arquivos para a raiz da branch `main` (`git push` ou "Add file → Upload files").
2. Settings → Pages → Source: *Deploy from a branch* → `main` / `/ (root)` → Save.
3. Acessar `https://<usuario>.github.io/<repositorio>/` (HTTPS — necessário para microfone e SW).
4. No celular: "Adicionar à tela inicial" / "Instalar app".

## Metrônomo (aba "Metrônomo")
- 30–250 BPM: −/+, slider, atalhos 60/80/100/120/140, **Tap tempo** (média dos últimos toques), nome do andamento (Largo…Prestissimo).
- Compassos 2/4, 3/4, 4/4, 6/8 (6/8 com acento secundário no 4º); subdivisão semínima, colcheia, tercina, semicolcheia.
- Agendamento por lookahead (`setTimeout` 25 ms + relógio do `AudioContext`, janela 120 ms) — tempo preciso, sem drift; cliques curtos com envelope (acento 1600 Hz, tempo 1000 Hz, subdivisão 750 Hz).
- Bolinhas de tempo sincronizadas ao áudio; Wake Lock enquanto toca. Aba, BPM, compasso e subdivisão persistidos.
