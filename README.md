# Car Dodger — mobile-friendly Pygame
# Works in Pydroid 3 (Android). No image assets required.
# Controls:
#  - Tap/hold LEFT / RIGHT on-screen buttons
#  - OR drag horizontally on the lower 40% of the screen
#  - Tap START / RESTART buttons to play

import pygame, random, math, sys

pygame.init()
pygame.mixer.quit()  # avoid Android mixer init issues if any

# --- Dynamic sizing for phones ---
info = pygame.display.Info()
SW, SH = max(480, info.current_w), max(800, info.current_h)  # safety defaults
# full screen for phones; if windowed is preferred, remove FULLSCREEN
screen = pygame.display.set_mode((SW, SH), pygame.FULLSCREEN)
pygame.display.set_caption("Car Dodger")

clock = pygame.time.Clock()
FPS = 60

# --- Colors ---
WHITE = (240, 240, 240)
BLACK = (10, 10, 10)
GRAY  = (40, 40, 40)
GREEN = (46, 204, 113)
RED   = (231, 76, 60)
YELLOW= (241, 196, 15)
BLUE  = (52, 152, 219)

# --- Metrics based on screen size ---
road_margin = int(SW * 0.1)
road_w = SW - road_margin*2
lane_count = 3
lane_w = road_w // lane_count
road_x = road_margin
road_y = 0
road_h = SH

# Player car size
car_w = int(lane_w * 0.55)
car_h = int(SH * 0.10)

# Enemy car size
enemy_w = car_w
enemy_h = car_h

# UI areas
button_h = int(SH * 0.11)
touch_zone_h = int(SH * 0.40)  # bottom area for drag steering
font = pygame.font.SysFont("arial", int(SH * 0.045))

# --- Helpers ---
def draw_text(surface, text, x, y, color=WHITE, center=False, bold=False, big=False):
    f = pygame.font.SysFont("arial", int(SH * (0.07 if big else 0.045)), bold=bold)
    s = f.render(text, True, color)
    r = s.get_rect()
    if center:
        r.center = (x, y)
    else:
        r.topleft = (x, y)
    surface.blit(s, r)

def lane_center(idx):
    return road_x + idx * lane_w + lane_w // 2

def clamp(v, lo, hi):
    return max(lo, min(hi, v))

