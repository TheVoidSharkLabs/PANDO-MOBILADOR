# 📱 Pando Mobilador

Cliente scrcpy customizado, construído com **Tauri**. Interface própria
(sem a UI feia do scrcpy padrão), tema dark + amarelo totalmente
personalizável, painel de atalhos, overlay de FPS e controle completo do
aparelho Android direto da janela do app.

## ✅ Recursos implementados

- Barra de título personalizada (sem moldura do sistema operacional)
- Bordas arredondadas + transparência real da janela (`transparent: true`)
- Botões de minimizar / maximizar / fechar customizados
- Overlay de FPS / bitrate sobre o vídeo
- Painel lateral de atalhos (voltar, home, abas, power, girar, print, toggles)
- Modal de configurações: bitrate, FPS, resolução, codec, modo de
  teclado/mouse (default/UHID/AOA/desativado), modo OTG, argumentos extras
- Temas prontos (Dark Amarelo, Dark Roxo, Gamer Red, Cyan, Claro) +
  personalização livre de cor de destaque, cor do painel e transparência
- Botões de voltar / home / abas / volume / power / girar tela
- Configuração inicial equivalente a
  `scrcpy --keyboard=uhid --mouse=uhid --video-bit-rate=50M --max-fps=120`
  (tudo isso exposto como sliders/toggles amigáveis — o usuário comum nunca
  vê linha de comando, só a barrinha de bitrate/fps e os toggles)
- Ícones 100% SVG inline (sprite próprio em `index.html`, sem libs externas)

## 🧠 Arquitetura (o "jeito certo", sem gambiarra)

```
Tauri (Rust)
   │
   ├─▶ executa o ADB          (src-tauri/src/adb.rs)
   │        │
   │        └─▶ localiza o dispositivo, envia o scrcpy-server via adb push
   │
   ├─▶ inicia o scrcpy         (src-tauri/src/scrcpy.rs)
   │        │
   │        └─▶ o próprio binário oficial `scrcpy` faz: iniciar o
   │            scrcpy-server no Android → abrir conexão TCP → receber e
   │            decodificar o vídeo H.264/H.265/AV1
   │
   └─▶ embute a janela nativa do scrcpy dentro da janela do Tauri
            (src-tauri/src/embed.rs, via SetParent/MoveWindow no Windows)
            e a redimensiona automaticamente sempre que o layout muda
            (sidebar abre/fecha, resize da janela etc.)
```

Ou seja: em vez de reescrever o protocolo do scrcpy do zero (o que é
exatamente o que costuma ficar "bugado"), o Pando Mobilador **usa o
scrcpy oficial como motor de captura/decodificação** — que é maduro e
rápido — e só assume o controle visual dele, encaixando a janela real
do scrcpy dentro do nosso frame customizado. Os botões de navegação
(voltar/home/abas/volume/power/girar) e o screenshot são enviados via
`adb shell input keyevent` / `adb exec-out screencap`, então funcionam
mesmo com o vídeo embutido.

> ⚠️ O embutimento via `SetParent` está implementado para **Windows**
> (a plataforma mais comum para esse tipo de ferramenta). Em Linux/macOS
> o scrcpy funciona normalmente, só que abre em uma janela separada —
> veja os comentários em `src-tauri/src/embed.rs` para saber onde entrar
> com XReparentWindow (X11) ou `addChildWindow` (Cocoa) se quiser estender.

## 📂 Estrutura do projeto

```
pando-mobilador/
├── src/                      → front-end (HTML/CSS/JS puro)
│   ├── index.html
│   ├── index.css
│   └── index.js
├── src-tauri/                → back-end Rust
│   ├── src/
│   │   ├── main.rs           → comandos Tauri + estado da app
│   │   ├── adb.rs            → integração com o ADB
│   │   ├── scrcpy.rs         → monta argumentos e sobe o processo scrcpy
│   │   ├── embed.rs          → embute a janela nativa do scrcpy (Windows)
│   │   └── models.rs         → structs compartilhadas (config, device...)
│   ├── icons/                → ícones do app (placeholder gerado)
│   ├── Cargo.toml
│   └── tauri.conf.json
├── assets/                   → 👉 COLOQUE AQUI os binários do scrcpy
│   └── README.md
├── package.json
└── README.md
```

## ▶️ Como rodar

Pré-requisitos: [Rust](https://rustup.rs), [Node.js](https://nodejs.org)
e as [dependências do Tauri](https://tauri.app/start/prerequisites/) para
o seu sistema operacional.

```bash
npm install

# 1) baixe o scrcpy oficial (https://github.com/Genymobile/scrcpy/releases)
#    e coloque os arquivos dentro da pasta assets/ (veja assets/README.md)

# 2) rode em modo desenvolvimento
npm run dev

# 3) gere o instalador final
npm run build
```

Os ícones em `src-tauri/icons/` são um placeholder simples gerado
automaticamente — rode `npx tauri icon caminho/para/sua-logo.png` para
substituí-los pela identidade visual final do Pando Mobilador.

## 🎨 Personalização de tema

Tudo fica salvo em `localStorage` (chaves `pando_theme` e
`pando_config`) e é aplicado via CSS variables em tempo real — a cor de
destaque (`--accent`) e a cor do painel (`--panel`) podem ser trocadas
livremente no menu **Configurações → Aparência**, além dos 5 temas
prontos (Dark Amarelo é o padrão).

## 🔧 Configurações de inicialização

O comando final enviado ao scrcpy é sempre montado dinamicamente e pode
ser conferido em **Configurações → Avançado → "Comando gerado"** — por
padrão equivale a:

```
scrcpy --video-bit-rate=50M --max-fps=120 --max-size=1280 \
  --video-codec=h264 --keyboard=uhid --mouse=uhid --stay-awake \
  --print-fps --window-borderless --window-title=PandoMobiladorScrcpy
```

O modo `--otg` fica disponível como um botão dedicado no painel lateral
("Modo OTG"), sem precisar espelhar a tela.
