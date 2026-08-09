# -*- coding: utf-8 -*-
"""
SEYTAN LEVEL - Pydroid 3 icin
Calistirmak icin: Pydroid 3 -> Pip -> "pygame" paketini kur -> bu dosyayi ac -> Run (play tusu)
Telefonu YATAY (landscape) tutarak oyna.
"""

import pygame, sys, os, json, math, random

pygame.init()
pygame.display.set_caption("Seytan Level")

# ---------------- EKRAN AYARLARI ----------------
INTERNAL_W, INTERNAL_H = 960, 540
FPS = 60

try:
    info = pygame.display.Info()
    if info.current_w > 0 and info.current_h > 0:
        screen = pygame.display.set_mode((info.current_w, info.current_h), pygame.FULLSCREEN)
    else:
        raise Exception("no info")
except Exception:
    screen = pygame.display.set_mode((960, 540))

SCREEN_W, SCREEN_H = screen.get_size()
game_surface = pygame.Surface((INTERNAL_W, INTERNAL_H))
clock = pygame.time.Clock()

def calc_blit_rect():
    scale = min(SCREEN_W / INTERNAL_W, SCREEN_H / INTERNAL_H)
    w, h = int(INTERNAL_W * scale), int(INTERNAL_H * scale)
    x, y = (SCREEN_W - w) // 2, (SCREEN_H - h) // 2
    return x, y, w, h, scale

BLIT_X, BLIT_Y, BLIT_W, BLIT_H, BLIT_SCALE = calc_blit_rect()

def screen_to_internal(sx, sy):
    ix = (sx - BLIT_X) / BLIT_SCALE
    iy = (sy - BLIT_Y) / BLIT_SCALE
    return ix, iy

try:
    FONT_BIG = pygame.font.SysFont("arial", 46, bold=True)
    FONT_MED = pygame.font.SysFont("arial", 26, bold=True)
    FONT_SMALL = pygame.font.SysFont("arial", 17)
    FONT_TINY = pygame.font.SysFont("arial", 13)
except Exception:
    FONT_BIG = pygame.font.Font(None, 52)
    FONT_MED = pygame.font.Font(None, 32)
    FONT_SMALL = pygame.font.Font(None, 22)
    FONT_TINY = pygame.font.Font(None, 18)

# ---------------- RENKLER ----------------
COL_BG_TOP = (26, 16, 48)
COL_BG_MID = (45, 22, 80)
COL_BG_BOT = (24, 10, 46)
COL_ACCENT = (255, 45, 85)
COL_ACCENT2 = (57, 255, 136)
COL_TEXT = (255, 223, 232)
COL_MUTED = (138, 122, 156)

COL_SOLID = (58, 42, 94)
COL_SOLID_TOP = (167, 139, 214)
COL_CRUMBLE = (74, 47, 110)
COL_CRUMBLE_TOP = (159, 127, 224)
COL_CRUMBLE_BREAK = (122, 48, 32)
COL_CRUMBLE_BREAK_TOP = (255, 107, 74)
COL_MOVING = (47, 95, 138)
COL_MOVING_TOP = (95, 179, 224)
COL_SPIKE_BG = (58, 13, 26)
COL_SPIKE = (255, 45, 85)

# ---------------- KIYAFETLER / MAGAZA ----------------
OUTFIT_COLORS = {
    "default": (255, 223, 232),
    "red": (255, 60, 70),
    "blue": (60, 140, 255),
    "green": (70, 220, 120),
}
OUTFIT_UNLOCK_LEVEL = {"red": 5, "blue": 10, "green": 15}
OUTFIT_NAMES_TR = {"red": "Kirmizi Kiyafet", "blue": "Mavi Kiyafet", "green": "Yesil Kiyafet"}
SHOP_COLORS_ORDER = ["default", "red", "blue", "green"]

# ---------------- KAYIT (ilerleme) ----------------
try:
    SAVE_PATH = os.path.join(os.path.dirname(os.path.abspath(__file__)), "seytan_level_save.json")
except Exception:
    SAVE_PATH = "seytan_level_save.json"

def load_progress():
    try:
        with open(SAVE_PATH, "r") as f:
            data = json.load(f)
            unlocked = max(1, min(16, int(data.get("unlocked", 1))))
            outfits = data.get("outfits", ["default"])
            if "default" not in outfits:
                outfits.append("default")
            selected = data.get("selected", "default")
            return unlocked, outfits, selected
    except Exception:
        return 1, ["default"], "default"

def save_progress(unlocked, outfits, selected):
    try:
        with open(SAVE_PATH, "w") as f:
            json.dump({"unlocked": unlocked, "outfits": outfits, "selected": selected}, f)
    except Exception:
        pass

unlocked_levels, unlocked_outfits, selected_outfit = load_progress()
total_deaths = 0

# ---------------- BOLUM VERISI ----------------
def P(x, y, w, h, t="solid", **kw):
    d = {"x": x, "y": y, "w": w, "h": h, "type": t}
    d.update(kw)
    return d

