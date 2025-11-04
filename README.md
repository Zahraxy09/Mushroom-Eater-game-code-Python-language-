 import sys
import math
import pygame

# ------------------ Config -------------------
WIDTH, HEIGHT = 960, 540
FPS = 60
TILE = 36
GRAVITY = 0.6
FRICTION = 0.85
MAX_X_SPEED = 6
JUMP_VELOCITY = -12
CAMERA_LERP = 0.1

# Color
WHITE = (240, 240, 240)
BLACK = (15, 15, 20)
SKY = (40, 45, 75)
GROUND = (70, 80, 95)
GREEN = (80, 200, 120)
RED = (220, 80, 100)
YELLOW = (245, 210, 100)
BLUE = (100, 160, 240)
ORANGE = (255, 150, 70)
# ----------------------------- Helpers ---------------------
def rect_from_grid(x, y):
    return pygame.Rect(x * TILE, y * TILE, TILE, TILE)


def sign(x):
    return (x > 0) - (x < 0)
def rect_from_grid(x, y):
    return pygame.Rect(x * TILE, y * TILE, TILE, TILE)


def sign(x):
    return (x > 0) - (x < 0)



# ----------------------------- Entities ---------------------
class Tile(pygame.sprite.Sprite):
    def __init__(self, rect):
        super().__init__()
        self.rect = rect

    def draw(self, surf, camera_x):
        r = self.rect.move(-camera_x, 0)
        pygame.draw.rect(surf, GROUND, r)


class Mushroom(pygame.sprite.Sprite):
    def __init__(self, x, y):
        super().__init__()
        self.rect = pygame.Rect(x * TILE + 8, y * TILE + 8, TILE - 16, TILE - 16)
        self.pulse = 0

    def update(self):
        self.pulse = (self.pulse + 2) % 360

    def draw(self, surf, camera_x):
        r = self.rect.move(-camera_x, 0)
        # Base
        pygame.draw.ellipse(surf, ORANGE, r)
        # Shine
        inner = r.inflate(-r.w * 0.5, -r.h * 0.5)
        inner.y -= 3
        pygame.draw.ellipse(surf, YELLOW, inner, 2)
        # Halo
        s = 4 + 2 * math.sin(math.radians(self.pulse))
        pygame.draw.ellipse(surf, (255, 255, 255, 40), r.inflate(s, s), 1)


