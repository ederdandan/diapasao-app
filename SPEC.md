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
- Registro: `navigator.serviceWorker.register('./sw.js')` (sem `/` inicial).

## Deploy (GitHub Pages)
1. Criar repositório no GitHub e enviar os arquivos para a raiz da branch `main` (`git push` ou "Add file → Upload files").
2. Settings → Pages → Source: *Deploy from a branch* → `main` / `/ (root)` → Save.
3. Acessar `https://<usuario>.github.io/<repositorio>/` (HTTPS — necessário para microfone e SW).
4. No celular: "Adicionar à tela inicial" / "Instalar app".