LEVELS = [
    # 1 - tutorial
    dict(start=(40, 440), goal=(880, 420, 40, 60), bg=1, hard=False, platforms=[
        P(0, 500, 300, 40, "solid"),
        P(360, 500, 200, 40, "solid"),
        P(620, 500, 340, 40, "solid"),
    ]),
    # 2 - KOLAYLASTIRILDI: artik imkansiz zipla yok, tek zararsiz tuzak var
    dict(start=(40, 440), goal=(880, 330, 40, 60), bg=1, hard=False, platforms=[
        P(0, 500, 300, 40, "solid"),
        P(300, 540, 600, 10, "spike"),
        P(340, 460, 80, 20, "fake"),      # gorunusu saglam ama gerek yok, ustune basmasan da olur
        P(420, 500, 100, 40, "solid"),
        P(560, 460, 110, 30, "solid"),
        P(720, 420, 110, 30, "solid"),
        P(860, 400, 160, 60, "solid"),
    ]),
    # 3 - crumble + spike tanisma
    dict(start=(40, 440), goal=(860, 320, 40, 60), bg=1, hard=False, platforms=[
        P(0, 500, 180, 40, "solid"),
        P(220, 500, 90, 40, "crumble"),
        P(360, 500, 90, 40, "crumble"),
        P(500, 460, 90, 40, "solid"),
        P(500, 540, 900, 20, "spike"),
        P(640, 420, 90, 40, "crumble"),
        P(780, 380, 140, 40, "solid"),
    ]),
    # 4 - hareketli platformlar
    dict(start=(40, 440), goal=(860, 280, 40, 60), bg=1, hard=False, platforms=[
        P(0, 500, 160, 40, "solid"),
        P(160, 540, 760, 10, "spike"),
        P(240, 470, 90, 24, "moving", axis="x", range=140, speed=2.2),
        P(470, 420, 90, 24, "moving", axis="x", range=120, speed=2.8),
        P(700, 370, 90, 24, "moving", axis="y", range=90, speed=2.0),
        P(840, 320, 120, 40, "solid"),
    ]),
    # 5 - COK ZOR
    dict(start=(40, 440), goal=(900, 90, 40, 60), bg=2, hard=True, platforms=[
        P(0, 500, 150, 40, "solid"),
        P(150, 540, 850, 10, "spike"),
        P(190, 470, 80, 20, "fake"),
        P(190, 500, 80, 40, "spike"),
        P(320, 440, 70, 20, "crumble"),
        P(430, 400, 70, 20, "fake"),
        P(430, 430, 70, 10, "spike"),
        P(540, 360, 70, 20, "moving", axis="x", range=110, speed=3.3),
        P(670, 320, 60, 20, "crumble"),
        P(670, 340, 60, 10, "spike"),
        P(760, 280, 60, 20, "fake"),
        P(760, 300, 60, 10, "spike"),
        P(840, 240, 60, 20, "moving", axis="y", range=100, speed=2.6),
        P(760, 190, 60, 20, "crumble"),
        P(880, 150, 140, 40, "solid"),
    ]),
    # 6 - nefes molasi
    dict(start=(40, 440), goal=(880, 340, 40, 60), bg=1, hard=False, platforms=[
        P(0, 500, 200, 40, "solid"),
        P(260, 500, 100, 40, "solid"),
        P(420, 460, 100, 40, "fake"),
        P(420, 500, 100, 40, "spike"),
        P(580, 460, 100, 40, "solid"),
        P(740, 420, 140, 40, "solid"),
    ]),
    # 7 - diken koridoru
    dict(start=(40, 440), goal=(860, 280, 40, 60), bg=1, hard=False, platforms=[
        P(0, 500, 180, 40, "solid"),
        P(180, 540, 760, 10, "spike"),
        P(210, 470, 60, 20, "solid"),
        P(320, 440, 60, 20, "crumble"),
        P(430, 410, 60, 20, "solid"),
        P(540, 440, 60, 20, "fake"),
        P(540, 470, 60, 10, "spike"),
        P(650, 410, 60, 20, "solid"),
        P(760, 380, 60, 20, "moving", axis="x", range=80, speed=2.4),
        P(840, 340, 140, 60, "solid"),
    ]),
    # 8 - dikey tirmanis
    dict(start=(40, 460), goal=(780, 20, 40, 60), bg=2, hard=False, platforms=[
        P(0, 500, 200, 40, "solid"),
        P(200, 540, 760, 10, "spike"),
        P(240, 440, 90, 20, "solid"),
        P(240, 380, 90, 20, "fake"),
        P(240, 400, 90, 10, "spike"),
        P(400, 360, 90, 20, "crumble"),
        P(400, 300, 90, 20, "solid"),
        P(560, 300, 90, 20, "moving", axis="y", range=120, speed=2.2),
        P(560, 220, 90, 20, "fake"),
        P(560, 240, 90, 10, "spike"),
        P(720, 200, 90, 20, "crumble"),
        P(720, 140, 90, 20, "solid"),
        P(780, 80, 180, 40, "solid"),
    ]),
    # 9 - cift hareketli + zamanlama
    dict(start=(40, 440), goal=(880, 160, 40, 60), bg=2, hard=False, platforms=[
        P(0, 500, 160, 40, "solid"),
        P(160, 540, 800, 10, "spike"),
        P(220, 460, 70, 20, "moving", axis="x", range=100, speed=3.0),
        P(360, 420, 70, 20, "fake"),
        P(360, 440, 70, 10, "spike"),
        P(480, 380, 70, 20, "crumble"),
        P(590, 340, 70, 20, "moving", axis="y", range=80, speed=2.5),
        P(700, 300, 70, 20, "solid"),
        P(700, 240, 70, 20, "fake"),
        P(700, 260, 70, 10, "spike"),
        P(800, 240, 140, 60, "solid"),
    ]),
    # 10 - COK ZOR
    dict(start=(40, 440), goal=(900, 60, 40, 60), bg=3, hard=True, platforms=[
        P(0, 500, 140, 40, "solid"),
        P(140, 540, 860, 10, "spike"),
        P(170, 460, 60, 20, "fake"),
        P(170, 490, 60, 10, "spike"),
        P(270, 420, 55, 18, "crumble"),
        P(360, 380, 55, 18, "fake"),
        P(360, 400, 55, 10, "spike"),
        P(450, 340, 55, 18, "moving", axis="x", range=90, speed=3.6),
        P(560, 300, 55, 18, "crumble"),
        P(560, 320, 55, 10, "spike"),
        P(650, 260, 55, 18, "fake"),
        P(650, 280, 55, 10, "spike"),
        P(740, 220, 55, 18, "moving", axis="y", range=90, speed=3.0),
        P(650, 160, 55, 18, "crumble"),
        P(760, 140, 55, 18, "fake"),
        P(760, 160, 55, 10, "spike"),
        P(850, 110, 55, 18, "solid"),
        P(880, 60, 160, 40, "solid"),
    ]),
    # 11 - nefes molasi
    dict(start=(40, 440), goal=(860, 360, 40, 60), bg=2, hard=False, platforms=[
        P(0, 500, 220, 40, "solid"),
        P(280, 480, 110, 30, "solid"),
        P(440, 460, 110, 30, "fake"),
        P(440, 500, 110, 30, "spike"),
        P(600, 440, 110, 30, "solid"),
        P(760, 420, 180, 40, "solid"),
    ]),
    # 12 - karisik
    dict(start=(40, 440), goal=(860, 260, 40, 60), bg=2, hard=False, platforms=[
        P(0, 500, 170, 40, "solid"),
        P(170, 540, 780, 10, "spike"),
        P(210, 460, 70, 20, "crumble"),
        P(320, 420, 70, 20, "moving", axis="x", range=100, speed=2.8),
        P(450, 460, 70, 20, "fake"),
        P(450, 490, 70, 10, "spike"),
        P(560, 420, 70, 20, "crumble"),
        P(670, 380, 70, 20, "solid"),
        P(780, 340, 70, 20, "moving", axis="y", range=70, speed=2.4),
        P(840, 300, 120, 60, "solid"),
    ]),
    # 13 - uzun crumble zinciri
    dict(start=(40, 440), goal=(860, 340, 40, 60), bg=2, hard=False, platforms=[
        P(0, 500, 150, 40, "solid"),
        P(150, 540, 760, 10, "spike"),
        P(190, 470, 60, 20, "crumble"),
        P(280, 470, 60, 20, "crumble"),
        P(370, 470, 60, 20, "fake"),
        P(370, 500, 60, 10, "spike"),
        P(460, 440, 60, 20, "crumble"),
        P(550, 440, 60, 20, "crumble"),
        P(640, 400, 60, 20, "moving", axis="x", range=90, speed=3.2),
        P(760, 400, 140, 60, "solid"),
    ]),
    # 14 - kombo gauntlet
    dict(start=(40, 440), goal=(860, 160, 40, 60), bg=3, hard=False, platforms=[
        P(0, 500, 140, 40, "solid"),
        P(140, 540, 820, 10, "spike"),
        P(180, 460, 60, 20, "fake"),
        P(180, 490, 60, 10, "spike"),
        P(280, 420, 60, 20, "moving", axis="y", range=80, speed=2.8),
        P(380, 380, 60, 20, "crumble"),
        P(470, 340, 60, 20, "fake"),
        P(470, 360, 60, 10, "spike"),
        P(560, 300, 60, 20, "moving", axis="x", range=100, speed=3.4),
        P(680, 300, 60, 20, "crumble"),
        P(770, 260, 60, 20, "fake"),
        P(770, 280, 60, 10, "spike"),
        P(840, 220, 160, 60, "solid"),
    ]),
    # 15 - FINAL, COK ZOR
    dict(start=(40, 440), goal=(900, 20, 40, 60), bg=3, hard=True, platforms=[
        P(0, 500, 120, 40, "solid"),
        P(120, 540, 880, 10, "spike"),
        P(150, 460, 55, 18, "fake"),
        P(150, 490, 55, 10, "spike"),
        P(240, 430, 50, 16, "crumble"),
        P(320, 400, 50, 16, "fake"),
        P(320, 420, 50, 10, "spike"),
        P(400, 360, 50, 16, "moving", axis="x", range=80, speed=3.8),
        P(500, 320, 50, 16, "crumble"),
        P(500, 340, 50, 10, "spike"),
        P(580, 280, 50, 16, "fake"),
        P(580, 300, 50, 10, "spike"),
        P(660, 240, 50, 16, "moving", axis="y", range=80, speed=3.4),
        P(580, 190, 50, 16, "crumble"),
        P(700, 180, 50, 16, "fake"),
        P(700, 200, 50, 10, "spike"),
        P(780, 150, 50, 16, "moving", axis="x", range=70, speed=4.0),
        P(880, 120, 50, 16, "crumble"),
        P(820, 90, 50, 16, "fake"),
        P(820, 110, 50, 10, "spike"),
        P(900, 20, 180, 40, "solid"),
    ]),
]

