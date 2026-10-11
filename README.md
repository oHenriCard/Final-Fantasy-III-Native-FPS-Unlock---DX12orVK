# FF3 Native FPS Unlock — 60+ FPS and DirectX 12 for FINAL FANTASY III (3D Remake, Steam)

Version 1.0 · Supported executable: `FF3_Win32.exe` (current Steam build, SHA-256 `feb914c5…3c9e`)

English first · Português (PT-BR) abaixo

---

## English

### What the mod does

The FF3 3D engine (inherited from the Nintendo DS) is a *tick* engine: every logic tick (30 per
second) also draws the screen, and the loop is capped with `SDL_Delay(33 ms - elapsed)`. There is no
delta-time anywhere, so simply uncapping the loop makes the game run twice as fast.

This mod applies the classic technique used by modern engines, **fixed timestep + render interpolation**:

- Logic keeps running at exactly 30 ticks/s (speed, battles, timers and scripts are 100% original).
- Every OpenGL call the game makes is intercepted through the EXE import table and recorded each tick,
  including copies of the vertex arrays.
- When the game presents a tick, the mod re-renders the recorded scene one or more times with vertex
  positions and matrices (camera/lights) interpolated between the previous and the current tick,
  delivering a new, distinct frame on every monitor refresh (or at the fixed rate set in `FrameRate`).
- The original 33 ms limiter is neutralised and replaced by precise pacing.

This is **not image-based frame generation** (no AI, no ghosting, no reconstruction artifacts): every
frame is a real render of the game's geometry. HUD, text and 2D sprites are interpolated too and stay sharp.

### DirectX 12 mode

With `Renderer=DirectX12` the game's fixed-function OpenGL runs on the **Mesa d3d12** driver (OpenGL on
Direct3D 12, developed by Microsoft and Collabora). All rendering becomes D3D12 command lists presented
through a DXGI flip-model swapchain; overlays such as RTSS identify the game as `D3D12`. The log confirms it:

```
[gl] vendor=Microsoft Corporation renderer=D3D12 (NVIDIA GeForce RTX 5060) version=4.6 (Compatibility Profile) Mesa 26.2.4
```

How it works: `SDL2.dll` loads the OpenGL library only when it creates the window; the mod intercepts that
`LoadLibrary`, loads `mesa\opengl32.dll`, redirects the EXE's 34 `gl*` imports to Mesa and the five GDI32
pixel-format / `SwapBuffers` functions SDL uses to Mesa's `wgl*` equivalents. The system `opengl32.dll` stays
loaded but is never used. If the `mesa` folder is missing or fails to load, the mod falls back to native
OpenGL and says so in the log.

Requirements: Windows 10/11 and a GPU with Direct3D 12 (Shader Model 6). The `mesa` folder holds
`opengl32.dll`, `libgallium_wgl.dll` (driver) and `dxil.dll` (shader signing), x86 build of mesa-dist-win
26.2.4 (MIT licence, see `mesa\LICENSE-mesa.txt`).

