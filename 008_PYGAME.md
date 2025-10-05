

# Pygame Basics — updated Oct 5, 2025

Beginner‑friendly guide to build a simple 2D game loop with **Pygame**. Covers install, window creation, the main loop, timing and delta time, input handling, drawing, images and sound, sprites and collisions, basic scenes, and packaging. Use with `002_SETUP.md` for environment setup.

---

**Assumptions and Conventions**
- macOS + zsh, Python 3.12+ or 3.13.  
- Run inside an activated virtual environment.  
- Pygame uses SDL2 under the hood. On macOS, window focus and permissions can affect input and audio.

---

## Table of Contents
- [0) Install and verify](#0-install-and-verify)
- [1) Your first window](#1-your-first-window)
- [2) The game loop and timing](#2-the-game-loop-and-timing)
- [3) Input handling (keyboard, mouse)](#3-input-handling-keyboard-mouse)
- [4) Drawing primitives and colors](#4-drawing-primitives-and-colors)
- [5) Images, sprites, and collision](#5-images-sprites-and-collision)
- [6) Text rendering](#6-text-rendering)
- [7) Sound effects and music](#7-sound-effects-and-music)
- [8) Scene/state organization](#8-scenestate-organization)
- [9) Simple physics with delta time](#9-simple-physics-with-delta-time)
- [10) Project structure](#10-project-structure)
- [11) Packaging and running](#11-packaging-and-running)
- [12) Troubleshooting](#12-troubleshooting)
- [13) Recap](#13-recap)

---

## 0) Install and verify
**What**: Install `pygame` and confirm it runs.
```sh
pip install pygame
python - <<'PY'
import pygame; print('pygame', pygame.__version__)
PY
```
If install fails on macOS, `brew install sdl2 sdl2_image sdl2_mixer sdl2_ttf` can help, but wheels usually include SDL2.

---

## 1) Your first window
`hello_pygame.py`:
```python
import pygame

pygame.init()
screen = pygame.display.set_mode((800, 600))
pygame.display.set_caption("Pygame Window")

running = True
while running:
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False

    screen.fill((30, 30, 30))
    pygame.display.flip()

pygame.quit()
```
Run:
```sh
python hello_pygame.py
```
Close the window or press `Ctrl+C` in the terminal to exit if needed.

---

## 2) The game loop and timing
Use `Clock` to regulate FPS and compute **delta time** (seconds since last frame) for smooth motion.
```python
import pygame

pygame.init()
screen = pygame.display.set_mode((800, 600))
clock = pygame.time.Clock()

running = True
x = 100
speed = 200  # pixels per second

while running:
    dt = clock.tick(60) / 1000.0   # limit to ~60 FPS and get seconds

    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False

    # update
    x += speed * dt
    if x > 800: x = -50

    # draw
    screen.fill((0,0,0))
    pygame.draw.rect(screen, (0,200,255), (x, 280, 50, 40))
    pygame.display.flip()

pygame.quit()
```
Notes:
- `tick(60)` caps the loop near 60 FPS and returns milliseconds elapsed.
- Multiply velocities by `dt` so motion is frame‑rate independent.

---

## 3) Input handling (keyboard, mouse)
Two ways to read input:
- **Event queue** for discrete presses, quits, text input.  
- **State polling** for held‑down keys.
```python
keys = pygame.key.get_pressed()
if keys[pygame.K_LEFT]:
    x -= speed * dt

for event in pygame.event.get():
    if event.type == pygame.KEYDOWN and event.key == pygame.K_SPACE:
        print("space pressed once")
    if event.type == pygame.MOUSEBUTTONDOWN and event.button == 1:
        print("clicked at", event.pos)
```
Common keys: `K_w, K_a, K_s, K_d`, arrows, `K_ESCAPE`.

---

## 4) Drawing primitives and colors
```python
screen.fill((30, 30, 30))
pygame.draw.line(screen, (255,255,255), (10,10), (200,10), 2)
pygame.draw.rect(screen, (0,128,255), (50,50,120,80), border_radius=8)
pygame.draw.circle(screen, (255,0,0), (400,300), 40)
pygame.draw.polygon(screen, (0,255,100), [(600,500),(700,550),(650,450)])
```
Define colors as RGB tuples 0–255.

---

## 5) Images, sprites, and collision
### Load and draw images
```python
img = pygame.image.load("assets/player.png").convert_alpha()
screen.blit(img, (x, y))
```
### Sprite classes and groups
```python
class Player(pygame.sprite.Sprite):
    def __init__(self, pos):
        super().__init__()
        self.image = pygame.Surface((40,40), pygame.SRCALPHA)
        pygame.draw.rect(self.image, (50,200,50), (0,0,40,40), border_radius=6)
        self.rect = self.image.get_rect(center=pos)
        self.vx, self.vy = 0.0, 0.0

    def update(self, dt):
        keys = pygame.key.get_pressed()
        speed = 240
        self.vx = (keys[pygame.K_RIGHT] - keys[pygame.K_LEFT]) * speed
        self.vy = (keys[pygame.K_DOWN]  - keys[pygame.K_UP])   * speed
        self.rect.x += int(self.vx * dt)
        self.rect.y += int(self.vy * dt)

player = Player((400,300))
all_sprites = pygame.sprite.Group(player)

# in loop
all_sprites.update(dt)
all_sprites.draw(screen)
```
### Collision
```python
enemy = pygame.sprite.Sprite()
enemy.image = pygame.Surface((40,40)); enemy.image.fill((200,60,60))
enemy.rect = enemy.image.get_rect(center=(600,300))

hit = pygame.sprite.collide_rect(player, enemy)
if hit:
    pygame.draw.rect(screen, (255,255,0), enemy.rect, 3)
```
Other options: `spritecollide`, `collide_mask` for per‑pixel masks, `collide_circle`.

---

## 6) Text rendering
```python
pygame.font.init()
font = pygame.font.SysFont(None, 24)          # default font
text_surf = font.render("Score: 10", True, (255,255,255))
screen.blit(text_surf, (10,10))
```
For TTF fonts:
```python
font = pygame.font.Font("assets/Roboto-Regular.ttf", 24)
```

---

## 7) Sound effects and music
```python
pygame.mixer.init()
laser = pygame.mixer.Sound("assets/laser.wav")
laser.play()

pygame.mixer.music.load("assets/music.ogg")
pygame.mixer.music.play(-1)   # loop
```
Volume and channels can be adjusted via `set_volume()` and `pygame.mixer.Channel`.

---

## 8) Scene/state organization
Keep logic modular. A simple **State** interface keeps menus, game, and pause screens separate.
```python
class State:
    def handle_event(self, e): ...
    def update(self, dt): ...
    def draw(self, screen): ...

class Menu(State):
    def __init__(self): self.time = 0
    def handle_event(self, e):
        if e.type == pygame.KEYDOWN: return 'GAME'
    def update(self, dt): self.time += dt
    def draw(self, s): s.fill((20,20,40))

class Game(State):
    def handle_event(self, e):
        if e.type == pygame.KEYDOWN and e.key == pygame.K_ESCAPE: return 'MENU'
    def update(self, dt): pass
    def draw(self, s): s.fill((0,0,0))

state = Menu()
while running:
    for e in pygame.event.get():
        nxt = state.handle_event(e)
        if nxt == 'GAME': state = Game()
        if nxt == 'MENU': state = Menu()
    state.update(dt)
    state.draw(screen)
    pygame.display.flip()
```

---

## 9) Simple physics with delta time
```python
# constant acceleration example (gravity)
y = 100.0
vy = 0.0
g = 980.0  # px/s^2

vy += g * dt
y  += vy * dt
if y > 560:
    y = 560; vy = -vy * 0.5  # bounce with damping
```
Use `dt` for integration. For more stable physics, consider fixed‑timestep updates in a sub‑loop.

---

## 10) Project structure
```
pygame-demo/
├── .venv/
├── assets/
│   ├── player.png
│   ├── laser.wav
│   └── Roboto-Regular.ttf
├── src/
│   ├── main.py
│   ├── states.py
│   └── sprites.py
└── requirements.txt
```
`src/main.py` skeleton:
```python
import pygame
from states import Game

def run():
    pygame.init()
    screen = pygame.display.set_mode((800,600))
    clock = pygame.time.Clock()
    running = True
    state = Game()
    while running:
        dt = clock.tick(60) / 1000.0
        for e in pygame.event.get():
            if e.type == pygame.QUIT: running = False
            state.handle_event(e)
        state.update(dt)
        state.draw(screen)
        pygame.display.flip()
    pygame.quit()

if __name__ == "__main__":
    run()
```

---

## 11) Packaging and running
For a self‑contained app:
```sh
pip install pyinstaller
pyinstaller --onefile --windowed src/main.py --name pygame-demo
```
Ship the `assets/` folder alongside the binary. On macOS, `--windowed` hides the terminal.

---

## 12) Troubleshooting
- **Black window or no redraw**: you forgot `pygame.display.flip()` or drawing after fill. Draw every frame.
- **High CPU**: missing `clock.tick(...)`. Always cap FPS.
- **Input lag on macOS**: ensure the game window is focused; disable macOS keyboard shortcuts that capture keys.
- **Audio errors**: initialize mixer after `pygame.init()` and verify file formats (`.wav`, `.ogg`).
- **Image not found**: use absolute paths or `pathlib` relative to `__file__`.
- **Stutter**: keep work per frame small; load assets once; avoid I/O in the loop.

---

## 13) Recap
```plaintext
Install pygame → open a window → main loop with Clock → handle input → draw → load images/sounds → use sprites & collisions → organize states → package
```

**Next**: Add animations (sprite sheets), tile maps (Tiled + pytmx), camera scrolling, and save/load with JSON. Keep updates tied to `dt` and avoid blocking the loop.