# ---------------- FIZIK (HIZLANDIRILDI) ----------------
GRAVITY = 0.62
MOVE_ACCEL = 1.35
MOVE_SPEED = 6.8
JUMP_VEL = -13.2
FRICTION = 0.88

# ---------------- OYUN DURUMU ----------------
STATE_MENU = "menu"
STATE_SELECT = "select"
STATE_PLAY = "play"
STATE_GAMEWIN = "gamewin"
STATE_SHOP = "shop"

state = STATE_MENU
current_level = 0
player = None
runtime_platforms = []
particles = []
cam_x = 0.0
level_deaths = 0
msg_text = ""
msg_sub = ""
msg_timer = 0
transitioning = False
trans_timer = 0

# touch/mouse pointer tracking: id -> (x, y) internal coords
active_pointers = {}
prev_jump_state = False

# ---------------- BUTON DIKDORTGENLERI (internal koordinatlar) ----------------
BTN_LEFT = pygame.Rect(30, INTERNAL_H - 130, 90, 90)
BTN_RIGHT = pygame.Rect(140, INTERNAL_H - 130, 90, 90)
BTN_JUMP = pygame.Rect(INTERNAL_W - 150, INTERNAL_H - 140, 110, 110)
BTN_BACK = pygame.Rect(14, 14, 70, 40)