`Renderer=Vulkan` (Mesa Zink on your GPU's Vulkan driver) is also available but only lightly tested.

### Skipping the boot logos and the opening video

- `SkipSplash=1` skips the Square Enix / Matrix logos. The logo scene's state machine is patched so its
  first state jumps straight to its last one (the transition that starts the movie); the screen stays
  black for the short initial fade (about one second).
- `SkipMovie=1` skips the opening FMV (`opening.mkv`). As soon as the movie starts, the mod does what the
  confirm button does: it stops and closes the movie, and the game proceeds to the title screen. Only the
  boot playback is skipped.

### ReShade and other overlays

- **OpenGL mode:** install ReShade for the **OpenGL** API (it becomes `opengl32.dll` in the game folder). Nothing
  else to do; the mod hooks ReShade's exports and ReShade applies its effects on every presented frame.
- **DirectX 12 mode:** also install ReShade for the **OpenGL** API, not for Direct3D 12. The mod detects the
  wrapper, keeps it in front, points its `[PROXY] ProxyLibrary` setting in `ReShade.ini` at `mesa\opengl32.dll`
  automatically and selects the d3d12 driver, so ReShade runs its OpenGL runtime on top of Mesa and reports
  "Running on D3D12 (your GPU)" in its log.
- A ReShade installed for the **Direct3D 12** API (`d3d12.dll` / `dxgi.dll` in the game folder) is not compatible
  with the translation layer in this 32-bit process (the game used to crash inside `dxgi.dll`). The mod now
  bypasses such wrappers, logs a warning and the game runs without effects; reinstall ReShade choosing OpenGL.
- `OverlayCompat=1` (default) enables all of the above, plus it makes Mesa create its D3D12 device through
  `D3D12CreateDevice` so overlays can see it. Set it to 0 to leave third-party DLLs and `ReShade.ini` untouched.

### Install

1. Copy `version.dll`, `FF3NativeFPSUnlock.ini` and the `mesa` folder into the game folder
   (`...\Steam\steamapps\common\Final Fantasy III\`, next to `FF3_Win32.exe`), or run `Install.cmd`.
   The `mesa` folder is only needed for `Renderer=DirectX12` / `Vulkan`.
2. `Install.cmd` finds the game folder (argument, Steam registry, every Steam library, or it asks), asks
   which API to use (0 OpenGL, 1 DirectX 12, 2 Vulkan), whether to skip the logos and whether to skip
   the opening video, and writes those answers into `FF3NativeFPSUnlock.ini`, keeping every other setting.
   To install into another copy: `Install.cmd "D:\Games\Final Fantasy III"`.
3. Launch the game from Steam as usual. No game file is modified; Steam Cloud and saves are untouched.
4. `FF3NativeFPSUnlock.log` in the game folder shows the renderer, pacing mode and statistics
   (`ticks/s` should read 30.00 and `frames/s` your target).

Ultimate ASI Loader users: use `asi_variant\FF3NativeFPSUnlock.asi` instead of `version.dll`.

**Uninstall:** delete `version.dll` (and, optionally, `FF3NativeFPSUnlock.ini`, `FF3NativeFPSUnlock.log` and the `mesa`
folder), or run `Uninstall.cmd`.

### Settings (`FF3NativeFPSUnlock.ini`)

| Key | Default | Meaning |
|---|---|---|
| `Enabled` | 1 | Turns the mod on/off |
| `Renderer` | OpenGL | `OpenGL` (native driver), `DirectX12` (Mesa d3d12), `Vulkan` (Mesa Zink, experimental) |
| `FrameRate` | 0 | Presented frames per second: `0` = monitor refresh rate, or `60`, `120`, `180`, `240` |
| `SkipSplash` | 0 | `1` = skip the Square Enix / Matrix logos at boot |
| `SkipMovie` | 0 | `1` = skip the opening video at boot |
| `OverlayCompat` | 1 | ReShade/overlay compatibility: auto-configures ReShade's proxy library in DirectX 12 mode, bypasses D3D12 wrappers, uses `D3D12CreateDevice` |
| `Interpolate` | 1 | `0` = precise pacing only, no interpolation (duplicate frames) |
| `TickRate` | 30 | Logic ticks per second. 30 = original speed. 60 = game runs 2x faster (speed hack) |
| `Mode` | auto | `vsync` locks to the refresh rate (multiples of 30 Hz); `timed` uses a timer; `auto` picks. A `FrameRate` different from the refresh rate forces `timed` |
| `VSync` | -1 | -1 keeps what the game sets (on), 0 off, 1 on |
| `MaxDelta2D` | 96 | Sprites/UI that move further than this between two ticks are not interpolated |
| `MaxDelta3D` | 0 | Same for 3D models (0 = unlimited) |
| `MatrixLerp` | 1 | Interpolates matrices (camera/lights) |
| `Log` | 1 | Writes `FF3NativeFPSUnlock.log` |
| `DebugDump` | 0 | Debug: every N s writes 3 `.ppm` frames (previous / midpoint / current) |

About `FrameRate`: with `0` the mod presents one frame per monitor refresh (60 Hz → 60 FPS, 180 Hz →
180 FPS). With a fixed value the mod presents exactly that rate regardless of the monitor; if the monitor
is slower than the value, VSync limits it to the monitor's rate.

### Known limitations

- The opening video (`opening.mkv`) plays at 30 FPS when not skipped: it is decoded video drawn through
  a different path.
- Latency: to show in-between frames the mod displays the game state up to one tick late
  (≈17 ms at 60 Hz, ≈27 ms at 180 Hz). Imperceptible in a turn-based JRPG, but it exists.
- On camera cuts or objects that "teleport", a single in-between frame may show the averaged position.
  Scene-cut detection (fewer than 50% of draws matching → no interpolation) and the displacement limits
  minimise this.
- Only the supported Steam build of `FF3_Win32.exe` works. If the game is updated, the mod checks the
  frame-limiter bytes and, on a mismatch, stays idle and says so in the log (the game runs normally at 30 FPS).
- DirectX 12 mode depends on Mesa; GPUs without D3D12 fall back to native OpenGL.

### How it works inside (for modders)

- `version.dll` is loaded automatically because `SDL2.dll` imports `VERSION.dll` and Windows searches the
  game folder first. The mod forwards the 17 real functions to the system `version.dll`.
- In `DllMain` the mod validates the EXE (`add ecx, 0x21` bytes in the limiter, RVA `0x10305D`) and
  replaces `SDL_GL_SwapWindow`, `SDL_Delay` and `SDL_GetTicks` in the EXE import table; the 28
  `OPENGL32.dll` functions are replaced immediately (OpenGL) or as soon as Mesa is loaded (DirectX 12).
- `SDL_Delay` is ignored only when called by the limiter (return address RVA `0x103071`).
- In `SDL_GL_SwapWindow` the mod matches the current tick's draw calls with the previous tick's (signature:
  call site, mode, vertex count, texture, format), interpolates vertices and matrices and re-renders.
- `SkipSplash` rewrites entry 0 of the logo scene's jump table (RVA `0xA963C`) to its state-4 handler
  (RVA `0xA95BD`); `SkipMovie` calls the game's stop (RVA `0x10AB50`) and close (RVA `0x109C10`) movie
  functions from the main loop once the movie flag (RVA `0x2022AE`) is set.
- Useful points: frame function RVA `0x96C0` (runs N ticks per frame, N = 1 or 2x/3x in fast-forward),
  limiter RVA `0x103046`–`0x103071`, 3D model draw RVA `0x12D57`, current scene index RVA `0x180A18`
  (1 = logos, 10 = movie).

Source code in `source/`. Builds with a single command using Zig (clang) 0.15:

```
zig cc -target x86-windows-gnu -O2 -s -shared -o version.dll ff3nativefpsunlock.c version.def -lkernel32 -luser32
```

### Credits

FF3 Native FPS Unlock by Henrique Passos. Mesa3D (MIT) by the Mesa project; Windows builds by pal1000
(mesa-dist-win); d3d12 driver by Microsoft and Collabora. `dxil.dll` from the DirectX Shader Compiler.
Built with Zig/clang.

---

## Português (PT-BR)

### O que o mod faz

O motor do FF3 3D (herdado do Nintendo DS) é um motor de *ticks*: a cada tick de lógica (30 por
segundo) ele também desenha a tela, e trava o loop com `SDL_Delay(33 ms - tempo gasto)`. Não existe
delta-time em lugar nenhum, então simplesmente destravar o loop faz o jogo rodar 2x mais rápido.

Este mod aplica a técnica clássica de motores modernos, **fixed timestep + interpolação de render**:

- A lógica continua rodando a exatos 30 ticks/s (velocidade, batalhas, timers e scripts 100% originais).
- Todas as chamadas OpenGL do jogo são interceptadas pela tabela de importação do EXE e gravadas a cada
  tick, incluindo cópias dos arrays de vértices.
- Quando o jogo apresenta um tick, o mod re-renderiza a cena gravada uma ou mais vezes com as posições
  dos vértices e as matrizes (câmera/luzes) interpoladas entre o tick anterior e o atual, entregando um
  frame novo e distinto a cada atualização do monitor (ou na taxa fixa escolhida em `FrameRate`).
- O limitador original de 33 ms é neutralizado e substituído por pacing preciso.

Isto **não é frame generation por imagem** (sem IA, sem ghosting, sem artefatos de reconstrução):
cada frame é um render real da geometria do jogo. HUD, texto e sprites 2D também são interpolados.

### Modo DirectX 12

Com `Renderer=DirectX12`, o OpenGL fixo do jogo é executado pelo driver **Mesa d3d12** (OpenGL sobre
Direct3D 12, desenvolvido pela Microsoft e Collabora). Toda a renderização vira command lists D3D12 e a
apresentação é feita por swapchain DXGI (flip model). Overlays como RTSS identificam o jogo como `D3D12`.
O log confirma com uma linha como:

```
[gl] vendor=Microsoft Corporation renderer=D3D12 (NVIDIA GeForce RTX 5060) version=4.6 (Compatibility Profile) Mesa 26.2.4
```

Como funciona: `SDL2.dll` carrega a biblioteca OpenGL só na criação da janela; o mod intercepta esse
`LoadLibrary`, carrega `mesa\opengl32.dll`, redireciona as 34 importações `gl*` do EXE para o Mesa e as
5 funções de pixel format / `SwapBuffers` que o SDL pega da GDI32 para as equivalentes `wgl*` do Mesa.
O `opengl32.dll` do sistema fica carregado mas nunca é usado. Se a pasta `mesa` não existir ou falhar,
o mod volta automaticamente para o OpenGL nativo e avisa no log.

Requisitos: Windows 10/11 com GPU que suporte Direct3D 12 (Shader Model 6). A pasta `mesa` contém
`opengl32.dll`, `libgallium_wgl.dll` (driver) e `dxil.dll` (assinatura de shaders), build x86 do
mesa-dist-win 26.2.4 (licença MIT, ver `mesa\LICENSE-mesa.txt`).

`Renderer=Vulkan` (Mesa Zink sobre o driver Vulkan da GPU) também está disponível, com poucos testes.

### Pular as logos de abertura e o vídeo inicial

- `SkipSplash=1` pula as logos Square Enix / Matrix. A máquina de estados da cena de logos é patcheada
  para que o primeiro estado salte direto para o último (a transição que inicia o vídeo); a tela fica
  preta durante o fade inicial curto (cerca de um segundo).
- `SkipMovie=1` pula o vídeo de abertura (`opening.mkv`). Assim que o vídeo começa, o mod faz o mesmo que
  o botão de confirmar: para e fecha o vídeo, e o jogo segue para a tela de título. Só a reprodução da
  abertura é pulada.

### ReShade e outros overlays

- **Modo OpenGL:** instale o ReShade para a API **OpenGL** (ele vira `opengl32.dll` na pasta do jogo). Não há
  mais nada a fazer; o mod intercepta as exportações do ReShade e ele aplica os efeitos em cada frame apresentado.
- **Modo DirectX 12:** instale o ReShade também para a API **OpenGL**, não para Direct3D 12. O mod detecta o
  wrapper, deixa-o na frente, aponta a opção `[PROXY] ProxyLibrary` do `ReShade.ini` para `mesa\opengl32.dll`
  automaticamente e seleciona o driver d3d12; assim o ReShade roda seu runtime OpenGL sobre o Mesa e registra
  "Running on D3D12 (sua GPU)" no log dele.
- Um ReShade instalado para a API **Direct3D 12** (`d3d12.dll` / `dxgi.dll` na pasta do jogo) não é compatível
  com a camada de tradução neste processo de 32 bits (o jogo caía dentro do `dxgi.dll`). O mod agora ignora esses
  wrappers, registra um aviso e o jogo roda sem efeitos; reinstale o ReShade escolhendo OpenGL.
- `OverlayCompat=1` (padrão) liga tudo isso e ainda faz o Mesa criar o dispositivo D3D12 via `D3D12CreateDevice`,
  para que overlays o enxerguem. Coloque 0 para não tocar em DLLs de terceiros nem no `ReShade.ini`.

### Instalação

1. Copie `version.dll`, `FF3NativeFPSUnlock.ini` e a pasta `mesa` para a pasta do jogo
   (`...\Steam\steamapps\common\Final Fantasy III\`, ao lado de `FF3_Win32.exe`), ou rode `Install.cmd`.
   A pasta `mesa` só é necessária para `Renderer=DirectX12` / `Vulkan`.
2. O `Install.cmd` localiza a pasta do jogo (argumento, registro da Steam, todas as bibliotecas Steam, ou
   pergunta), pergunta qual API usar (0 OpenGL, 1 DirectX 12, 2 Vulkan), se quer pular as logos e se quer
   pular o vídeo inicial, e grava as respostas em `FF3NativeFPSUnlock.ini` preservando as outras opções.
   Para instalar em outra cópia: `Install.cmd "D:\Jogos\Final Fantasy III"`.
3. Abra o jogo normalmente pela Steam. Nenhum arquivo do jogo é modificado; Steam Cloud e saves não
   são tocados.
4. `FF3NativeFPSUnlock.log` na pasta do jogo mostra renderizador, modo de pacing e estatísticas
   (`ticks/s` deve ficar em 30.00 e `frames/s` na taxa escolhida).

Usuários do **Ultimate ASI Loader**: use `asi_variant\FF3NativeFPSUnlock.asi` em vez do `version.dll`.

**Desinstalar:** apague `version.dll` (e, se quiser, `FF3NativeFPSUnlock.ini`, `FF3NativeFPSUnlock.log` e a pasta
`mesa`), ou rode `Uninstall.cmd`.

### Configuração (`FF3NativeFPSUnlock.ini`)

| Chave | Padrão | Função |
|---|---|---|
| `Enabled` | 1 | Liga/desliga o mod |
| `Renderer` | OpenGL | `OpenGL` (driver nativo), `DirectX12` (Mesa d3d12), `Vulkan` (Mesa Zink, experimental) |
| `FrameRate` | 0 | Frames apresentados por segundo: `0` = taxa do monitor, ou `60`, `120`, `180`, `240` |
| `SkipSplash` | 0 | `1` = pula as logos Square Enix / Matrix na abertura |
| `SkipMovie` | 0 | `1` = pula o vídeo de abertura |
| `OverlayCompat` | 1 | Compatibilidade com ReShade/overlays: configura o proxy do ReShade no modo DirectX 12, ignora wrappers D3D12, usa `D3D12CreateDevice` |
| `Interpolate` | 1 | `0` = só pacing preciso, sem interpolação (frames duplicados) |
| `TickRate` | 30 | Ticks de lógica por segundo. 30 = velocidade original. 60 = jogo 2x mais rápido (speed hack) |
| `Mode` | auto | `vsync` trava no refresh (múltiplos de 30 Hz); `timed` usa timer; `auto` escolhe. Um `FrameRate` diferente do refresh força `timed` |
| `VSync` | -1 | -1 mantém o que o jogo define (ligado), 0 desliga, 1 liga |
| `MaxDelta2D` | 96 | Sprites/UI que se deslocam mais que isso entre dois ticks não são interpolados |
| `MaxDelta3D` | 0 | Idem para modelos 3D (0 = sem limite) |
| `MatrixLerp` | 1 | Interpola matrizes (câmera/luzes) |
| `Log` | 1 | Grava `FF3NativeFPSUnlock.log` |
| `DebugDump` | 0 | Debug: a cada N s grava 3 frames `.ppm` (anterior / meio / atual) |

Sobre `FrameRate`: com `0`, o mod entrega um frame a cada refresh do monitor (60 Hz → 60 FPS, 180 Hz →
180 FPS). Com um valor fixo, o mod apresenta exatamente essa taxa, independente do monitor; se o monitor
for mais lento que o valor, o VSync limita à taxa do monitor.

### Limitações conhecidas

- O vídeo de abertura (`opening.mkv`), quando não é pulado, roda a 30 FPS: é um vídeo decodificado por
  outro caminho.
- Latência: para mostrar frames intermediários o mod exibe o estado do jogo com um atraso de até um tick
  (≈17 ms a 60 Hz, ≈27 ms a 180 Hz). Em um JRPG por turnos é imperceptível, mas existe.
- Em cortes de câmera ou objetos que "teleportam", um único frame intermediário pode mostrar a posição
  média. Há detecção de corte (menos de 50% dos draws casando → sem interpolação) e limites de
  deslocamento para minimizar isso.
- Só funciona no build Steam suportado do `FF3_Win32.exe`. Se o jogo for atualizado, o mod verifica os
  bytes do limitador de frames e, se não baterem, fica inativo e avisa no log (jogo roda normal, a 30 FPS).
- O modo DirectX 12 depende do Mesa; em GPUs sem D3D12 o mod cai para OpenGL nativo.

### Como funciona por dentro (para quem quer mexer)

- `version.dll` é carregado automaticamente porque `SDL2.dll` importa `VERSION.dll` e o Windows procura
  primeiro na pasta do jogo. O mod repassa as 17 funções reais do `version.dll` do sistema.
- No `DllMain`, o mod valida o EXE (bytes de `add ecx, 0x21` no limitador, RVA `0x10305D`) e substitui
  na IAT do EXE `SDL_GL_SwapWindow`, `SDL_Delay` e `SDL_GetTicks`; as 28 funções `OPENGL32.dll` são
  substituídas na hora (OpenGL) ou assim que o Mesa é carregado (DirectX 12).
- `SDL_Delay` é ignorado apenas quando chamado pelo limitador (endereço de retorno RVA `0x103071`).
- Em `SDL_GL_SwapWindow` o mod casa os draw calls do tick atual com os do anterior (assinatura: call
  site, modo, contagem de vértices, textura, formato), interpola vértices e matrizes e re-renderiza.
- `SkipSplash` reescreve a entrada 0 da jump table da cena de logos (RVA `0xA963C`) para o handler do
  estado 4 (RVA `0xA95BD`); `SkipMovie` chama as funções do jogo de parar (RVA `0x10AB50`) e fechar
  (RVA `0x109C10`) o vídeo a partir do loop principal assim que a flag do vídeo (RVA `0x2022AE`) liga.
- Pontos úteis: função de frame RVA `0x96C0` (roda N ticks por frame, N = 1 ou 2x/3x em avanço rápido),
  limitador RVA `0x103046`–`0x103071`, draw 3D RVA `0x12D57`, índice da cena atual RVA `0x180A18`
  (1 = logos, 10 = vídeo).

Código-fonte em `source/`. Compila com um único comando usando Zig (clang) 0.15:

```
zig cc -target x86-windows-gnu -O2 -s -shared -o version.dll ff3nativefpsunlock.c version.def -lkernel32 -luser32
```

### Créditos

FF3 Native FPS Unlock por Henrique Passos. Mesa3D (MIT) pelo projeto Mesa; builds Windows por pal1000
(mesa-dist-win); driver d3d12 por Microsoft e Collabora. `dxil.dll` do DirectX Shader Compiler.
Compilado com Zig/clang.
