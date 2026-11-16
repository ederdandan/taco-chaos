# ESPECIFICAÇÃO TÉCNICA (SPEC) — TACO! GATO! CHAOS! (PWA P2P)

**Versão:** 1.0.0  
**Arquitetura:** 100% Client-Side / Serverless P2P (WebRTC)  
**Hospedagem:** GitHub Pages (Estático)  
**Público-alvo:** 2 a 6 jogadores em smartphones/computadores  

---

## 1. Visão Geral e Objetivos

O **TACO! GATO! CHAOS!** é uma adaptação digital party reflex baseada no jogo *Taco Gato Cabra Queijo Pizza*, enriquecida com modificadores de caos sensorial, gestos mobile (swipe para cima, swipe para baixo e toque duplo) e interação social em tempo real.

O projeto opera **100% no navegador**, sem servidor backend próprio, hospedado de forma gratuita no **GitHub Pages**, utilizando recursos modernos de navegador (Cache API, WebRTC, Web Audio API e LocalStorage).

---

## 2. Pilares de Arquitetura

### 2.1. Topologia de Rede (WebRTC em Estrela)
* **Host Autoritativo:** O primeiro jogador a criar a sala atua como o **Host**. Ele executa a máquina de estados oficial da partida, controla o baralho, a ordem das palavras, as penalidades e a IA dos robôs.
* **Clientes:** Conectam-se diretamente ao Host via `RTCDataChannel` (WebRTC).
* **Signaling Serverless:** Utiliza a infraestrutura pública do **PeerJS** (`0.peerjs.com` ou fallback TURN/STUN público) exclusivamente para a troca inicial de SDP/ICE (handshake de ~1 segundo). Após a conexão estabelecida, todo o tráfego é ponto-a-ponto (P2P).
* **Capacidade:** 2 a 6 participantes por sala (jogadores humanos e/ou bots).

### 2.2. Sistema de Contas e Persistência Local
* **Sem Auth Externa:** Sem necessidade de OAuth, Google, e-mail ou servidor.
* **Armazenamento:** `localStorage` do navegador com o seguinte esquema de dados:
  ```json
  {
    "userId": "usr_9f8a3c4b_d1e2",
    "name": "Lucas",
    "avatar": "😎",
    "createdAt": 1700000000000,
    "stats": {
      "matches": 14,
      "wins": 6,
      "slaps": 128
    },
    "activeSession": {
      "roomId": "TACO7",
      "slotIndex": 2
    }
  }
  ```
* **Auto-Login:** Ao abrir o site no GitHub Pages, o perfil é carregado do `localStorage`. Se não existir, apresenta a tela rápida de criação de perfil.

### 2.3. Tolerância a Quedas: Assunção por Robô (Bot Failover)
* **Detecção:** O Host monitora o evento `peer.on('close')` e `peer.on('error')` de cada conexão.
* **Substituição:**
  * O Host marca a vaga correspondente como `isBot: true`.
  * O motor de IA do Host assume os turnos do jogador (descarte de carta após delay de 900–1300ms e batidas na arena com atraso de reação humano de 750–1450ms).
  * A partida segue normalmente para os outros participantes sem interrupções.

### 2.4. Reconexão e Retomada de Controle
* **Persistência de Sessão:** O cliente armazena o `roomId` atual no `localStorage`.
* **Fluxo de Reconexão:**
  1. O cliente reconecta ao Host e envia o pacote `{ type: 'RECONNECT', userId, name }`.
  2. O Host localiza a vaga associada àquele `userId`.
  3. O Host remove a flag `isBot` daquela vaga e vincula o novo canal WebRTC.
  4. O Host envia o snapshot completo do jogo (`type: 'STATE_SYNC'`).
  5. A interface do jogador reidrata seu baralho, placar e o botão de ação.

