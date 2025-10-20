import pygame
import random
import sys

# ------------------ Config ------------------
WIDTH, HEIGHT = 480, 640
FPS = 60

PLAYER_SPEED = 6
BULLET_SPEED = -10
ENEMY_SPEED_MIN = 2
ENEMY_SPEED_MAX = 5
ENEMY_SPAWN_TIME = 800  # ms
POWERUP_SPAWN_TIME = 10000  # ms
POWERUP_DURATION = 6000  # ms

# ------------------ Initialize ------------------
pygame.init()
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Space Shooter")
clock = pygame.time.Clock()
font = pygame.font.SysFont("Arial", 20)

# ------------------ Colors ------------------
WHITE = (255, 255, 255)
BLACK = (0, 0, 0)
RED   = (255, 50, 50)
GREEN = (50, 255, 50)
BLUE  = (50, 120, 255)
YELLOW = (255, 220, 50)
PURPLE = (170, 80, 255)

# ------------------ Sprites ------------------
class Player(pygame.sprite.Sprite):
    def __init__(self):
        super().__init__()
        self.w = 48
        self.h = 36
        self.image = pygame.Surface((self.w, self.h), pygame.SRCALPHA)
        # draw a simple ship
        pygame.draw.polygon(self.image, BLUE, [(0,self.h),(self.w/2,0),(self.w,self.h)])
        pygame.draw.rect(self.image, (20,20,80), (self.w*0.35,self.h*0.5,self.w*0.3,self.h*0.35))
        self.rect = self.image.get_rect(midbottom=(WIDTH//2, HEIGHT-10))
        self.speed = PLAYER_SPEED
        self.lives = 3
        self.last_shot = 0
        self.shoot_delay = 300  # ms
        self.powered = False
        self.power_time = 0

    def update(self):
        keys = pygame.key.get_pressed()
        if keys[pygame.K_LEFT] or keys[pygame.K_a]:
            self.rect.x -= self.speed
        if keys[pygame.K_RIGHT] or keys[pygame.K_d]:
            self.rect.x += self.speed
        # clamp
        if self.rect.left < 0: self.rect.left = 0
        if self.rect.right > WIDTH: self.rect.right = WIDTH

        # powerup timer
        if self.powered and pygame.time.get_ticks() > self.power_time:
            self.powered = False
            self.shoot_delay = 300

    def shoot(self):
        now = pygame.time.get_ticks()
        if now - self.last_shot < self.shoot_delay:
            return
        self.last_shot = now
        if self.powered:
            # triple shot
            bullets.add(Bullet(self.rect.centerx, self.rect.top, -10, -1))
            bullets.add(Bullet(self.rect.centerx, self.rect.top, -10, 0))
            bullets.add(Bullet(self.rect.centerx, self.rect.top, -10, 1))
        else:
            bullets.add(Bullet(self.rect.centerx, self.rect.top, BULLET_SPEED, 0))

class Bullet(pygame.sprite.Sprite):
    def __init__(self, x, y, vy, vx_offset):
        super().__init__()
        self.image = pygame.Surface((6, 12))
        self.image.fill(YELLOW)
        self.rect = self.image.get_rect(center=(x + vx_offset*6, y))
        self.vy = vy
        self.vx = vx_offset * 3

    def update(self):
        self.rect.y += self.vy
        self.rect.x += self.vx
        if self.rect.bottom < 0 or self.rect.top > HEIGHT:
            self.kill()

class Enemy(pygame.sprite.Sprite):
    def __init__(self):
        super().__init__()
        size = random.randint(24, 48)
        self.image = pygame.Surface((size, size), pygame.SRCALPHA)
        pygame.draw.circle(self.image, RED, (size//2, size//2), size//2)
        # add an eye
        pygame.draw.circle(self.image, (20,20,20), (size//2, size//2), max(2, size//6))
        self.rect = self.image.get_rect(midtop=(random.randint(20, WIDTH-20), -size))
        self.vy = random.randint(ENEMY_SPEED_MIN, ENEMY_SPEED_MAX)
        self.vx = random.choice([-1,0,1]) * random.random() * 1.5

    def update(self):
        self.rect.y += self.vy
        self.rect.x += self.vx
        # bounce on sides
        if self.rect.left < 0 or self.rect.right > WIDTH:
            self.vx = -self.vx
        if self.rect.top > HEIGHT:
            self.kill()
            # penalty handled externally if needed

class PowerUp(pygame.sprite.Sprite):
    def __init__(self, kind):
        super().__init__()
        self.kind = kind  # "rapid" or "shield"
        self.image = pygame.Surface((26, 26), pygame.SRCALPHA)
        color = PURPLE if kind=="rapid" else GREEN
        pygame.draw.rect(self.image, color, (0,0,26,26), border_radius=6)
        label = "R" if kind=="rapid" else "S"
        txt = font.render(label, True, BLACK)
        self.image.blit(txt, (6,4))
        self.rect = self.image.get_rect(center=(random.randint(20, WIDTH-20), -20))
        self.vy = 2

    def update(self):
        self.rect.y += self.vy
        if self.rect.top > HEIGHT:
            self.kill()

# ------------------ Groups ------------------
player = Player()
players = pygame.sprite.GroupSingle(player)
bullets = pygame.sprite.Group()
enemies = pygame.sprite.Group()
powerups = pygame.sprite.Group()
all_sprites = pygame.sprite.Group(player)

# ------------------ Timers ------------------
ENEMY_EVENT = pygame.USEREVENT + 1
pygame.time.set_timer(ENEMY_EVENT, ENEMY_SPAWN_TIME)

POWERUP_EVENT = pygame.USEREVENT + 2
pygame.time.set_timer(POWERUP_EVENT, POWERUP_SPAWN_TIME)

# ------------------ Game State ------------------
score = 0
level = 1
running = True
game_over = False

def draw_text(surf, text, size, x, y, color=WHITE):
    f = pygame.font.SysFont("Arial", size)
    txt = f.render(text, True, color)
    rect = txt.get_rect(topleft=(x,y))
    surf.blit(txt, rect)

def spawn_enemy_wave(count):
    for _ in range(count):
        e = Enemy()
        enemies.add(e)
        all_sprites.add(e)

# initial wave
spawn_enemy_wave(4)

# ------------------ Main Loop ------------------
while running:
    clock.tick(FPS)
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False
        if event.type == ENEMY_EVENT and not game_over:
            # spawn enemies gradually
            for _ in range(random.randint(1, 2)):
                e = Enemy()
                enemies.add(e)
                all_sprites.add(e)
        if event.type == POWERUP_EVENT and not game_over:
            kind = random.choice(["rapid", "shield"])
            p = PowerUp(kind)
            powerups.add(p)
            all_sprites.add(p)
        if event.type == pygame.KEYDOWN:
            if event.key == pygame.K_SPACE and not game_over:
                player.shoot()
            if event.key == pygame.K_r and game_over:
                # reset
                enemies.empty(); bullets.empty(); powerups.empty()
                player = Player()
                players = pygame.sprite.GroupSingle(player)
                all_sprites = pygame.sprite.Group(player)
                score = 0; level = 1; game_over = False

    if not game_over:
        # update
        all_sprites.update()
        bullets.update()
        enemies.update()
        powerups.update()

        # bullet hits enemy
        hits = pygame.sprite.groupcollide(enemies, bullets, True, True)
        for hit in hits:
            score += 10
            # small chance to spawn extra enemy or powerup
            if random.random() < 0.1:
                e = Enemy()
                enemies.add(e)
                all_sprites.add(e)

        # enemy hits player
        if not player.powered:
            enemy_hits = pygame.sprite.spritecollide(player, enemies, True, pygame.sprite.collide_rect)
            if enemy_hits:
                player.lives -= 1
                if player.lives <= 0:
                    game_over = True

        else:
            # if powered (shield), collisions don't harm but still remove enemy
            enemy_hits = pygame.sprite.spritecollide(player, enemies, True)
            if enemy_hits:
                score += 5

        # player collects powerup
        power_hits = pygame.sprite.spritecollide(player, powerups, True)
        for pu in power_hits:
            if pu.kind == "rapid":
                player.powered = True
                player.shoot_delay = 100
                player.power_time = pygame.time.get_ticks() + POWERUP_DURATION
            elif pu.kind == "shield":
                player.powered = True
                player.shoot_delay = 200
                player.power_time = pygame.time.get_ticks() + POWERUP_DURATION
                player.lives = min(player.lives + 1, 5)

        # level up by score
        if score // 100 + 1 > level:
            level += 1
            # speed up enemies slightly
            ENEMY_SPAWN_TIME = max(300, ENEMY_SPAWN_TIME - 50)
            pygame.time.set_timer(ENEMY_EVENT, ENEMY_SPAWN_TIME)

    # draw
    screen.fill((8,8,20))  # space background
    # stars
    for i in range(30):
        # lightweight star field: draw few circles randomly (deterministic each frame isn't required)
        pass

    all_sprites.draw(screen)
    bullets.draw(screen)
    enemies.draw(screen)
    powerups.draw(screen)

    # HUD
    draw_text(screen, f"Score: {score}", 20, 10, 10, WHITE)
    draw_text(screen, f"Lives: {player.lives}", 20, 10, 35, WHITE)
    draw_text(screen, f"Level: {level}", 20, WIDTH-110, 10, WHITE)
    if player.powered:
        draw_text(screen, "POWER-UP", 18, WIDTH-110, 35, YELLOW)

    if game_over:
        # overlay
        overlay = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
        overlay.fill((0,0,0,180))
        screen.blit(overlay, (0,0))
        draw_text(screen, "GAME OVER", 48, WIDTH//2 - 120, HEIGHT//2 - 60, RED)
        draw_text(screen, f"Final Score: {score}", 28, WIDTH//2 - 90, HEIGHT//2, WHITE)
        draw_text(screen, "Press R to restart", 24, WIDTH//2 - 110, HEIGHT//2 + 40, WHITE)

    pygame.display.flip()

pygame.quit()
sys.exit()