# --- Classes ---
class Car:
    def __init__(self):
        self.w = car_w
        self.h = car_h
        self.x = lane_center(lane_count // 2) - self.w // 2
        self.y = SH - self.h - int(SH * 0.15)
        self.speed = int(SH * 0.6)  # px/sec vertical baseline (used for feel)
        self.move_speed = int(SW * 0.9)  # px/sec horizontal
        self.color = BLUE
        self.target_x = self.x

    @property
    def rect(self):
        return pygame.Rect(int(self.x), int(self.y), self.w, self.h)

    def update(self, dt, holding_left, holding_right, drag_dx):
        # Buttons movement
        dx = 0
        if holding_left:  dx -= 1
        if holding_right: dx += 1
        self.x += dx * self.move_speed * dt

        # Drag steering (fine control)
        self.x += drag_dx

        # Keep inside road
        left_bound = road_x + int(lane_w * 0.1)
        right_bound = road_x + road_w - self.w - int(lane_w * 0.1)
        self.x = clamp(self.x, left_bound, right_bound)

    def draw(self, surf):
        pygame.draw.rect(surf, self.color, self.rect, border_radius=int(self.w*0.2))
        # windshield
        w = self.rect
        pygame.draw.rect(surf, WHITE, (w.x+int(self.w*0.15), w.y+int(self.h*0.12),
                                       int(self.w*0.7), int(self.h*0.22)), border_radius=10)
        # stripes
        pygame.draw.rect(surf, BLACK, (w.x+int(self.w*0.1), w.y+int(self.h*0.5),
                                       int(self.w*0.8), int(self.h*0.08)), border_radius=6)

class Enemy:
    def __init__(self, speed_mult=1.0):
        self.w = enemy_w
        self.h = enemy_h
        self.lane = random.randrange(lane_count)
        self.x = lane_center(self.lane) - self.w // 2
        self.y = -self.h - random.randint(0, int(SH*0.5))
        base_speed = SH * (0.7 + random.random()*0.5)  # px/sec
        self.speed = base_speed * speed_mult
        self.color = random.choice([RED, YELLOW, GREEN])

    @property
    def rect(self):
        return pygame.Rect(int(self.x), int(self.y), self.w, self.h)

    def update(self, dt):
        self.y += self.speed * dt

    def offscreen(self):
        return self.y > SH + self.h

    def draw(self, surf):
        pygame.draw.rect(surf, self.color, self.rect, border_radius=int(self.w*0.18))
        # headlights
        r = self.rect
        pygame.draw.rect(surf, WHITE, (r.x+int(self.w*0.2), r.bottom-int(self.h*0.18),
                                       int(self.w*0.6), int(self.h*0.10)), border_radius=8)

# --- Game state ---
STATE_MENU = 0
STATE_PLAY = 1
STATE_GAMEOVER = 2

def make_button_rects():
    left = pygame.Rect(0, SH - button_h, SW//2, button_h)
    right = pygame.Rect(SW//2, SH - button_h, SW//2, button_h)
    start_w, start_h = int(SW*0.55), int(SH*0.12)
    start = pygame.Rect((SW - start_w)//2, int(SH*0.58), start_w, start_h)
    return left, right, start

LEFT_BTN, RIGHT_BTN, START_BTN = make_button_rects()
TOUCH_ZONE = pygame.Rect(0, SH - touch_zone_h, SW, touch_zone_h)

def draw_road(surf, scroll):
    surf.fill((25,25,25))
    # road
    pygame.draw.rect(surf, GRAY, (road_x, road_y, road_w, road_h))
    # lane markers (scrolling)
    dash_h = int(SH * 0.08)
    gap = dash_h
    for lane in range(1, lane_count):
        x = road_x + lane * lane_w
        y = (-scroll) % (dash_h + gap) - dash_h
        while y < SH:
            pygame.draw.rect(surf, WHITE, (x-3, y, 6, dash_h))
            y += dash_h + gap
    # road edges
    pygame.draw.rect(surf, YELLOW, (road_x-6, 0, 6, SH))
    pygame.draw.rect(surf, YELLOW, (road_x+road_w, 0, 6, SH))

def draw_buttons(surf, holdL, holdR):
    # left
    pygame.draw.rect(surf, (0,0,0,0), LEFT_BTN, border_radius=0)
    pygame.draw.rect(surf, (255,255,255,50))
    pygame.draw.rect(surf, (255,255,255,40))
    # translucent buttons
    lcol = (255,255,255,60)
    rcol = (255,255,255,60)
    pygame.draw.rect(surf, (255,255,255,40))
    s = pygame.Surface((SW, button_h), pygame.SRCALPHA)
    pygame.draw.rect(s, (255,255,255,35), (0,0,SW,button_h))
    surf.blit(s, (0, SH - button_h))
    sbtn = pygame.Surface((LEFT_BTN.w, LEFT_BTN.h), pygame.SRCALPHA)
    pygame.draw.rect(sbtn, (255,255,255,80 if holdL else 45), sbtn.get_rect(), border_radius=0)
    surf.blit(sbtn, LEFT_BTN.topleft)
    sbtn = pygame.Surface((RIGHT_BTN.w, RIGHT_BTN.h), pygame.SRCALPHA)
    pygame.draw.rect(sbtn, (255,255,255,80 if holdR else 45), sbtn.get_rect(), border_radius=0)
    surf.blit(sbtn, RIGHT_BTN.topleft)
    # arrows
    draw_text(surf, "◀ LEFT", LEFT_BTN.centerx, LEFT_BTN.centery, BLACK, center=True, bold=True)
    draw_text(surf, "RIGHT ▶", RIGHT_BTN.centerx, RIGHT_BTN.centery, BLACK, center=True, bold=True)

def draw_start_button(surf, label="TAP TO START"):
    s = pygame.Surface(START_BTN.size, pygame.SRCALPHA)
    pygame.draw.rect(s, (255,255,255,60), s.get_rect(), border_radius=18)
    surf.blit(s, START_BTN.topleft)
    draw_text(surf, label, START_BTN.centerx, START_BTN.centery, BLACK, center=True, bold=True)

def collide(a: pygame.Rect, b: pygame.Rect):
    return a.colliderect(b)

def main():
    state = STATE_MENU
    player = Car()
    enemies = []
    score = 0.0
    best = 0.0
    spawn_timer = 0.0
    spawn_cd = 0.8  # seconds
    speed_mult = 1.0
    road_scroll = 0.0

    holding_left = False
    holding_right = False
    dragging = False
    last_drag_x = None

    global LEFT_BTN, RIGHT_BTN, START_BTN
    LEFT_BTN, RIGHT_BTN, START_BTN = make_button_rects()

    while True:
        dt = clock.tick(FPS) / 1000.0
        # --- Input ---
        drag_dx = 0
        for e in pygame.event.get():
            if e.type == pygame.QUIT:
                pygame.quit(); sys.exit()

            if e.type == pygame.KEYDOWN:
                if e.key == pygame.K_ESCAPE:
                    pygame.quit(); sys.exit()
                if state == STATE_MENU and e.key in (pygame.K_RETURN, pygame.K_SPACE):
                    state = STATE_PLAY
                    player = Car(); enemies=[]; score=0; speed_mult=1.0; spawn_cd=0.8

            if e.type == pygame.MOUSEBUTTONDOWN:
                mx, my = e.pos
                if state == STATE_MENU:
                    if START_BTN.collidepoint(mx, my):
                        state = STATE_PLAY
                        player = Car(); enemies=[]; score=0; speed_mult=1.0; spawn_cd=0.8
                elif state == STATE_GAMEOVER:
                    if START_BTN.collidepoint(mx, my):
                        state = STATE_PLAY
                        player = Car(); enemies=[]; score=0; speed_mult=1.0; spawn_cd=0.8
                else:
                    # playing
                    if LEFT_BTN.collidepoint(mx, my):  holding_left = True
                    if RIGHT_BTN.collidepoint(mx, my): holding_right = True
                    if TOUCH_ZONE.collidepoint(mx, my):
                        dragging = True
                        last_drag_x = mx

            if e.type == pygame.MOUSEBUTTONUP:
                mx, my = e.pos
                holding_left = False
                holding_right = False
                dragging = False
                last_drag_x = None

            if e.type == pygame.MOUSEMOTION and dragging and last_drag_x is not None:
                mx, my = e.pos
                drag_dx = (mx - last_drag_x) * 0.7  # sensitivity
                last_drag_x = mx

        # --- Update ---
        if state == STATE_PLAY:
            # Increase difficulty with time
            score += dt
            speed_mult = 1.0 + score * 0.05
            road_scroll += dt * SH * (0.7 + (speed_mult-1)*0.6)

            # spawn enemies
            spawn_timer += dt
            if spawn_timer >= spawn_cd:
                spawn_timer = 0.0
                enemies.append(Enemy(speed_mult=speed_mult))
                # tighten spawn as score rises
                spawn_cd = max(0.28, 0.8 - score * 0.02)

            # update enemies
            for en in enemies:
                en.update(dt)
            enemies = [en for en in enemies if not en.offscreen()]

            # player
            player.update(dt, holding_left, holding_right, drag_dx)

            # collisions
            prect = player.rect
            for en in enemies:
                if collide(prect, en.rect):
                    state = STATE_GAMEOVER
                    best = max(best, score)
                    break

        # --- Draw ---
        draw_road(screen, road_scroll)

        if state == STATE_MENU:
            draw_text(screen, "CAR DODGER", SW//2, int(SH*0.28), WHITE, center=True, bold=True, big=True)
            draw_text(screen, "Avoid traffic. Survive as long as you can.", SW//2, int(SH*0.38), WHITE, center=True)
            draw_text(screen, "Controls: LEFT / RIGHT buttons or drag at bottom.", SW//2, int(SH*0.44), WHITE, center=True)
            # demo car
            demo = Car()
            demo.y = int(SH*0.55) - demo.h//2
            demo.draw(screen)
            draw_start_button(screen, "TAP TO START")
        elif state == STATE_PLAY:
            # draw entities
            for en in enemies:
                en.draw(screen)
            player.draw(screen)

            # HUD
            draw_text(screen, f"Score: {int(score)}", road_x+10, 10, WHITE, bold=True)
            draw_text(screen, f"Best: {int(best)}", road_x+10, 10+int(SH*0.05), WHITE)
            draw_buttons(screen, holding_left, holding_right)
        else:  # GAMEOVER
            # keep last frame background
            for en in enemies:
                en.draw(screen)
            player.draw(screen)

            s = pygame.Surface((SW, SH), pygame.SRCALPHA)
            pygame.draw.rect(s, (0,0,0,160), (0,0,SW,SH))
            screen.blit(s, (0,0))

            draw_text(screen, "GAME OVER", SW//2, int(SH*0.32), WHITE, center=True, bold=True, big=True)
            draw_text(screen, f"Score: {int(score)}   Best: {int(best)}", SW//2, int(SH*0.42), WHITE, center=True, bold=True)
            draw_start_button(screen, "RESTART")

        pygame.display.flip()

if __name__ == "__main__":
    main()