### 2.5. Sistema de Stickers em Tempo Real
* **Drawer de Stickers:** Menu retrátil com stickers interativos temáticos (ex: 🌮, 🐱, 🐐, 🧀, 🍕, 🦍, 🦄, 🤡, 🔥, 💀, 💥).
* **Broadcast:** Envio de mensagem `{ type: 'STICKER', fromSlot, stickerId }` para o Host, que replica aos demais clientes.
* **Animação:** Bolhas flutuantes com animação de subida (*float up + fade out*) saindo do avatar do jogador emissor.
* **Cooldown:** Limite de 1.5s entre envios para evitar flood e não atrapalhar os reflexos.

### 2.6. Stickers Cômicos Automáticos de Derrota
* **Disparador:** Quando um jogador perde a rodada (é o último a bater ou queima a largada batendo fora de hora).
* **Ação do Host:**
  * Sorteia um meme/sticker de derrota:
    * 🤡 *Palhaço do Monte*
    * 🐢 *Modo Tartaruga*
    * 💀 *F no Chat*
    * 🧀 *Derretido pelo Queijo*
    * 🎒 *Levou a Mochila Cheia*
  * Envia evento `{ type: 'PENALTY', loserSlot, sticker, reason }`.
  * Todos os clientes tocam som de penalidade e exibem uma explosão cômica do sticker sobre o avatar do perdedor.

---

### 3. Protocolo de Mensagens WebRTC (JSON)

| Tipo | Origem | Descrição |
| :--- | :--- | :--- |
| `JOIN_ROOM` | Cliente ➔ Host | Solicita entrada na sala com `userId`, `name`, `avatar`. |
| `ROOM_UPDATE` | Host ➔ Todos | Atualiza lista de jogadores, prontidão e slots ocupados. |
| `START_GAME` | Host ➔ Todos | Inicia a partida e distribui as cartas iniciais. |
| `PLAY_CARD` | Cliente/Bot ➔ Host | Descarte de carta pelo jogador do turno atual. |
| `CARD_REVEALED` | Host ➔ Todos | Notifica carta aberta, modificador de caos e palavra da sequência. |
| `TRIGGER_ACTIVE` | Host ➔ Todos | Notifica que a palavra combinou ou carta especial apareceu. |
| `SLAP` | Cliente ➔ Host | Batida na mesa com timestamp para arbitragem. |
| `SLAP_CONFIRMED` | Host ➔ Todos | Exibe o carimbo da mão do jogador que acabou de bater. |
| `ROUND_RESOLVED` | Host ➔ Todos | Entrega o monte ao perdedor com placar atualizado e sticker de derrota. |
| `STICKER` | Cliente ➔ Todos | Envia reação em tempo real para os outros participantes. |
| `RECONNECT` | Cliente ➔ Host | Solicita reassumir vaga anterior após queda. |
| `STATE_SYNC` | Host ➔ Cliente | Snapshot completo para o cliente reconectado. |

---

### 4. Estrutura de Arquivos da Aplicação

```
taco-chaos/
├── index.html          # Interface PWA, Layout, Estilos, Áudio e Motor do Jogo
├── manifest.json       # Configuração de instalação PWA (Nome, Ícones, Cores)
├── sw.js               # Service Worker com Cache API (suporte offline)
├── SPEC.md             # Esta especificação técnica
```

---

### 5. Fases de Execução

1. **Fase 1:** Criar `SPEC.md`, `manifest.json` e `sw.js` para conformidade PWA offline.
2. **Fase 2:** Implementar sistema de perfil/conta no `localStorage` (modal de boas-vindas / alternância de perfil).
3. **Fase 3:** Integrar biblioteca PeerJS e criar o Lobby P2P (Criar Sala, Entrar com Código/Link, até 6 slots).
4. **Fase 4:** Migrar a máquina de estados para o Host com suporte a múltiplos clientes reais + bots.
5. **Fase 5:** Implementar bot failover automático ao desconectar e reconexão transparente por `userId`.
6. **Fase 6:** Implementar a barra de stickers interativos e as reações automáticas de derrota.

---

### 6. Melhorias de Jogabilidade Implementadas (v1.1)