class Enemy(pygame.sprite.Sprite):
    def __init__(self, x, y, tiles):
        super().__init__()
        self.rect = pygame.Rect(x * TILE + 2, y * TILE + 8, TILE - 4, TILE - 8)
        self.vel = pygame.Vector2(2, 0)
        self.tiles = tiles
        self.on_ground = False
        self.dir = 1

    def update(self):
        # Simple patrol: change direction at edges or when hitting a wall
        self.vel.y += GRAVITY
        if self.vel.y > 12:
            self.vel.y = 12

        self.rect.x += int(self.vel.x)
        hit = collide_solid(self.rect, self.tiles)
        if hit:
            self.rect.x -= int(self.vel.x)
            self.dir *= -1
            self.vel.x = 2 * self.dir

        self.rect.y += int(self.vel.y)
        hit = collide_solid(self.rect, self.tiles)
        if hit:
            if self.vel.y > 0:
                self.on_ground = True
            self.vel.y = 0
            # Resolve minimal
            while collide_solid(self.rect, self.tiles):
                self.rect.y -= 1
        else:
            self.on_ground = False

        # Edge detection: if near an edge, turn around
        front_x = self.rect.centerx + self.dir * (self.rect.w // 2 + 2)
        foot_y = self.rect.bottom + 2
        tile_below = pygame.Rect(front_x // TILE * TILE, foot_y // TILE * TILE, TILE, TILE)
        if not any(t.colliderect(tile_below) for t in [t.rect for t in self.tiles]):
            self.dir *= -1
            self.vel.x = 2 * self.dir

    def draw(self, surf, camera_x):
        r = self.rect.move(-camera_x, 0)
        pygame.draw.rect(surf, RED, r, border_radius=6)
        eye = pygame.Rect(0, 0, 6, 6)
        eye.center = (r.centerx + 6 * self.dir, r.centery - 4)
        pygame.draw.circle(surf, WHITE, eye.center, 3)


class Player(pygame.sprite.Sprite):
    def __init__(self, spawn, tiles):
        super().__init__()
        self.rect = pygame.Rect(spawn[0] * TILE + 4, spawn[1] * TILE + 4, TILE - 8, TILE - 8)
        self.vel = pygame.Vector2(0, 0)
        self.on_ground = False
        self.tiles = tiles
        self.facing = 1
        self.coyote_time = 0.0
        self.jump_buffer = 0.0
        self.score = 0
        self.alive = True

    def handle_input(self, keys):
        if not self.alive:
            return
        ax = 0
        if keys[pygame.K_LEFT] or keys[pygame.K_a]:
            ax -= 1
        if keys[pygame.K_RIGHT] or keys[pygame.K_d]:
            ax += 1

        self.vel.x += ax * 0.8
        if ax == 0:
            self.vel.x *= FRICTION
            if abs(self.vel.x) < 0.1:
                self.vel.x = 0

        self.vel.x = max(-MAX_X_SPEED, min(MAX_X_SPEED, self.vel.x))
        if ax != 0:
            self.facing = sign(ax) or self.facing

    def try_jump(self):
        # Consume jump if buffered and coyote time allows
        if self.jump_buffer > 0 and self.coyote_time > 0:
            self.vel.y = JUMP_VELOCITY
            self.on_ground = False
            self.coyote_time = 0
            self.jump_buffer = 0

    def update(self, mushrooms, enemies, dt):
        if not self.alive:
            return

        # Timers
        if self.on_ground:
            self.coyote_time = 0.12
        else:
            self.coyote_time = max(0, self.coyote_time - dt)
        self.jump_buffer = max(0, self.jump_buffer - dt)

        # Gravity
        self.vel.y += GRAVITY
        if self.vel.y > 16:
            self.vel.y = 16

        # Horizontal movement & collision
        self.rect.x += int(self.vel.x)
        for tile in collide_solid_list(self.rect, self.tiles):
            if self.vel.x > 0:
                self.rect.right = tile.left
            elif self.vel.x < 0:
                self.rect.left = tile.right
            self.vel.x = 0

        # Vertical movement & collision
        self.rect.y += int(self.vel.y)
        self.on_ground = False
        for tile in collide_solid_list(self.rect, self.tiles):
            if self.vel.y > 0:
                self.rect.bottom = tile.top
                self.on_ground = True
            elif self.vel.y < 0:
                self.rect.top = tile.bottom
            self.vel.y = 0

        # Collect mushrooms
        for m in list(mushrooms):
            if self.rect.colliderect(m.rect):
                mushrooms.remove(m)
                self.score += 1

        # Enemy collisions
        for e in enemies:
            if self.rect.colliderect(e.rect):
                # Stomp check
                if self.vel.y > 3 and self.rect.bottom - e.rect.top < 12:
                    # Bounce
                    self.vel.y = JUMP_VELOCITY * 0.7
                    # "Defeat" enemy by pushing it down
                    e.rect.y += 9999
                else:
                    self.die()

    def die(self):
        self.alive = False

    def buffer_jump(self):
        self.jump_buffer = 0.15

    def draw(self, surf, camera_x):
        r = self.rect.move(-camera_x, 0)
        body = pygame.Rect(r.x, r.y, r.w, r.h)
        pygame.draw.rect(surf, BLUE, body, border_radius=6)
        # Face
        eye_offset = 6 if self.facing >= 0 else -6
        pygame.draw.circle(surf, WHITE, (body.centerx + eye_offset, body.y + 10), 3)
        # Feet
        pygame.draw.rect(surf, BLACK, (body.x + 4, body.bottom - 6, body.w - 8, 6), border_radius=3)


# ----------------------------- Collision -----------------------------
def collide_solid(rect, tiles):
    for t in tiles:
        if rect.colliderect(t.rect):
            return t.rect
    return None


def collide_solid_list(rect, tiles):
    return [t.rect for t in tiles if rect.colliderect(t.rect)]


# ----------------------------- World Build -----------------------------
def build_world(level):
    tiles = []
    mushrooms = pygame.sprite.Group()
    enemies = pygame.sprite.Group()
    spawn = (1, 1)

    for y, row in enumerate(level):
        for x, ch in enumerate(row):
            if ch == "#":
                tiles.append(Tile(rect_from_grid(x, y)))
            elif ch == "M":
                mushrooms.add(Mushroom(x, y))
            elif ch == "E":
                enemies.add(Enemy(x, y, tiles))
            elif ch == "P":
                spawn = (x, y)
    return tiles, mushrooms, enemies, spawn


# ----------------------------- Camera -----------------------------
class Camera:
    def __init__(self):
        self.x = 0

    def update(self, target_rect):
        target = target_rect.centerx - WIDTH // 2
        self.x += (target - self.x) * CAMERA_LERP
        self.x = max(0, min(self.x, LEVEL_WIDTH - WIDTH))


# ----------------------------- UI -----------------------------
def draw_hud(surf, score, total, alive):
    font = pygame.font.SysFont("consolas", 24)
    text = f"Score: {score}/{total}"
    label = font.render(text, True, WHITE)
    surf.blit(label, (14, 10))
    if not alive:
        big = pygame.font.SysFont("consolas", 40, bold=True)
        msg = big.render("You Died - Press R to Restart", True, WHITE)
        surf.blit(msg, (WIDTH // 2 - msg.get_width() // 2, HEIGHT // 2 - 60))

    if total == score and alive:
        big = pygame.font.SysFont("consolas", 40, bold=True)
        msg = big.render("You Win! Press R to Play Again", True, WHITE)
        surf.blit(msg, (WIDTH // 2 - msg.get_width() // 2, HEIGHT // 2 - 60))


# ----------------------------- Main -----------------------------
def reset_game():
    tiles, mushrooms, enemies, spawn = build_world(LEVEL)
    player = Player(spawn, tiles)
    camera = Camera()
    return tiles, mushrooms, enemies, player, camera


if __name__ == "__main__":
    pygame.init()
    screen = pygame.display.set_mode((WIDTH, HEIGHT))
    clock = pygame.time.Clock()
    pygame.display.set_caption("Mushroom Eater")

    global LEVEL_WIDTH
    LEVEL_WIDTH = len(LEVEL[0]) * TILE

    tiles, mushrooms, enemies, player, camera = reset_game()

    running = True
    while running:
        dt = clock.tick(FPS) / 1000.0

        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False

            if event.type == pygame.KEYDOWN:
                if event.key in (pygame.K_SPACE, pygame.K_w, pygame.K_UP):
                    player.buffer_jump()
                if event.key == pygame.K_r:
                    tiles, mushrooms, enemies, player, camera = reset_game()
                if event.key == pygame.K_ESCAPE:
                    running = False

        keys = pygame.key.get_pressed()
        player.handle_input(keys)
        player.try_jump()

        # Update world
        for m in mushrooms:
            m.update()
        for e in enemies:
            e.update()
        player.update(mushrooms, enemies, dt)
        camera.update(player.rect)

        # Render
        screen.fill(SKY)

        # Parallax background (simple stripes)
        for i in range(0, WIDTH, 80):
            pygame.draw.line(screen, (60, 65, 100), (i, 0), (i, HEIGHT), 1)

        # Ground tiles
        for t in tiles:
            t.draw(screen, camera.x)

        # Collectibles and enemies
        for m in mushrooms:
            m.draw(screen, camera.x)
        for e in enemies:
                for e in enemies:

            e.draw(screen, camera.x)

        # Player
        player.draw(screen, camera.x)

        # HUD
        draw_hud(screen, player.score, len(LEVEL) * 0 + len(mushrooms) + player.score, player.alive)

        pygame.display.flip()

    pygame.quit()
    sys.exit()