MENU_BTN_PLAY = pygame.Rect(INTERNAL_W // 2 - 140, 280, 280, 60)
MENU_BTN_SELECT = pygame.Rect(INTERNAL_W // 2 - 140, 360, 280, 60)
MENU_BTN_SHOP = pygame.Rect(INTERNAL_W // 2 - 140, 440, 280, 60)

SELECT_BACK = pygame.Rect(20, 20, 100, 44)
GRID_COLS, GRID_ROWS = 5, 3
GRID_CELL = 130
GRID_GAP = 18
GRID_START_X = (INTERNAL_W - (GRID_COLS * GRID_CELL + (GRID_COLS - 1) * GRID_GAP)) // 2
GRID_START_Y = 130

SHOP_BACK = pygame.Rect(20, 20, 100, 44)

def level_cell_rect(i):
    col = i % GRID_COLS
    row = i // GRID_COLS
    x = GRID_START_X + col * (GRID_CELL + GRID_GAP)
    y = GRID_START_Y + row * (GRID_CELL + GRID_GAP)
    return pygame.Rect(x, y, GRID_CELL, GRID_CELL)

def shop_cell_rect(i):
    w, h = 170, 190
    gap = 26
    total_w = len(SHOP_COLORS_ORDER) * w + (len(SHOP_COLORS_ORDER) - 1) * gap
    start_x = (INTERNAL_W - total_w) // 2
    x = start_x + i * (w + gap)
    y = 190
    return pygame.Rect(x, y, w, h)

# ---------------- YARDIMCI ----------------
def rects_overlap(a, b):
    return a["x"] < b["x"] + b["w"] and a["x"] + a["w"] > b["x"] and a["y"] < b["y"] + b["h"] and a["y"] + a["h"] > b["y"]

def show_msg(title, sub="", duration=0):
    global msg_text, msg_sub, msg_timer
    msg_text = title
    msg_sub = sub
    msg_timer = duration

def load_level(i):
    global player, runtime_platforms, cam_x, current_level, level_deaths
    current_level = i
    lvl = LEVELS[i]
    sx, sy = lvl["start"]
    player = {"x": sx, "y": sy, "w": 26, "h": 34, "vx": 0.0, "vy": 0.0,
              "on_ground": False, "dead": False}
    runtime_platforms = []
    for p in lvl["platforms"]:
        rp = dict(p)
        rp["gone"] = False
        rp["crumbling"] = False
        rp["crumble_t"] = 0
        rp["stepped"] = False
        rp["phase"] = random.uniform(0, math.pi * 2)
        rp["cur_x"] = rp["x"]
        rp["cur_y"] = rp["y"]
        runtime_platforms.append(rp)
    cam_x = 0.0
    level_deaths = 0
    if lvl["hard"]:
        show_msg("BOLUM {0} - COK ZOR!".format(i + 1), "Iyi sanslar...", 90)
    else:
        show_msg("BOLUM {0}".format(i + 1), "", 70)

def die_player():
    global level_deaths, total_deaths
    if player["dead"]:
        return
    player["dead"] = True
    level_deaths += 1
    total_deaths += 1
    for _ in range(18):
        particles.append({
            "x": player["x"] + player["w"] / 2, "y": player["y"] + player["h"] / 2,
            "vx": random.uniform(-4, 4), "vy": random.uniform(-4, 4), "life": 28
        })

def win_level():
    global unlocked_levels, unlocked_outfits, state
    nxt = current_level + 2
    newly_unlocked_outfit = None
    if nxt > unlocked_levels:
        unlocked_levels = min(16, nxt)
        for color, req in OUTFIT_UNLOCK_LEVEL.items():
            if unlocked_levels >= req and color not in unlocked_outfits:
                unlocked_outfits.append(color)
                newly_unlocked_outfit = color
        save_progress(unlocked_levels, unlocked_outfits, selected_outfit)
    if current_level >= len(LEVELS) - 1:
        state = STATE_GAMEWIN
    else:
        load_level(current_level + 1)
        if newly_unlocked_outfit:
            show_msg("BOLUM {0}".format(current_level + 1),
                      "Yeni kiyafet acildi: {0}! Magazadan giy.".format(OUTFIT_NAMES_TR[newly_unlocked_outfit]), 150)

# ---------------- GUNCELLEME ----------------
def get_held_buttons():
    held = {"left": False, "right": False, "jump": False}
    for pid, (x, y) in active_pointers.items():
        if BTN_LEFT.collidepoint(x, y):
            held["left"] = True
        if BTN_RIGHT.collidepoint(x, y):
            held["right"] = True
        if BTN_JUMP.collidepoint(x, y):
            held["jump"] = True
    keys = pygame.key.get_pressed()
    if keys[pygame.K_LEFT] or keys[pygame.K_a]:
        held["left"] = True
    if keys[pygame.K_RIGHT] or keys[pygame.K_d]:
        held["right"] = True
    if keys[pygame.K_UP] or keys[pygame.K_w] or keys[pygame.K_SPACE]:
        held["jump"] = True
    return held

def update_play():
    global cam_x, prev_jump_state, msg_timer

    if msg_timer > 0:
        msg_timer -= 1

    if player["dead"]:
        for pt in particles:
            pt["x"] += pt["vx"]; pt["y"] += pt["vy"]; pt["vy"] += 0.3; pt["life"] -= 1
        particles[:] = [pt for pt in particles if pt["life"] > 0]
        if len([1 for pt in particles]) == 0 or True:
            player["_respawn_t"] = player.get("_respawn_t", 0) + 1
            if player["_respawn_t"] > 26:
                load_level(current_level)
        return

    held = get_held_buttons()
    if held["left"]:
        player["vx"] -= MOVE_ACCEL
    if held["right"]:
        player["vx"] += MOVE_ACCEL
    player["vx"] *= FRICTION
    if abs(player["vx"]) > MOVE_SPEED:
        player["vx"] = MOVE_SPEED if player["vx"] > 0 else -MOVE_SPEED

    jump_edge = held["jump"] and not prev_jump_state
    prev_jump_state = held["jump"]
    if jump_edge and player["on_ground"]:
        player["vy"] = JUMP_VEL
        player["on_ground"] = False

    player["vy"] += GRAVITY
    if player["vy"] > 18:
        player["vy"] = 18

    for p in runtime_platforms:
        if p["type"] == "moving" and not p["gone"]:
            p["phase"] += 0.032 * p.get("speed", 2)
            off = math.sin(p["phase"]) * (p.get("range", 100) / 2)
            p["cur_x"] = p["x"] + (off if p.get("axis") == "x" else 0)
            p["cur_y"] = p["y"] + (off if p.get("axis") == "y" else 0)
        else:
            p["cur_x"] = p["x"]
            p["cur_y"] = p["y"]

    # X hareketi
    player["x"] += player["vx"]
    for p in runtime_platforms:
        if p["gone"] or p["type"] in ("spike", "fake"):
            continue
        box = {"x": p["cur_x"], "y": p["cur_y"], "w": p["w"], "h": p["h"]}
        if rects_overlap(player, box):
            if player["vx"] > 0:
                player["x"] = box["x"] - player["w"]
            elif player["vx"] < 0:
                player["x"] = box["x"] + box["w"]
            player["vx"] = 0

    # Y hareketi
    player["on_ground"] = False
    prev_vy = player["vy"]
    player["y"] += player["vy"]
    for p in runtime_platforms:
        if p["gone"] or p["type"] in ("spike", "fake"):
            continue
        box = {"x": p["cur_x"], "y": p["cur_y"], "w": p["w"], "h": p["h"]}
        if rects_overlap(player, box):
            if prev_vy > 0 and (player["y"] + player["h"] - prev_vy) <= box["y"] + 2:
                player["y"] = box["y"] - player["h"]
                player["vy"] = 0
                player["on_ground"] = True
                if p["type"] == "crumble":
                    p["stepped"] = True
            elif prev_vy < 0 and (player["y"] - prev_vy) >= box["y"] + box["h"] - 2:
                player["y"] = box["y"] + box["h"]
                player["vy"] = 0

    for p in runtime_platforms:
        if p["type"] == "crumble" and not p["gone"]:
            if p["stepped"] and not p["crumbling"]:
                p["crumbling"] = True
                p["crumble_t"] = 0
            if p["crumbling"]:
                p["crumble_t"] += 16
                if p["crumble_t"] > 350:
                    p["gone"] = True

    if player["y"] > INTERNAL_H + 100:
        die_player()
        return

    for p in runtime_platforms:
        if p["gone"] or p["type"] != "spike":
            continue
        spike_box = {"x": p["cur_x"] + 4, "y": p["cur_y"] + 4, "w": p["w"] - 8, "h": p["h"]}
        if rects_overlap(player, spike_box):
            die_player()
            return

    lvl = LEVELS[current_level]
    gx, gy, gw, gh = lvl["goal"]
    goal_box = {"x": gx, "y": gy, "w": gw, "h": gh}
    if rects_overlap(player, goal_box):
        win_level()
        return

    cam_x = max(0, min(player["x"] - INTERNAL_W / 2, 1400))

# ---------------- CIZIM ----------------
def draw_vgrad(surf, rect, top_col, bot_col):
    x, y, w, h = rect
    for i in range(h):
        t = i / max(1, h - 1)
        col = tuple(int(top_col[c] + (bot_col[c] - top_col[c]) * t) for c in range(3))
        pygame.draw.line(surf, col, (x, y + i), (x + w, y + i))

def draw_bg(bg_level):
    draw_vgrad(game_surface, (0, 0, INTERNAL_W, INTERNAL_H), COL_BG_TOP, COL_BG_BOT)
    n = 10 if bg_level == 3 else 7 if bg_level == 2 else 4
    for i in range(n):
        x = ((i * 137 - cam_x * 0.3) % (INTERNAL_W + 200)) - 100
        col = COL_ACCENT if i % 2 == 0 else (123, 63, 228)
        s = pygame.Surface((60, 60), pygame.SRCALPHA)
        pygame.draw.circle(s, col + (60,), (30, 30), 30)
        game_surface.blit(s, (x - 30, 30 + (i % 3) * 90))

def draw_platform(p):
    if p["gone"]:
        return
    px = p["cur_x"] - cam_x
    py = p["cur_y"]
    w, h = p["w"], p["h"]
    if px + w < -20 or px > INTERNAL_W + 20:
        return

    if p["type"] == "spike":
        pygame.draw.rect(game_surface, COL_SPIKE_BG, (px, py, w, h))
        spike_w = 14
        sx = px
        while sx < px + w:
            pts = [(sx, py + h), (sx + spike_w / 2, py - 4), (sx + spike_w, py + h)]
            pygame.draw.polygon(game_surface, COL_SPIKE, pts)
            sx += spike_w
        return

    fill, top = COL_SOLID, COL_SOLID_TOP
    if p["type"] == "crumble":
        if p["crumbling"]:
            fill, top = COL_CRUMBLE_BREAK, COL_CRUMBLE_BREAK_TOP
        else:
            fill, top = COL_CRUMBLE, COL_CRUMBLE_TOP
    elif p["type"] == "moving":
        fill, top = COL_MOVING, COL_MOVING_TOP
    elif p["type"] in ("solid", "fake"):
        fill, top = COL_SOLID, COL_SOLID_TOP  # fake gorunusce solid ile AYNI - bu tuzagin sirri

    pygame.draw.rect(game_surface, fill, (px, py, w, h))
    pygame.draw.rect(game_surface, top, (px, py, w, 4))

def draw_player():
    px = player["x"] - cam_x
    py = player["y"]
    body_col = OUTFIT_COLORS.get(selected_outfit, COL_TEXT)
    pygame.draw.rect(game_surface, body_col, (px, py, player["w"], player["h"]), border_radius=4)
    pygame.draw.rect(game_surface, COL_ACCENT, (px + 4, py + 8, 6, 6))
    pygame.draw.rect(game_surface, COL_ACCENT, (px + player["w"] - 10, py + 8, 6, 6))

def draw_goal():
    lvl = LEVELS[current_level]
    gx, gy, gw, gh = lvl["goal"]
    px, py = gx - cam_x, gy
    pygame.draw.rect(game_surface, (20, 40, 20), (px, py, gw, gh))
    pygame.draw.rect(game_surface, COL_ACCENT2, (px, py, 6, gh))
    pygame.draw.polygon(game_surface, COL_ACCENT2, [(px + 6, py + 6), (px + 34, py + 16), (px + 6, py + 26)])

def draw_button(rect, label, active=False):
    col = COL_ACCENT if active else (255, 45, 85)
    s = pygame.Surface((rect.w, rect.h), pygame.SRCALPHA)
    pygame.draw.rect(s, (255, 45, 85, 60 if not active else 140), s.get_rect(), border_radius=16)
    pygame.draw.rect(s, (255, 45, 85, 200), s.get_rect(), width=2, border_radius=16)
    game_surface.blit(s, rect.topleft)
    txt = FONT_SMALL.render(label, True, (255, 200, 210))
    game_surface.blit(txt, (rect.centerx - txt.get_width() // 2, rect.centery - txt.get_height() // 2))

def draw_play():
    lvl = LEVELS[current_level]
    draw_bg(lvl.get("bg", 1))
    draw_goal()
    for p in runtime_platforms:
        draw_platform(p)
    if not player["dead"]:
        draw_player()
    for pt in particles:
        s = pygame.Surface((5, 5), pygame.SRCALPHA)
        alpha = max(0, min(255, int(255 * pt["life"] / 28)))
        s.fill((255, 45, 85, alpha))
        game_surface.blit(s, (pt["x"] - cam_x, pt["y"]))

    held = get_held_buttons()
    draw_button(BTN_LEFT, "<", held["left"])
    draw_button(BTN_RIGHT, ">", held["right"])
    draw_button(BTN_JUMP, "ZIPLA", held["jump"])
    draw_button(BTN_BACK, "X", False)

    # HUD
    hud1 = FONT_SMALL.render("BOLUM {0} / {1}".format(current_level + 1, len(LEVELS)), True, COL_TEXT)
    hud2 = FONT_SMALL.render("OLUM: {0}".format(level_deaths), True, COL_TEXT)
    game_surface.blit(hud1, (100, 20))
    game_surface.blit(hud2, (INTERNAL_W - hud2.get_width() - 20, 20))

    if msg_timer > 0 or msg_text:
        alpha = 255 if msg_timer > 15 else max(0, int(255 * msg_timer / 15))
        if alpha > 0:
            t1 = FONT_BIG.render(msg_text, True, COL_TEXT)
            s1 = pygame.Surface(t1.get_size(), pygame.SRCALPHA)
            s1.blit(t1, (0, 0))
            s1.set_alpha(alpha)
            game_surface.blit(s1, (INTERNAL_W // 2 - t1.get_width() // 2, INTERNAL_H // 2 - 60))
            if msg_sub:
                t2 = FONT_SMALL.render(msg_sub, True, (255, 179, 198))
                s2 = pygame.Surface(t2.get_size(), pygame.SRCALPHA)
                s2.blit(t2, (0, 0))
                s2.set_alpha(alpha)
                game_surface.blit(s2, (INTERNAL_W // 2 - t2.get_width() // 2, INTERNAL_H // 2))

def draw_menu():
    draw_bg(2)
    title = FONT_BIG.render("SEYTAN LEVEL", True, COL_ACCENT)
    game_surface.blit(title, (INTERNAL_W // 2 - title.get_width() // 2, 120))
    sub = FONT_SMALL.render("15 BOLUM * GUVENME HICBIR SEYE", True, COL_MUTED)
    game_surface.blit(sub, (INTERNAL_W // 2 - sub.get_width() // 2, 185))

    draw_button(MENU_BTN_PLAY, "OYNA")
    draw_button(MENU_BTN_SELECT, "BOLUM SEC")
    draw_button(MENU_BTN_SHOP, "MAGAZA")

    info = FONT_TINY.render("Acilan en yuksek bolum: {0} / 15   Toplam olum: {1}".format(unlocked_levels, total_deaths), True, COL_MUTED)
    game_surface.blit(info, (INTERNAL_W // 2 - info.get_width() // 2, 520))

def draw_select():
    draw_bg(2)
    title = FONT_MED.render("BOLUM SEC", True, COL_TEXT)
    game_surface.blit(title, (INTERNAL_W // 2 - title.get_width() // 2, 40))
    draw_button(SELECT_BACK, "GERI")

    for i in range(len(LEVELS)):
        rect = level_cell_rect(i)
        locked = (i + 1) > unlocked_levels
        hard = LEVELS[i]["hard"]
        s = pygame.Surface((rect.w, rect.h), pygame.SRCALPHA)
        if locked:
            pygame.draw.rect(s, (60, 50, 80, 140), s.get_rect(), border_radius=14)
            pygame.draw.rect(s, (100, 90, 120, 200), s.get_rect(), width=2, border_radius=14)
        else:
            base_col = (255, 45, 85) if hard else (95, 179, 224)
            pygame.draw.rect(s, base_col + (70,), s.get_rect(), border_radius=14)
            pygame.draw.rect(s, base_col + (220,), s.get_rect(), width=2, border_radius=14)
        game_surface.blit(s, rect.topleft)

        if locked:
            lock_txt = FONT_MED.render("?", True, (150, 140, 170))
            game_surface.blit(lock_txt, (rect.centerx - lock_txt.get_width() // 2, rect.centery - lock_txt.get_height() // 2 - 6))
        else:
            num_txt = FONT_MED.render(str(i + 1), True, COL_TEXT)
            game_surface.blit(num_txt, (rect.centerx - num_txt.get_width() // 2, rect.centery - num_txt.get_height() // 2 - 6))
            if hard:
                hard_txt = FONT_TINY.render("ZOR", True, (255, 220, 220))
                game_surface.blit(hard_txt, (rect.centerx - hard_txt.get_width() // 2, rect.bottom - 22))

def draw_shop():
    draw_bg(2)
    title = FONT_MED.render("MAGAZA - KIYAFETLER", True, COL_TEXT)
    game_surface.blit(title, (INTERNAL_W // 2 - title.get_width() // 2, 40))
    draw_button(SHOP_BACK, "GERI")

    for i, key in enumerate(SHOP_COLORS_ORDER):
        rect = shop_cell_rect(i)
        locked = key not in unlocked_outfits
        col = OUTFIT_COLORS[key]
        s = pygame.Surface((rect.w, rect.h), pygame.SRCALPHA)
        if locked:
            pygame.draw.rect(s, (60, 50, 80, 140), s.get_rect(), border_radius=14)
            pygame.draw.rect(s, (100, 90, 120, 200), s.get_rect(), width=2, border_radius=14)
        else:
            border_col = COL_ACCENT2 + (255,) if key == selected_outfit else col + (220,)
            pygame.draw.rect(s, col + (80,), s.get_rect(), border_radius=14)
            pygame.draw.rect(s, border_col, s.get_rect(), width=3 if key == selected_outfit else 2, border_radius=14)
        game_surface.blit(s, rect.topleft)

        if locked:
            lock_txt = FONT_MED.render("?", True, (150, 140, 170))
            game_surface.blit(lock_txt, (rect.centerx - lock_txt.get_width() // 2, rect.y + 55))
        else:
            pygame.draw.rect(game_surface, col, (rect.centerx - 32, rect.y + 30, 64, 64), border_radius=10)
            if key == selected_outfit:
                sec_txt = FONT_TINY.render("GIYILI", True, COL_ACCENT2)
                game_surface.blit(sec_txt, (rect.centerx - sec_txt.get_width() // 2, rect.y + 100))

        label = "Varsayilan" if key == "default" else OUTFIT_NAMES_TR[key]
        lbl_txt = FONT_TINY.render(label, True, COL_TEXT if not locked else COL_MUTED)
        game_surface.blit(lbl_txt, (rect.centerx - lbl_txt.get_width() // 2, rect.bottom - 40))

        if locked:
            need = OUTFIT_UNLOCK_LEVEL[key]
            req_txt = FONT_TINY.render("Bolum {0}'de acilir".format(need), True, COL_MUTED)
            game_surface.blit(req_txt, (rect.centerx - req_txt.get_width() // 2, rect.bottom - 20))

def draw_gamewin():
    draw_bg(3)
    t1 = FONT_BIG.render("TUM BOLUMLER BITTI!", True, COL_ACCENT2)
    game_surface.blit(t1, (INTERNAL_W // 2 - t1.get_width() // 2, 200))
    t2 = FONT_SMALL.render("Toplam olum: {0}".format(total_deaths), True, COL_TEXT)
    game_surface.blit(t2, (INTERNAL_W // 2 - t2.get_width() // 2, 270))
    draw_button(GAMEWIN_SELECT_BTN, "BOLUM SEC")
    draw_button(GAMEWIN_MENU_BTN, "ANA MENU")

GAMEWIN_SELECT_BTN = pygame.Rect(INTERNAL_W // 2 - 140, 380, 280, 60)
GAMEWIN_MENU_BTN = pygame.Rect(INTERNAL_W // 2 - 140, 460, 280, 60)

# ---------------- OLAY YONETIMI ----------------
def handle_tap(ix, iy):
    global state, current_level, selected_outfit
    if state == STATE_MENU:
        if MENU_BTN_PLAY.collidepoint(ix, iy):
            start_at = min(unlocked_levels, len(LEVELS)) - 1
            state = STATE_PLAY
            load_level(max(0, start_at))
        elif MENU_BTN_SELECT.collidepoint(ix, iy):
            state = STATE_SELECT
        elif MENU_BTN_SHOP.collidepoint(ix, iy):
            state = STATE_SHOP
    elif state == STATE_SELECT:
        if SELECT_BACK.collidepoint(ix, iy):
            state = STATE_MENU
            return
        for i in range(len(LEVELS)):
            if (i + 1) <= unlocked_levels and level_cell_rect(i).collidepoint(ix, iy):
                state = STATE_PLAY
                load_level(i)
                return
    elif state == STATE_SHOP:
        if SHOP_BACK.collidepoint(ix, iy):
            state = STATE_MENU
            return
        for i, key in enumerate(SHOP_COLORS_ORDER):
            if shop_cell_rect(i).collidepoint(ix, iy) and key in unlocked_outfits:
                selected_outfit = key
                save_progress(unlocked_levels, unlocked_outfits, selected_outfit)
                return
    elif state == STATE_PLAY:
        if BTN_BACK.collidepoint(ix, iy):
            state = STATE_SELECT
    elif state == STATE_GAMEWIN:
        if GAMEWIN_SELECT_BTN.collidepoint(ix, iy):
            state = STATE_SELECT
        elif GAMEWIN_MENU_BTN.collidepoint(ix, iy):
            state = STATE_MENU

def process_events():
    global active_pointers
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            pygame.quit(); sys.exit()
        elif event.type == pygame.KEYDOWN and event.key == pygame.K_ESCAPE:
            pygame.quit(); sys.exit()

        elif event.type == pygame.MOUSEBUTTONDOWN:
            ix, iy = screen_to_internal(*event.pos)
            active_pointers["mouse"] = (ix, iy)
            handle_tap(ix, iy)
        elif event.type == pygame.MOUSEMOTION:
            if "mouse" in active_pointers:
                ix, iy = screen_to_internal(*event.pos)
                active_pointers["mouse"] = (ix, iy)
        elif event.type == pygame.MOUSEBUTTONUP:
            active_pointers.pop("mouse", None)

        elif event.type == pygame.FINGERDOWN:
            sx, sy = event.x * SCREEN_W, event.y * SCREEN_H
            ix, iy = screen_to_internal(sx, sy)
            active_pointers[event.finger_id] = (ix, iy)
            handle_tap(ix, iy)
        elif event.type == pygame.FINGERMOTION:
            sx, sy = event.x * SCREEN_W, event.y * SCREEN_H
            ix, iy = screen_to_internal(sx, sy)
            if event.finger_id in active_pointers:
                active_pointers[event.finger_id] = (ix, iy)
        elif event.type == pygame.FINGERUP:
            active_pointers.pop(event.finger_id, None)

# ---------------- ANA DONGU ----------------
def main():
    global state
    while True:
        process_events()

        if state == STATE_PLAY:
            update_play()

        if state == STATE_MENU:
            draw_menu()
        elif state == STATE_SELECT:
            draw_select()
        elif state == STATE_PLAY:
            draw_play()
        elif state == STATE_GAMEWIN:
            draw_gamewin()
        elif state == STATE_SHOP:
            draw_shop()

        scale_surf = pygame.transform.smoothscale(game_surface, (BLIT_W, BLIT_H))
        screen.fill((5, 3, 10))
        screen.blit(scale_surf, (BLIT_X, BLIT_Y))
        pygame.display.flip()
        clock.tick(FPS)

if __name__ == "__main__":
    main()