| # | Recurso | Como funciona |
| :--- | :--- | :--- |
| 1 | **Narração das palavras** | A cada carta, o aparelho fala a palavra da sequência (`speechSynthesis`, voz pt-BR do sistema, sem arquivos de áudio). Liga/desliga em *Perfil → Preferências* (`prefs.voice`). Respeita o botão de som. |
| 1b | **Modo Difícil** | Esconde a palavra escrita e o destaque da sequência (`🔊 ???`): o jogador bate pelo que ouve. Preferência pessoal (`prefs.hideWord`); exige narração ativa e é suspenso automaticamente com o som desligado. |
| 3 | **Carta-armadilha 🦥 PREGUIÇA** | ~8% das cartas. Parece especial, mas **nunca** é gatilho: quem bater leva o monte (meme *"Caiu na armadilha!"*). Bots caem nela com 15% de chance cada. |
| 9 | **QR Code da sala** | Botão **📱 QR** na sala exibe o QR do link de convite (`#sala=CODIGO`) e o código em letras grandes. Usa `qrcodejs` via CDN; offline mostra só o código. |

O perfil no `localStorage` ganhou o campo `prefs: { voice, hideWord }`.

---

### 7. Roadmap de Melhorias Futuras

Esforço: **B** = baixo (horas), **M** = médio (1–2 dias), **A** = alto.

#### 7.1. Núcleo do Jogo
| Ideia | Esforço | Descrição |
| :--- | :---: | :--- |
| **Velocidade progressiva** | B | O intervalo entre cartas dos bots cai conforme o monte cresce (ex: 1,2s → 0,6s) e reseta quando alguém leva o monte. Monte grande = jogo rápido e tenso. |
| **Novas cartas especiais** | M | 🔄 *Inverter* (inverte a ordem dos turnos), 🐙 *Polvo* (tocar em 3 pontos da tela), 🪞 *Espelho* (carta exibe o nome de outra), ⏩ *Pular* (sequência pula uma palavra). |
| **Novos modificadores de caos** | B | Carta que treme ou desliza ao cair, "palavra fantasma" que pisca antes da real, texto espelhado, carta que some após 300ms. |
| **Partidas mais curtas** | B | Opção de 5–7 cartas iniciais ou *morte súbita* após X minutos (vence quem tiver menos cartas). |

#### 7.2. Justiça e Competição
| Ideia | Esforço | Descrição |
| :--- | :---: | :--- |
| **Batida por tempo de reação** | M | Hoje o host bate com latência zero e clientes perdem 30–150ms de rede. Cada aparelho mede o próprio tempo (carta revelada → toque) e o host ordena as batidas por esse tempo. Exibir *"Você bateu em 412ms"*. |
| **Dificuldade dos bots** | B | Fácil / Médio / Difícil (tempo de reação), e bots que erram às vezes (queimam a largada, demoram no gesto). |

#### 7.3. Social / Festa
| Ideia | Esforço | Descrição |
| :--- | :---: | :--- |
| **Configurações da sala** | M | Host escolhe nº de cartas, caos on/off, cartas especiais ativas e velocidade. Desligar o caos também melhora a acessibilidade (o modificador *DALTONISMO* atrapalha daltônicos). Enviado no `GAME_STARTED`. |
| **Modo espectador** | M | Quem chega no meio da partida assiste (em vez de ser recusado com `GAME_IN_PROGRESS`) e entra automaticamente na próxima. |
| **Resumo pós-partida** | B | Destaques: ⚡ mais rápido (reação média), 🎒 quem mais levou monte, 🤡 mais largadas queimadas. |
| **Conquistas locais** | B | Badges no `localStorage` junto de `stats`: "Reflexo de 300ms", "Venceu sem levar monte", "5 batidas falsas numa partida". |
| **Revanche com "Pronto"** | B | Na tela de vitória cada jogador confirma; a revanche começa quando todos estiverem prontos (hoje só o host decide). |

#### 7.4. Experiência
| Ideia | Esforço | Descrição |
| :--- | :---: | :--- |
| **Tutorial jogável** | M | Partida guiada de ~1 minuto ensinando a regra da palavra, especiais, armadilha e gestos. |
| **Migração de host** | A | Se o host cair, o cliente mais antigo assume a sala com o último snapshot (hoje a sala é encerrada). |
