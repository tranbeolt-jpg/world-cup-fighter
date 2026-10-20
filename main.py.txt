"""
Menu "World Cup Fighter" cho Streamlit.

Cài đặt và chạy:
    pip install -U streamlit
    streamlit run game_menu.py

Lần chạy đầu, file tự tạo thư mục wcf_menu/ cạnh file này để chứa giao diện.
Nhạc nền: đặt file nhạc của bạn cạnh file này (xem mục NHẠC NỀN bên dưới).
Âm thanh bấm nút (click, trống tùng tùng, tiếng chiêng) được tự tổng hợp, xem mục ÂM THANH.
Khi bấm mục menu, biến `choice` nhận tên mục: "single", "two", "editor",
"options", "menu" (khi bấm BACK), hoặc "pick:07" (khi chọn nhân vật 07), hoặc "start:07:hard" (khi chọn xong nhân vật và độ khó), "pause", "resume" hoặc "menu" (nút ở màn đánh nhau).
Viết logic game ở cuối file.
"""
import html
import json
import random
import shutil
from pathlib import Path

import streamlit as st
import streamlit.components.v1 as components

# ═══════════════ CẤU HÌNH: sửa nội dung ở đây ═══════════════
GAME_TITLE = "WORLD CUP FIGHTER"
SUBTITLE = "AGE OF THE WARRIOR"
SITE_URL = "WWW.WORLDCUPFIGHTER.COM"
CREDIT_LEFT = "A Your Name Game. © 2026."
CREDIT_RIGHT = "More games at: www.yoursite.com"

FONT = "Rampart One"      # font kiểu chữ cọ trên Google Fonts (có thể đổi)
TEXT_COLOR = "#fbf5a4"    # vàng nhạt như chữ trong ảnh
HOVER_COLOR = "#ffb43c"   # màu khi rê chuột hoặc chọn bằng phím Tab
INK_COLOR = "#2a1a06"     # màu dòng credit phía trên

# ═══════════════ NHẠC NỀN ═══════════════
# Đặt file nhạc BẠN CÓ QUYỀN DÙNG cạnh game_menu.py, ví dụ mars.mp3
# Gợi ý: "Mars, the Bringer of War" (Holst) hoặc "Ride of the Valkyries" (Wagner) đều là nhạc cổ điển hùng tráng.
MUSIC_FILE = "mars.mp3"  # None = không phát nhạc
MUSIC_START = "0:00"         # giây bắt đầu đoạn muốn phát, dạng "phút:giây", ví dụ "1:23"
MUSIC_END = None             # giây kết thúc đoạn lặp lại; None = phát hết bài rồi lặp từ MUSIC_START
MUSIC_VOLUME = 0.6           # âm lượng từ 0 đến 1

# ═══════════════ ÂM THANH KHI BẤM NÚT ═══════════════
# Âm thanh được tự tổng hợp trong trình duyệt, không cần file ghi âm.
SFX_ON = True        # False = tắt âm thanh khi bấm
SFX_VOLUME = 0.6     # âm lượng hiệu ứng từ 0 đến 1
# "drum" = trống tùng tùng, "gong" = tiếng chiêng, "click" = tiếng click
SFX_BUTTONS = {
    "single": "drum",
    "two": "drum",
    "editor": "click",
    "options": "gong",
    "menu": "click",  # nút BACK
    "easy": "click",  # độ khó EASY
    "hard": "drum",  # độ khó HARD
    "impossible": "gong",  # độ khó IMPOSSIBLE
}
SFX_OPTION = "click"  # khi chọn On/Off, Normal, Low... trong màn hình options

# Mục menu: (tên hiển thị, mã, x mép trái chữ, độ dài chữ, đường nền y)
# Toạ độ theo ảnh gốc 1448 × 1086. Đổi tên thì chỉnh lại độ dài cho khớp.
MENU = [
    ("SINGLE PLAYER", "single", 1052, 352, 815),
    ("2 PLAYER", "two", 1186, 218, 892),
    ("CHARACTER EDITOR", "editor", 958, 446, 970),
    ("OPTIONS/CONTROLS", "options", 940, 464, 1047),
]
BACK = ("BACK", "back", 1294, 110, 1047)  # nút quay lại ở màn hình con

# ═══════════════ ÁO ĐỘI BÓNG CỦA NHÂN VẬT ═══════════════
# Các đội do bạn tự đặt (không dùng logo hay áo của đội bóng thật). Có thể đổi tên và màu tuỳ ý.
# pattern: "plain" (một màu), "hoops" (sọc ngang), "vertical" (sọc dọc), "sash" (sọc chéo)
KITS = [
    {"name": "Rồng Tím", "base": "#6a1b9a", "stripe": "#ffd54f", "pattern": "sash", "trim": "#ffd54f"},
    {"name": "Rừng Xanh", "base": "#1b5e20", "stripe": "#66bb6a", "pattern": "vertical", "trim": "#ffffff"},
    {"name": "Lửa Cam", "base": "#ef6c00", "stripe": "#ef6c00", "pattern": "plain", "trim": "#1a1a1a"},
    {"name": "Sóng Xanh", "base": "#0d47a1", "stripe": "#4fc3f7", "pattern": "hoops", "trim": "#ffffff"},
]

W, H = 1448, 1086
MENU_RIGHT = 1404  # mép phải của cột menu
HIT_RIGHT = 1410   # mép phải vùng bấm

# ═══════════════ MÀN HÌNH OPTIONS / CONTROLS (theo ảnh tham khảo) ═══════════════
OPT_PANELS = [(152, 27, 381, 418), (550, 27, 380, 430), (947, 27, 418, 450)]  # x, y, rộng, cao
OPT_BACK_HIT = (22, 702, 530, 58)  # vùng bấm "< BACK TO MAIN MENU"
# (nhãn, đường nền y, [(nhóm, giá trị, chữ, mép trái, rộng ô, đang chọn)])
OPT_ROWS = [
    ("Weapons:", 177, [("weapons", "none", "None", 296, 58, False),
                       ("weapons", "normal", "Normal", 368, 78, True),
                       ("weapons", "more", "More", 462, 50, False)]),
    ("Blood:", 213, [("blood", "on", "On", 295, 38, True),
                     ("blood", "off", "Off", 347, 43, False)]),
    ("Slow Mo:", 249, [("slowmo", "on", "On", 295, 38, True),
                       ("slowmo", "off", "Off", 347, 43, False)]),
    ("Quality:", 285, [("quality", "low", "Low", 296, 48, True),
                       ("quality", "medium", "Medium", 362, 78, False),
                       ("quality", "high", "High", 462, 45, False)]),
    ("Music:", 411, [("music", "on", "On", 296, 37, True),
                     ("music", "off", "Off", 347, 43, False)]),
]
# (phím, mô tả, đường nền y)
P1_ROWS = [("A/D", "Walk Left/Right", 123), ("W", "Jump", 159), ("S", "Block", 195),
           ("T", "Punch", 249), ("Y", "Kick", 285), ("U", "Draw/Throw Weapon", 339)]
P2_ROWS = [("Left/Right", "Walk Left/Right", 123), ("Up", "Jump", 159), ("Down", "Block", 195),
           ("4", "Punch", 249), ("5", "Kick", 285), ("6", "Draw/Throw Weapon", 339)]
P1_MOVES = [("T+Y", "Special Move", 501), ("S+T", "Grab", 537),
            ("S+Y", "Sweep (When close)", 574), ("S+U", "Pick Up/Put Down Weapon", 610)]
P2_MOVES = [("4+5", "Special Move", 501), ("Down+4", "Grab", 537),
            ("Down+5", "Sweep (When close)", 574), ("Down+6", "Pick Up/Put Down Weapon", 610)]
PARA = ("Use different combinations of", "directions and attacks to", "perform different moves")
PARA_Y = (394, 422, 450)

# ═══════════════ HÌNH VẼ (vẽ lại theo bố cục ảnh gốc) ═══════════════
CLOUDS = [
    "M186,80 C184,64 200,54 222,55 C236,44 262,44 276,54 C292,50 300,68 296,78 C292,90 272,92 252,91 C232,93 204,94 190,90 C186,88 186,84 186,80 Z",
    "M420,70 C440,56 480,54 500,62 C516,70 514,96 500,106 C486,114 460,112 446,116 C436,120 430,130 428,140 C436,152 430,168 418,178 C412,190 420,206 412,220 C400,236 360,240 332,234 C312,230 300,218 300,204 C300,190 310,182 322,176 C340,166 352,150 352,136 C352,122 360,114 372,110 C380,100 396,84 420,70 Z",
    "M410,130 C420,122 446,124 468,126 C488,130 500,140 494,152 C486,164 460,166 440,164 C424,162 408,156 406,146 C405,138 406,134 410,130 Z",
    "M0,108 C36,110 66,128 80,156 C90,180 84,200 72,212 C98,228 128,250 150,282 C168,306 166,336 150,352 C128,370 96,372 70,364 C42,356 20,344 0,332 Z",
    "M598,0 L846,0 C852,20 852,54 842,82 C830,110 804,124 776,122 C756,120 744,108 730,98 C712,86 684,88 662,80 C636,72 614,62 604,44 C600,32 598,16 598,0 Z",
    "M960,96 C964,80 990,70 1020,72 C1050,62 1088,62 1110,70 C1142,58 1196,58 1222,68 C1252,66 1280,74 1292,94 C1300,112 1286,122 1262,124 C1232,126 1196,128 1170,134 C1140,142 1112,148 1082,146 C1048,144 1020,140 996,134 C976,128 958,116 960,96 Z",
    "M1246,122 C1272,114 1302,126 1316,150 C1326,170 1320,194 1312,206 C1336,212 1368,224 1366,240 C1364,256 1326,262 1290,266 C1250,270 1196,272 1162,266 C1134,260 1130,242 1150,232 C1170,222 1206,220 1240,218 C1256,216 1258,200 1258,180 C1258,156 1252,136 1246,122 Z",
    "M1322,40 C1350,30 1400,24 1448,30 L1448,214 C1424,218 1392,214 1374,200 C1356,186 1362,166 1372,156 C1384,144 1376,132 1360,128 C1342,124 1326,114 1322,96 C1318,78 1322,56 1322,40 Z",
]
HILL = (
    "M0,524 C40,520 80,516 112,512 C170,498 226,468 300,458 "
    "C350,450 395,446 432,447 C500,450 560,458 620,466 "
    "C700,476 780,486 860,494 C930,500 990,508 1060,516 "
    "C1180,528 1300,540 1448,560 L1448,1086 L0,1086 Z"
)
# Một nhân vật (chân ở y≈357, tâm x=0). Quả bóng nằm bên trái, tay cầm bóng.
ATHLETE = (
    '<ellipse cx="0" cy="31" rx="20" ry="31"/>'
    '<path d="M-9,54 L9,54 L11,80 L-11,80 Z"/>'
    '<path d="M-14,74 Q-30,80 -40,78 Q-54,80 -57,96 Q-59,112 -53,132 Q-48,160 -44,196 '
    'L44,196 Q48,160 53,132 Q59,112 57,96 Q54,80 40,78 Q30,80 14,74 Z"/>'
    '<path d="M-45,190 L45,190 L51,254 L20,254 L0,226 L-20,254 L-51,254 Z"/>'
    '<path d="M-47,92 L-66,138" fill="none" stroke="#000" stroke-width="20" stroke-linecap="round"/>'
    '<path d="M-66,138 L-62,180" fill="none" stroke="#000" stroke-width="16" stroke-linecap="round"/>'
    '<path d="M47,92 L60,138" fill="none" stroke="#000" stroke-width="19" stroke-linecap="round"/>'
    '<path d="M60,138 L63,210" fill="none" stroke="#000" stroke-width="15" stroke-linecap="round"/>'
    '<circle cx="-70" cy="196" r="26"/>'
    '<path d="M-34,246 L-36,300" fill="none" stroke="#000" stroke-width="28" stroke-linecap="round"/>'
    '<path d="M-36,300 L-38,346" fill="none" stroke="#000" stroke-width="20" stroke-linecap="round"/>'
    '<path d="M34,246 L36,300" fill="none" stroke="#000" stroke-width="28" stroke-linecap="round"/>'
    '<path d="M36,300 L38,346" fill="none" stroke="#000" stroke-width="20" stroke-linecap="round"/>'
    '<ellipse cx="-42" cy="351" rx="17" ry="6"/>'
    '<ellipse cx="42" cy="351" rx="17" ry="6"/>'
)
# Hình người đang đá bên cuối tiêu đề (màu như chữ)
KICK = (
    '<g transform="translate(14 0)" fill="#fbf5a4" stroke="#fbf5a4" stroke-linecap="round" stroke-linejoin="round">'
    '<circle cx="1156" cy="792" r="12" stroke="none"/>'
    '<path d="M1150,808 L1136,864" fill="none" stroke-width="22"/>'
    '<path d="M1148,814 L1116,778" fill="none" stroke-width="10"/>'
    '<path d="M1150,818 L1184,836" fill="none" stroke-width="9"/>'
    '<path d="M1136,862 L1186,842 L1220,822" fill="none" stroke-width="14"/>'
    '<path d="M1136,862 L1128,912" fill="none" stroke-width="14"/>'
    '<ellipse cx="1122" cy="915" rx="13" ry="5" stroke="none"/>'
    "</g>"
)

BASE_DIR = Path(__file__).resolve().parent
COMPONENT_DIR = BASE_DIR / "wcf_menu"

PAGE_CSS = """
header, footer, #MainMenu, [data-testid="stHeader"], [data-testid="stToolbar"],
[data-testid="stDecoration"], [data-testid="stStatusWidget"], [data-testid="stBottom"] {
  display: none !important;
}
html, body, .stApp { background: #000 !important; }
[data-testid="stAppViewContainer"], [data-testid="stMain"], .main { overflow: hidden !important; }
.block-container, [data-testid="stMainBlockContainer"] { padding: 0 !important; max-width: 100% !important; }
[data-testid="stVerticalBlock"] { gap: 0 !important; }
"""

HTML_TEMPLATE = """<!doctype html>
<html lang="vi">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>__TITLE__</title>
<style>
@import url('https://fonts.googleapis.com/css2?family=__FONT_URL__&display=swap');
:root {
  --u: calc(min(100vw, 133.333vh) / 1448);  /* 1 đơn vị = 1 px của ảnh gốc */
  --text: __TEXT__;
  --hover: __HOVER__;
  --ink: __INK__;
}
* { box-sizing: border-box; }
html, body { margin: 0; width: 100%; height: 100%; overflow: hidden; background: #000; }
body { display: flex; align-items: center; justify-content: center; }
#stage { position: relative; flex: none; user-select: none; -webkit-user-select: none;
         width: calc(var(--u) * 1448); height: calc(var(--u) * 1086); }
#stage > svg { position: absolute; inset: 0; width: 100%; height: 100%; display: block; }
.lbl { font-family: '__FONT__', Impact, 'Arial Black', sans-serif; font-weight: 700; fill: var(--text);
       transition: fill .15s ease, transform .15s ease, filter .15s ease; animation: fade-in .8s ease both; }
.cr { font-family: 'Courier New', Courier, monospace; fill: var(--ink); font-size: 26px;
      animation: fade-in .8s ease both; }
.mark { fill: var(--hover); opacity: .35; transition: opacity .15s ease; }
@keyframes fade-in { from { opacity: 0; } to { opacity: 1; } }
.hit { position: absolute; appearance: none; margin: 0; padding: 0; border: 0; border-radius: 0;
       background: transparent; color: inherit; font: inherit; cursor: pointer; }
.hit:focus { outline: none; }
.hit:focus-visible { outline: 2px solid var(--hover); outline-offset: 2px; }

/* ----- Màn hình options: bảng, chữ và các ô chọn ----- */
.panel { fill: #0c0803; fill-opacity: .52; stroke: #fff5d2; stroke-opacity: .5; stroke-width: 1.5px; }
.ot { font-family: Arial, Helvetica, 'Liberation Sans', sans-serif; font-size: 21px; fill: #f4f4f4;
  filter: drop-shadow(0 0 2px rgba(0,0,0,.9)) drop-shadow(0 1px 1px rgba(0,0,0,.7)); }
.ot.b { font-weight: 700; }
.ot.k { font-weight: 700; fill: #f5ec9a; }
.ot.y { fill: #f5ec9a; }
.pt { font-family: '__FONT__', Impact, 'Arial Black', sans-serif; font-weight: 700; font-size: 36px; fill: #f6f1d6; }
.opt-html { position: absolute; inset: 0; pointer-events: none; }
.opt { position: absolute; pointer-events: auto; cursor: pointer; margin: 0; padding: 0; appearance: none;
       background: transparent; text-align: center; white-space: nowrap; color: #f4f4f4;
       font-family: Arial, Helvetica, 'Liberation Sans', sans-serif; font-size: calc(var(--u) * 21);
       border: calc(var(--u) * 1.5) solid transparent; border-radius: 0; }
.opt:hover { color: var(--hover); }
.opt.sel { border-color: #f4efd6; color: #f7e98a; }
.opt:focus { outline: none; }
.opt:focus-visible { outline: 2px solid var(--hover); outline-offset: 2px; }
.v-opt { animation: fade-in .5s ease both; }

/* ----- Trạng thái: menu / màn hình con / options ----- */
#stage[data-view="menu"] :is(.v-sub, .v-opt, .v-choose, .v-diff, .v-fight),
#stage[data-view="sub"] :is(.v-menu, .v-opt, .v-choose, .v-diff, .v-fight),
#stage[data-view="opt"] :is(.v-menu, .v-sub, .t-main, .v-choose, .v-diff, .v-fight),
#stage[data-view="choose"] :is(.v-menu, .v-sub, .v-opt, .t-main, .v-diff, .v-fight),
#stage[data-view="diff"] :is(.v-menu, .v-sub, .v-opt, .t-main, .v-choose, .v-fight),
#stage[data-view="fight"] :is(.v-menu, .v-sub, .v-opt, .t-main, .v-choose, .v-diff, .v-scene) { display: none; }

/* ----- Màn hình chọn nhân vật: lưới 7 × 5 chân dung và khung xem trước ----- */
.choose-html { position: absolute; inset: 0; pointer-events: none; }
.grid { position: absolute; left: calc(var(--u) * 28); top: calc(var(--u) * 36);
        width: calc(var(--u) * 672); height: calc(var(--u) * 484); padding: calc(var(--u) * 12);
        background: rgba(12, 8, 3, .5); box-shadow: inset 0 0 0 calc(var(--u) * 1.5) rgba(255, 245, 210, .5);
        display: grid; grid-template-columns: repeat(7, calc(var(--u) * 84));
        grid-template-rows: repeat(5, calc(var(--u) * 84)); gap: calc(var(--u) * 10); }
.tile { appearance: none; margin: 0; padding: calc(var(--u) * 4); pointer-events: auto; cursor: pointer;
        border: calc(var(--u) * 2) solid #3a2208; border-radius: calc(var(--u) * 4);
        background: linear-gradient(#f5cd7a, #c98a2e); }
.tile svg { display: block; width: 100%; height: 100%; }
.tile:hover { border-color: var(--hover); }
.tile.sel { border-color: #fff3b0; box-shadow: 0 0 calc(var(--u) * 10) rgba(255, 220, 120, .85); }
.tile:focus { outline: none; }
.tile:focus-visible { outline: 2px solid var(--hover); outline-offset: 2px; }
.preview { position: absolute; left: calc(var(--u) * 720); top: calc(var(--u) * 36);
           width: calc(var(--u) * 690); height: calc(var(--u) * 484);
           display: flex; flex-direction: column; align-items: center; justify-content: center;
           gap: calc(var(--u) * 18); background: rgba(12, 8, 3, .6);
           border: calc(var(--u) * 1.5) solid rgba(255, 245, 210, .5); }
.pv-art { width: calc(var(--u) * 340); height: calc(var(--u) * 340); }
.pv-art svg { display: block; width: 100%; height: 100%; }
.pv-name { font-family: '__FONT__', Impact, 'Arial Black', sans-serif; font-weight: 700;
           font-size: calc(var(--u) * 40); color: #f6f1d6; }
.pv-team { font-family: Arial, Helvetica, 'Liberation Sans', sans-serif; font-weight: 700;
           font-size: calc(var(--u) * 26); color: #ffd54f; }
.v-choose { animation: fade-in .5s ease both; }

/* ----- Màn chọn độ khó: ba nút EASY, HARD, IMPOSSIBLE ----- */
.diff-html { position: absolute; inset: 0; pointer-events: none; }
.diff-btn { position: absolute; left: calc(var(--u) * 28); width: calc(var(--u) * 672); height: calc(var(--u) * 146);
            appearance: none; margin: 0; padding: 0; border: 0; border-radius: 0; cursor: pointer; pointer-events: auto;
            display: flex; flex-direction: column; align-items: center; justify-content: center;
            gap: calc(var(--u) * 10); background: rgba(12, 8, 3, .5);
            box-shadow: inset 0 0 0 calc(var(--u) * 1.5) rgba(255, 245, 210, .5); }
.diff-easy { top: calc(var(--u) * 36); }
.diff-hard { top: calc(var(--u) * 205); }
.diff-impossible { top: calc(var(--u) * 374); }
.diff-btn:hover { background: rgba(40, 24, 8, .7); box-shadow: inset 0 0 0 calc(var(--u) * 2.5) var(--hover); }
.diff-btn:focus { outline: none; }
.diff-btn:focus-visible { outline: 2px solid var(--hover); outline-offset: 2px; }
.diff-btn .dl { font-family: '__FONT__', Impact, 'Arial Black', sans-serif; font-weight: 700;
                font-size: calc(var(--u) * 60); line-height: 1; color: #f6f1d6; }
.diff-hard .dl { color: #ffb43c; }
.diff-impossible .dl { color: #ff5a4a; }
.diff-btn .dc { font-family: Arial, Helvetica, 'Liberation Sans', sans-serif; font-size: calc(var(--u) * 22); color: #f4f4f4; }
.v-diff { animation: fade-in .5s ease both; }

/* ----- Màn đánh nhau: nền rừng, hai nhân vật, thanh tên và thanh nút dưới ----- */
.v-fight { animation: fade-in .4s ease both; }
.pause-layer { display: none; }
#stage.paused .pause-layer { display: inline; }
.fight-html { position: absolute; inset: 0; pointer-events: none; }
.fight-btn { position: absolute; appearance: none; margin: 0; padding: 0; border: 0; background: transparent;
             cursor: pointer; pointer-events: auto; }
.fight-btn:focus { outline: none; }
.fight-btn:focus-visible { outline: 2px solid var(--hover); outline-offset: 2px; }

/* Rê chuột hoặc chọn bằng Tab thì mục đó đổi màu */
__HOVER_RULES__
@media (prefers-reduced-motion: reduce) { .lbl, .cr, .v-opt { animation: none; } }
</style>
</head>
<body>
<div id="stage" data-screen="menu" data-view="menu">
<svg viewBox="0 0 1448 1086" xmlns="http://www.w3.org/2000/svg" aria-hidden="true">__SVG__</svg>
__OPT_HTML__
__CHOOSE_HTML__
__DIFF_HTML__
__FIGHT_HTML__
__HITS__
</div>
__MUSIC_TAG__
<script>
var SFX = __SFX_CONFIG__;

/* ---- Âm thanh tự tổng hợp bằng Web Audio (không dùng file ghi âm) ---- */
var sfxCtx = null, sfxOut = null;

function sfxReady() {
  if (!sfxCtx) {
    var AC = window.AudioContext || window.webkitAudioContext;
    if (!AC) { return null; }
    try { sfxCtx = new AC(); } catch (e) { return null; }
    var comp = sfxCtx.createDynamicsCompressor();
    sfxOut = sfxCtx.createGain();
    sfxOut.gain.value = SFX.volume;
    sfxOut.connect(comp);
    comp.connect(sfxCtx.destination);
  }
  if (sfxCtx.state === "suspended") { sfxCtx.resume().catch(function () {}); }
  return sfxCtx;
}

function sfxPlay(name) {
  if (!SFX.on) { return; }
  var ctx = sfxReady();
  if (!ctx) { return; }
  var t = ctx.currentTime + 0.02;
  if (name === "drum") { sfxDrum(ctx, sfxOut, t); }
  else if (name === "gong") { sfxGong(ctx, sfxOut, t); }
  else { sfxClick(ctx, sfxOut, t); }
}

function sfxNoise(ctx, seconds) {
  var buf = ctx.createBuffer(1, Math.floor(ctx.sampleRate * seconds), ctx.sampleRate);
  var d = buf.getChannelData(0);
  for (var i = 0; i < d.length; i++) { d[i] = Math.random() * 2 - 1; }
  return buf;
}

/* Tiếng click ngắn */
function sfxClick(ctx, out, t) {
  var osc = ctx.createOscillator();
  var g = ctx.createGain();
  osc.type = "triangle";
  osc.frequency.setValueAtTime(1500, t);
  osc.frequency.exponentialRampToValueAtTime(700, t + 0.035);
  g.gain.setValueAtTime(0.5, t);
  g.gain.exponentialRampToValueAtTime(0.0001, t + 0.06);
  osc.connect(g);
  g.connect(out);
  osc.start(t);
  osc.stop(t + 0.07);
}

/* Một tiếng trống: thân trống (sin, tần số giảm dần) + tiếng dùi đập */
function sfxDrumHit(ctx, out, t, pitch, level) {
  var osc = ctx.createOscillator();
  var g = ctx.createGain();
  osc.type = "sine";
  osc.frequency.setValueAtTime(pitch * 1.9, t);
  osc.frequency.exponentialRampToValueAtTime(pitch, t + 0.07);
  g.gain.setValueAtTime(level, t);
  g.gain.exponentialRampToValueAtTime(0.0001, t + 0.42);
  osc.connect(g);
  g.connect(out);
  osc.start(t);
  osc.stop(t + 0.45);

  var src = ctx.createBufferSource();
  var bp = ctx.createBiquadFilter();
  var ng = ctx.createGain();
  src.buffer = sfxNoise(ctx, 0.05);
  bp.type = "bandpass";
  bp.frequency.value = 1600;
  bp.Q.value = 0.9;
  ng.gain.setValueAtTime(level * 0.5, t);
  ng.gain.exponentialRampToValueAtTime(0.0001, t + 0.025);
  src.connect(bp);
  bp.connect(ng);
  ng.connect(out);
  src.start(t);
  src.stop(t + 0.03);
}

/* Trống "tùng tùng": hai tiếng liền nhau, tiếng sau trầm hơn một chút */
function sfxDrum(ctx, out, t) {
  sfxDrumHit(ctx, out, t, 110, 0.9);
  sfxDrumHit(ctx, out, t + 0.3, 92, 0.8);
}

/* Tiếng chiêng: nhiều âm không hài hoà, ngân lâu, có độ rung nhẹ */
function sfxGong(ctx, out, t) {
  var f0 = 84;
  var partials = [  /* [tỉ lệ tần số, thời gian ngân (giây), độ to] */
    [1.0, 5.0, 0.9], [1.51, 4.2, 0.7], [2.02, 3.4, 0.55],
    [2.66, 2.4, 0.35], [3.13, 1.7, 0.25], [4.19, 1.1, 0.15]
  ];
  for (var i = 0; i < partials.length; i++) {
    var p = partials[i];
    for (var k = 0; k < 2; k++) {
      var osc = ctx.createOscillator();
      var g = ctx.createGain();
      osc.type = "sine";
      osc.frequency.value = f0 * p[0] + (k === 1 ? 1.2 : 0);  /* lệch 1,2 Hz tạo độ rung */
      g.gain.setValueAtTime(0.0001, t);
      g.gain.linearRampToValueAtTime(p[2] * 0.16, t + 0.015);
      g.gain.setTargetAtTime(0, t + 0.015, p[1] / 6);  /* ngân dần, gần hết sau p[1] giây */
      osc.connect(g);
      g.connect(out);
      osc.start(t);
      osc.stop(t + p[1] + 0.3);
    }
  }
  var src = ctx.createBufferSource();
  var bp = ctx.createBiquadFilter();
  var ng = ctx.createGain();
  src.buffer = sfxNoise(ctx, 0.08);
  bp.type = "bandpass";
  bp.frequency.value = 700;
  bp.Q.value = 0.7;
  ng.gain.setValueAtTime(0.45, t);
  ng.gain.exponentialRampToValueAtTime(0.0001, t + 0.09);
  src.connect(bp);
  bp.connect(ng);
  ng.connect(out);
  src.start(t);
  src.stop(t + 0.1);
}

(function () {
  var stage = document.getElementById("stage");
  var heading = document.getElementById("heading");
  var tiles = stage.querySelectorAll("[data-pick]");

  /* Chọn một nhân vật: đánh dấu ô rồi chuyển sang chọn độ khó */
  var selectedTile = null, selectedId = null;

  /* Đưa chân dung của một ô vào khung xem trước (màn chọn nhân vật) */
  function previewTile(tile) {
    var markup = tile.querySelector("svg").outerHTML;
    document.getElementById("pvArt").innerHTML = markup.split('id="kit-').join('id="pv-kit-').split('url(#kit-').join('url(#pv-kit-');
    document.getElementById("pvName").textContent = tile.getAttribute("aria-label");
    document.getElementById("pvTeam").textContent = tile.getAttribute("data-team");
  }

  /* Đưa chân dung nhân vật đã chọn vào khung xem trước của màn chọn độ khó */
  function setDiffPreview(tile) {
    var markup = tile.querySelector("svg").outerHTML;
    document.getElementById("dfArt").innerHTML = markup.split('id="kit-').join('id="df-kit-').split('url(#kit-').join('url(#df-kit-');
    document.getElementById("dfName").textContent = tile.getAttribute("aria-label");
    document.getElementById("dfTeam").textContent = tile.getAttribute("data-team");
  }

  function selectTile(tile) {
    for (var i = 0; i < tiles.length; i++) { tiles[i].classList.remove("sel"); }
    tile.classList.add("sel");
    selectedTile = tile;
    selectedId = tile.getAttribute("data-pick");
    previewTile(tile);
    setDiffPreview(tile);
  }

  var ready = false;

  function send(type, data) {
    var msg = { isStreamlitMessage: true, type: type };
    for (var k in data) { msg[k] = data[k]; }
    window.parent.postMessage(msg, "*");
  }

  /* Chiều cao khung = tỉ lệ 4:3 theo chiều rộng, nhưng không vượt quá cửa sổ */
  function fitFrame() {
    var h = Math.round(window.innerWidth * 0.75);
    try {
      var limit = window.parent.innerHeight;
      if (limit) { h = Math.min(h, limit); }
    } catch (e) { /* không đọc được khung cha: giữ tỉ lệ 4:3 */ }
    send("streamlit:setFrameHeight", { height: Math.max(120, h) });
  }

  function showScreen(name) {
    var view = name === "menu" ? "menu" : (name === "options" ? "opt" : ((name === "single" || name === "choose") ? "choose" : (name === "diff" ? "diff" : (name === "fight" ? "fight" : "sub"))));
    if (view === "sub") {
      var src = document.getElementById("lbl-" + name);
      heading.textContent = src.textContent;
      heading.setAttribute("x", src.getAttribute("x"));
      heading.setAttribute("textLength", src.getAttribute("textLength"));
    }
    if (view === "diff" && selectedTile) { setDiffPreview(selectedTile); }
    stage.setAttribute("data-screen", name);
    stage.setAttribute("data-view", view);
  }

  /* Sau khi chọn độ khó: hiện màn đánh nhau với nhân vật đã chọn ở bên trái */
  function showFight(level) {
    stage.classList.remove("paused");
    document.getElementById("fbanner").textContent = level.toUpperCase();
    document.getElementById("fnameL").textContent = selectedTile ? selectedTile.getAttribute("aria-label") : "";
    var all = stage.querySelectorAll(".ft");
    for (var i = 0; i < all.length; i++) { all[i].style.display = "none"; }
    var mine = document.getElementById("ft-" + selectedId);
    if (mine) { mine.style.display = "inline"; }
    showScreen("fight");
  }

  stage.addEventListener("click", function (ev) {
    var pick = ev.target.closest("[data-pick]");
    if (pick) {  /* chọn nhân vật: gửi về Python rồi sang màn chọn độ khó */
      selectTile(pick);
      sfxPlay("click");
      send("streamlit:setComponentValue", { value: "pick:" + pick.getAttribute("data-pick"), dataType: "json" });
      showScreen("diff");
      return;
    }
    var diff = ev.target.closest("[data-diff]");
    if (diff && selectedId) {  /* chọn độ khó: gửi mã nhân vật và độ khó về Python */
      var level = diff.getAttribute("data-diff");
      sfxPlay(SFX.buttons[level] || "click");
      send("streamlit:setComponentValue", { value: "start:" + selectedId + ":" + level, dataType: "json" });
      showFight(level);
      return;
    }
    var fb = ev.target.closest("[data-fight]");
    if (fb) {  /* các nút dưới màn đánh nhau */
      var act = fb.getAttribute("data-fight");
      sfxPlay("click");
      if (act === "menu") {
        stage.classList.remove("paused");
        showScreen("menu");
        send("streamlit:setComponentValue", { value: "menu", dataType: "json" });
      } else if (act === "options") {
        showScreen("options");
      } else {
        var on = !stage.classList.contains("paused");
        stage.classList.toggle("paused", on);
        send("streamlit:setComponentValue", { value: on ? "pause" : "resume", dataType: "json" });
      }
      return;
    }
    var back = ev.target.closest("[data-back]");
    if (back) {  /* quay lại màn trước trong cùng giao diện, không gửi về Python */
      sfxPlay("click");
      showScreen(back.getAttribute("data-back"));
      return;
    }
    var opt = ev.target.closest("[data-group]");
    if (opt) {  /* chọn On/Off, Normal/More... chỉ đổi trên màn hình */
      var same = stage.querySelectorAll('[data-group="' + opt.getAttribute("data-group") + '"]');
      for (var i = 0; i < same.length; i++) { same[i].classList.remove("sel"); }
      opt.classList.add("sel");
      sfxPlay(SFX.option);
      return;
    }
    var btn = ev.target.closest("[data-go]");
    if (!btn) { return; }
    var target = btn.getAttribute("data-go");
    sfxPlay(SFX.buttons[target] || "click");
    showScreen(target);
    send("streamlit:setComponentValue", { value: target, dataType: "json" });
  });

  stage.addEventListener("mouseover", function (ev) {  /* rê chuột vào ô nào thì xem trước nhân vật đó */
    var t = ev.target.closest ? ev.target.closest("[data-pick]") : null;
    if (t && stage.getAttribute("data-view") === "choose") { previewTile(t); }
  });
  stage.addEventListener("focusin", function (ev) {
    var t = ev.target.closest ? ev.target.closest("[data-pick]") : null;
    if (t) { previewTile(t); }
  });
  window.addEventListener("message", function (e) {
    if (e.data && e.data.type === "streamlit:render") { ready = true; }
  });
  window.addEventListener("resize", fitFrame);
  window.addEventListener("load", fitFrame);
  send("streamlit:componentReady", { apiVersion: 1 });
  fitFrame();
  if (tiles.length) { previewTile(tiles[0]); }
  setTimeout(function () {
    if (!ready) { send("streamlit:componentReady", { apiVersion: 1 }); }
  }, 500);
  /* Nhạc nền: tải cả file rồi phát từ data-start, lặp lại đoạn đến data-end (nếu có) */
  var music = document.getElementById("bgm");
  if (music) {
    var startAt = parseFloat(music.getAttribute("data-start")) || 0;
    var endAttr = music.getAttribute("data-end");
    var endAt = endAttr ? parseFloat(endAttr) : null;
    var musicOn = false;
    music.volume = parseFloat(music.getAttribute("data-volume")) || 0.6;

    function playMusic() {
      if (musicOn) { return; }
      var p = music.play();
      if (p && p.then) {
        p.then(function () { musicOn = true; }, function () { /* bị chặn: thử lại ở lần bấm kế */ });
      } else {
        musicOn = true;
      }
    }
    function cue() { music.currentTime = startAt; playMusic(); }

    music.addEventListener("loadedmetadata", cue);
    music.addEventListener("timeupdate", function () {
      if (endAt !== null && music.currentTime >= endAt) { music.currentTime = startAt; }
    });
    music.addEventListener("ended", function () {
      music.currentTime = startAt;
      music.play().catch(function () {});
    });
    document.addEventListener("pointerdown", playMusic);
    document.addEventListener("keydown", playMusic);

    /* Tải hết file rồi mới gán cho trình phát: tua đến giây bắt đầu sẽ chính xác */
    fetch(music.getAttribute("data-src")).then(function (r) {
      return r.blob();
    }).then(function (blob) {
      music.src = URL.createObjectURL(blob);
    }).catch(function () { /* không tải được nhạc: bỏ qua */ });
  }
})();
</script>
<script>
/* =====================================================================
   ĐIỀU KHIỂN NHÂN VẬT P1 (theo ảnh "PLAYER 1 CONTROLS")
   ===================================================================== */
(function () {
  var NS = "http://www.w3.org/2000/svg";
  var GROUND = 880;   /* y mặt đất, khớp với translate(... 880) của nhân vật */
  var S = 1.2;        /* tỉ lệ nhân vật */
  var G = 2600;       /* trọng lực */
  var JUMP = 900;     /* vận tốc lúc nhảy */
  var WALK = 420;     /* tốc độ đi */

  var stage = document.getElementById("stage");
  var keys = {}, pressedAt = {};
  var pl = null, op = null, wpn = null, sword = null;
  var loopOn = false, last = 0;

  /* d: thời lượng đòn; w: khoảng thời gian có thể trúng; r: tầm đánh;
     dmg: sát thương; kd: làm đối thủ ngã */
  var MOVES = {
    punch:   { d: 0.28, w: [0.08, 0.18], r: 300, dmg: 6,  kd: false },
    kick:    { d: 0.42, w: [0.12, 0.26], r: 300, dmg: 9,  kd: false },
    special: { d: 0.55, w: [0.20, 0.35], r: 320, dmg: 22, kd: false },
    grab:    { d: 0.50, w: [0.15, 0.25], r: 260, dmg: 12, kd: true },
    sweep:   { d: 0.50, w: [0.20, 0.30], r: 230, dmg: 5,  kd: true },
    enemy:   { d: 0.32, w: [0.12, 0.22], r: 280, dmg: 7,  kd: false },
    slash:   { d: 0.45, w: [0.18, 0.30], r: 250, dmg: 11, kd: false },
    skill:   { d: 0.75, w: [0.40, 0.55], r: 340, dmg: 20, kd: false }
  };
  /* Độ khó của đối thủ:
     cd: nghỉ giữa hai đòn; mul: hệ số sát thương; speed: tốc độ tiến lại (0 = đứng yên như bao cát);
     moves: các đòn thường; skill: xác suất dùng Kiếm Lửa; block: xác suất đỡ khi người chơi ra đòn */
  var LEVEL = {
    EASY:       { cd: 2.4, mul: 0.6, speed: 0,   moves: ["punch", "slash"], skill: 0,    block: 0 },
    HARD:       { cd: 1.5, mul: 1,   speed: 200, moves: ["punch", "kick", "slash"], skill: 0.15, block: 0.25 },
    IMPOSSIBLE: { cd: 0.8, mul: 1.4, speed: 360, moves: ["punch", "kick", "slash", "slash"], skill: 0.35, block: 0.5 }
  };

  function svgEl(tag, attrs) {
    var e = document.createElementNS(NS, tag);
    for (var k in attrs) { e.setAttribute(k, attrs[k]); }
    return e;
  }

  /* Thanh kiếm: hilt ở gốc, lưỡi hướng về +x */
  function getSword() {
    if (sword) { return; }
    sword = svgEl("g", { id: "wcf-sword", "class": "v-fight" });
    sword.appendChild(svgEl("rect", { x: -22, y: -3, width: 22, height: 6, fill: "#5a3418" }));
    sword.appendChild(svgEl("rect", { x: -3, y: -11, width: 6, height: 22, fill: "#c9a227" }));
    sword.appendChild(svgEl("path", { d: "M3,-4 L100,-4 L118,0 L100,4 L3,4 Z",
      fill: "#dfe6ea", stroke: "#37474f", "stroke-width": 1.2 }));
    stage.querySelector("svg").appendChild(sword);
  }

  function makeFighter(x, dir, root) {
    return {
      root: root, x: x, h: 0, vy: 0, dir: dir, air: false, walk: false, ph: 0,
      hp: 100, act: null, kd: 0, blocking: false, flash: 0, mul: 1, A: [0, 0, 0, 0],
      af: root.querySelector(".arm-f"), ab: root.querySelector(".arm-b"),
      lf: root.querySelector(".leg-f"), lb: root.querySelector(".leg-b"),
      osw: root.querySelector(".osw"), blade: root.querySelector(".osw-blade"), lvl: null, blockT: 0
    };
  }

  function start(f, name) {
    var m = MOVES[name];
    f.act = { n: name, t: 0, d: m.d, w: m.w, r: m.r, dmg: m.dmg * f.mul, kd: m.kd, hit: false };
  }

  function hurt(f, dmg, kd) {
    f.hp = Math.max(0, f.hp - dmg);
    f.flash = 0.12;
    if (kd) { f.kd = Math.max(f.kd, 0.9); f.act = null; }
    if (f === op) { op.x = Math.min(1340, op.x + pl.dir * 18); }
    sfxPlay(dmg >= 12 ? "drum" : "click");
  }

  /* Diễn tiến một đòn: đúng khung thời gian và đủ gần thì gây sát thương */
  function step(f, foe, dt) {
    var a = f.act;
    if (!a) { return; }
    a.t += dt;
    if (!a.hit && a.t >= a.w[0] && a.t <= a.w[1] && foe.kd <= 0 && foe.hp > 0 &&
        Math.abs(foe.x - f.x) <= a.r) {
      a.hit = true;
      hurt(foe, a.dmg * (foe.blocking ? 0.2 : 1), a.kd);
    }
    if (a.t >= a.d) { f.act = null; }
  }

  function pickup() {
    if (wpn.s === "ground" && Math.abs(wpn.x - pl.x) <= 130) { wpn.s = "hand"; }
    else if (wpn.s === "hand") { wpn.s = "ground"; wpn.h = 0; wpn.x = pl.x + pl.dir * 40; }
  }

  function throwOrDraw() {
    if (wpn.s === "back") { wpn.s = "hand"; }
    else if (wpn.s === "hand") {
      wpn.s = "fly"; wpn.x = pl.x + pl.dir * 190; wpn.h = 367 + pl.h;
      wpn.vx = pl.dir * 900; wpn.vy = 180; wpn.rot = 0;
    }
  }

  function updateWeapon(dt) {
    var w = wpn, o = op;
    if (w.s !== "fly") { return; }
    w.x += w.vx * dt;
    w.h += w.vy * dt;
    w.vy -= G * dt;
    w.rot += 900 * dt;
    if (o.kd <= 0 && o.hp > 0 && Math.abs(o.x - w.x) < 80 && w.h < 200) {
      hurt(o, 22, true);
      w.s = "ground"; w.h = 0;
    } else if (w.h <= 0 || w.x < 40 || w.x > 1408) {
      w.s = "ground"; w.h = 0; w.x = Math.max(40, Math.min(1408, w.x));
    }
  }

  /* Chọn đòn tiếp theo của đối thủ theo cấp độ */
  function pickMove(L) {
    if (Math.random() < L.skill) { return "skill"; }
    return L.moves[Math.floor(Math.random() * L.moves.length)];
  }

  /* Đối thủ theo cấp độ: tiến lại gần, đỡ đòn, dùng kiếm và Kiếm Lửa */
  function updateEnemy(dt) {
    var o = op, p = pl, L = o.lvl;
    o.dir = p.x < o.x ? -1 : 1;
    o.walk = false;
    o.blockT = Math.max(0, o.blockT - dt);
    if (p.act && !p.act.seen) {        /* mỗi đòn của người chơi chỉ được xét đỡ một lần */
      p.act.seen = true;
      if (!o.act && Math.random() < L.block) { o.blockT = 0.45; }
    }
    o.blocking = o.blockT > 0 && !o.act;
    if (o.kd > 0) {
      o.kd -= dt;
      o.act = null;
      o.flash = Math.max(0, o.flash - dt);
      return;
    }
    var gap = Math.abs(p.x - o.x);
    if (L.speed > 0) {                 /* cấp độ có tiến lại gần */
      if (!o.act && !o.blocking && gap > 170) {
        o.x += (p.x < o.x ? -1 : 1) * L.speed * dt;
        o.walk = true;
        o.ph += dt * 12;
      }
      o.x = Math.max(p.x + 70, Math.min(1380, o.x));
    } else if (!o.act) {               /* bao cát: trở về vị trí cũ sau khi bị đẩy */
      o.x += (1090 - o.x) * Math.min(1, dt * 2.5);
    }
    o.cd -= dt;
    if (!o.act && !o.blocking && p.hp > 0 && o.cd <= 0) {
      var mv = pickMove(L);
      if (gap <= MOVES[mv].r) {        /* chỉ ra đòn khi đủ gần */
        start(o, mv);
        o.cd = L.cd * (0.7 + Math.random() * 0.7);
      }
    }
    step(o, p, dt);
    o.flash = Math.max(0, o.flash - dt);
  }

  function update(dt) {
    var p = pl, o = op;
    p.dir = o.x >= p.x ? 1 : -1;
    p.blocking = !!keys.s && !p.act && !p.air;
    p.walk = false;
    if (!p.act && !keys.s) {
      var mv = (keys.d ? 1 : 0) - (keys.a ? 1 : 0);
      if (mv) { p.x += mv * WALK * dt; p.walk = !p.air; }
    }
    p.x = Math.max(120, Math.min(o.x - 70, p.x));
    if (p.walk) { p.ph += dt * 12; }
    if (p.air) {
      p.vy -= G * dt;
      p.h += p.vy * dt;
      if (p.h <= 0) { p.h = 0; p.vy = 0; p.air = false; }
    }
    if (p.hp <= 0) { document.getElementById("ko").style.display = "inline"; return; }
    if (o.hp <= 0) {  /* đối thủ hết máu: giữ đối thủ nằm xuống và hiện YOU WIN */
      o.kd = 99;
      document.getElementById("win").style.display = "inline";
      return;
    }
    step(p, o, dt);
    p.flash = Math.max(0, p.flash - dt);
    updateEnemy(dt);
    updateWeapon(dt);
  }

  /* Đặt tư thế tay, chân, độ nghiêng và hiệu ứng cho một nhân vật */
  function poseFighter(f) {
    var a = f.act, n = a ? a.n : "", k = a ? Math.sin(Math.PI * Math.min(1, a.t / a.d)) : 0;
    var A = [0, 0, 0, 0], L = [0, 0], lean = 0, glow = false;
    if (f.kd > 0) { lean = 90; }
    else if (n === "punch" || n === "enemy") { A[0] = 70 * k; }
    else if (n === "kick") { L[0] = -74 * k; }
    else if (n === "special") { A[0] = 60 * k; L[0] = -74 * k; glow = "#ffd54f"; }
    else if (n === "slash") { A[0] = 40 * k; }
    else if (n === "skill") { A[0] = 50 * k; L[0] = -30 * k; glow = "#ff5a4a"; }
    else if (n === "grab") { A[0] = 40 * k; A[2] = 40 * k; }
    else if (n === "sweep") { L[0] = -24 * k; lean = 20 * k; }
    else if (f.blocking) { A[1] = -55; A[3] = -60; }
    else if (f.air) { L[0] = -30; L[1] = 15; }
    else if (f.walk) { var s = Math.sin(f.ph) * 18; L[0] = s; L[1] = -s; }
    f.A = A;
    f.root.setAttribute("transform", "translate(" + f.x + " " + (GROUND - f.h) + ") scale(" +
      (S * f.dir) + " " + S + ") rotate(" + lean + ")");
    f.af.setAttribute("transform", "translate(" + A[0] + " 0) rotate(" + A[1] + " 48 -318)");
    f.ab.setAttribute("transform", "translate(" + A[2] + " 0) rotate(" + A[3] + " -46 -318)");
    f.lf.setAttribute("transform", "rotate(" + L[0] + " 18 -196)");
    f.lb.setAttribute("transform", "rotate(" + L[1] + " -16 -196)");
    f.root.style.filter = glow ? "drop-shadow(0 0 14px " + glow + ")" : (f.flash > 0 ? "brightness(1.9)" : "");
  }

  /* Đặt kiếm theo trạng thái: đeo sau lưng, cầm trên tay, nằm đất, đang bay */
  function placeSword() {
    var w = wpn, p = pl, t;
    if (w.s === "back") {
      t = "translate(" + (p.x - 48 * p.dir) + " " + (GROUND - p.h - 384) + ") rotate(" +
        (p.dir > 0 ? -135 : -45) + ") scale(1.5)";
    } else if (w.s === "hand") {
      var fr = p.A[1] * Math.PI / 180, c = Math.cos(fr), sn = Math.sin(fr);
      var ax = 112 * c - 12 * sn, ay = 112 * sn + 12 * c;   /* vector từ vai đến tay */
      var th = Math.atan2(ay, ax) * 180 / Math.PI;
      var ang = p.dir > 0 ? th - 14 : 194 - th;
      t = "translate(" + (p.x + p.dir * S * (48 + ax + p.A[0])) + " " +
        (GROUND - p.h + S * (ay - 318)) + ") rotate(" + ang + ") scale(1.5)";
    } else if (w.s === "ground") {
      t = "translate(" + w.x + " " + (GROUND - 3) + ") rotate(-3) scale(1.5)";
    } else {
      t = "translate(" + w.x + " " + (GROUND - w.h) + ") rotate(" + w.rot + ") scale(1.5)";
    }
    sword.setAttribute("transform", t);
  }

  /* Kiếm của đối thủ: nằm trong tay, vung khi ra đòn, giơ lên khi đỡ, đổi màu khi dùng Kiếm Lửa */
  function poseOppSword(f) {
    var a = f.act, n = a ? a.n : "", ang = -55;
    if (n === "slash" || n === "skill") { ang = -95 + 135 * Math.min(1, a.t / a.d); }
    else if (f.blocking) { ang = -100; }
    f.osw.setAttribute("transform", "translate(160 -306) rotate(" + ang + ") scale(1.25)");
    f.blade.setAttribute("fill", n === "skill" ? "#ff8a65" : "#dfe6ea");
  }

  function render() {
    if (!pl) { return; }
    poseFighter(pl);
    poseFighter(op);
    if (op.osw) { poseOppSword(op); }
    placeSword();
    document.getElementById("hpP").setAttribute("width", (670 * pl.hp / 100).toFixed(1));
    var w = 670 * op.hp / 100, hb = document.getElementById("hpB");
    hb.setAttribute("width", w.toFixed(1));
    hb.setAttribute("x", (1428 - w).toFixed(1));
  }

  function frame(ts) {
    if (stage.getAttribute("data-view") !== "fight") { loopOn = false; return; }
    var dt = Math.max(0, Math.min(0.033, (ts - last) / 1000));
    last = ts;
    if (!stage.classList.contains("paused")) { update(dt); }
    render();
    requestAnimationFrame(frame);
  }

  function startFight() {
    var root = null, roots = stage.querySelectorAll(".ft");
    for (var i = 0; i < roots.length; i++) {
      if (roots[i].style.display === "inline") { root = roots[i]; }
    }
    var foeRoot = document.getElementById("ft-opp");
    if (!root || !foeRoot) { return; }
    if (!root.querySelector(".arm-f") || !foeRoot.querySelector(".leg-b")) {
      console.error("Thiếu nhóm arm-f/arm-b/leg-f/leg-b: hãy sửa fighter_parts trước.");
      return;
    }
    var lvl = LEVEL[document.getElementById("fbanner").textContent.trim()] || LEVEL.HARD;
    pl = makeFighter(360, 1, root);
    op = makeFighter(1090, -1, foeRoot);
    op.lvl = lvl;
    op.mul = lvl.mul;
    op.cd = lvl.cd;
    wpn = { s: "back", x: 0, h: 0, vx: 0, vy: 0, rot: 0 };
    getSword();
    document.getElementById("ko").style.display = "none";
    document.getElementById("win").style.display = "none";
    keys = {};
    last = performance.now();
    if (!loopOn) { loopOn = true; requestAnimationFrame(frame); }
  }

  function onKey(e, down) {
    var k = (e.key || "").toLowerCase();
    if (k.length !== 1 || "adwstyu".indexOf(k) < 0) { return; }
    keys[k] = down;
    if (!down || e.repeat || !pl || stage.getAttribute("data-view") !== "fight") { return; }
    if (pl.hp <= 0 || stage.classList.contains("paused")) { return; }
    var now = performance.now();
    pressedAt[k] = now;
    var held = function (x) { return keys[x] || now - (pressedAt[x] || 0) < 120; };
    if (k === "w") {
      if (!pl.air && !pl.act) { pl.air = true; pl.vy = JUMP; }
    } else if (k === "t") {
      start(pl, held("s") ? "grab" : (held("y") ? "special" : "punch"));
    } else if (k === "y") {
      start(pl, held("s") ? "sweep" : (held("t") ? "special" : "kick"));
    } else if (k === "u") {
      if (held("s")) { pickup(); } else { throwOrDraw(); }
    } else if (k === "s") {
      if (held("t")) { start(pl, "grab"); }
      else if (held("y")) { start(pl, "sweep"); }
      else if (held("u")) { pickup(); }
    }
  }

  document.addEventListener("keydown", function (e) { onKey(e, true); });
  document.addEventListener("keyup", function (e) { onKey(e, false); });
  window.addEventListener("blur", function () { keys = {}; });
  new MutationObserver(function () {
    if (stage.getAttribute("data-view") === "fight") { startFight(); }
  }).observe(stage, { attributes: true, attributeFilter: ["data-view"] });
})();
</script>
</body>
</html>
"""


def esc(s: str) -> str:
    return html.escape(s, quote=True)


def scene_svg() -> str:
    """Bầu trời, mặt trời phát sáng, mây, đồi và hai nhân vật đứng."""
    clouds = "".join(f'<path d="{d}"/>' for d in CLOUDS)
    defs = (
        "<defs>"
        '<linearGradient id="sky" x1="0" y1="0" x2="0" y2="1">'
        '<stop offset="0" stop-color="#e59226"/>'
        '<stop offset=".45" stop-color="#f3a53d"/>'
        '<stop offset=".8" stop-color="#f9b955"/>'
        '<stop offset="1" stop-color="#fcc866"/>'
        "</linearGradient>"
        '<radialGradient id="glow" cx="900" cy="450" r="500" gradientUnits="userSpaceOnUse">'
        '<stop offset="0" stop-color="#fffdf2"/>'
        '<stop offset=".18" stop-color="#fffbd0"/>'
        '<stop offset=".38" stop-color="#ffe98a" stop-opacity=".95"/>'
        '<stop offset=".62" stop-color="#ffc85a" stop-opacity=".55"/>'
        '<stop offset="1" stop-color="#ffb040" stop-opacity="0"/>'
        "</radialGradient>"
        "</defs>"
    )
    return (
        defs
        + f'<rect width="{W}" height="{H}" fill="url(#sky)"/>'
        + f'<rect width="{W}" height="{H}" fill="url(#glow)"/>'
        + f'<g fill="#6b4a1a" opacity=".46">{clouds}</g>'
        + f'<path d="{HILL}" fill="#000"/>'
        + '<g fill="#000">'
        + f'<g transform="translate(620 160) scale(-.88 .88)">{ATHLETE}</g>'
        + f'<g transform="translate(819 140)">{ATHLETE}</g>'
        + "</g>"
    )


def text_el(text, x, y, size, width, cls="lbl", uid=None, delay=0.0) -> str:
    """Chữ SVG có độ dài cố định, giữ đúng độ dài như ảnh gốc dù font khác nhau."""
    id_attr = f' id="{uid}"' if uid else ""
    return (
        f'<text class="{cls}"{id_attr} x="{x}" y="{y}" font-size="{size}" '
        f'textLength="{width}" lengthAdjust="spacingAndGlyphs" '
        f'style="animation-delay:{delay:.2f}s">{esc(text)}</text>'
    )


def marker(x, base, key, pointing) -> str:
    """Mũi tên nhỏ đứng cạnh mục menu, sáng lên khi rê chuột."""
    if pointing == "right":
        d = f"M{x},{base - 27} L{x},{base - 7} L{x + 14},{base - 17} Z"
    else:
        d = f"M{x + 14},{base - 27} L{x + 14},{base - 7} L{x},{base - 17} Z"
    return f'<path class="mark" id="mark-{key}" d="{d}"/>'


def ot(x, y, s, cls="", anchor="start", size=21) -> str:
    """Chữ thường ở màn hình options (Arial, không ép độ dài)."""
    extra = f' text-anchor="{anchor}"' if anchor != "start" else ""
    return f'<text class="ot {cls}" x="{x}" y="{y}" font-size="{size}"{extra}>{esc(s)}</text>'


def pt(x, y, s, width) -> str:
    """Tiêu đề bảng kiểu chữ cọ, ép đúng độ dài như ảnh."""
    return (f'<text class="pt" x="{x}" y="{y}" textLength="{width}" '
            f'lengthAdjust="spacingAndGlyphs">{esc(s)}</text>')


def controls(title, title_x, title_w, key_x, label_x, center_x, rows, moves) -> list:
    out = [pt(title_x, 80, title, title_w)]
    for key, label, base in rows + moves:
        out.append(ot(key_x, base, key, "k"))
        out.append(ot(label_x, base, label))
    for line, base in zip(PARA, PARA_Y):
        out.append(ot(center_x, base, line, "b", "middle", 20))
    return out


def options_svg() -> str:
    """Nền, bảng, chữ và phần chân trang của màn hình options (phần SVG)."""
    out = [
        '<g class="v-opt">',
        f'<rect x="0" y="1032" width="{W}" height="54" fill="#6e0c0c"/>',
    ]
    out += [f'<rect class="panel" x="{x}" y="{y}" width="{w}" height="{h}"/>'
            for x, y, w, h in OPT_PANELS]
    out.append(pt(173, 80, "OPTIONS", 100))
    out.append(ot(173, 124, "Graphics / Performance Options", "b"))
    out += [ot(173, base, label) for label, base, _items in OPT_ROWS]
    out.append(ot(173, 358, "Sound Options", "b"))
    out.append(ot(342, 532, "Click here to clear saved data", "y", "middle", 20))
    out += controls("PLAYER 1 CONTROLS", 570, 252, 571, 644, 740, P1_ROWS, P1_MOVES)
    out += controls("PLAYER 2 CONTROLS", 968, 262, 968, 1095, 1137, P2_ROWS, P2_MOVES)
    out.append(KICK)
    # Khoảng cách các dòng chữ dưới cùng đủ rộng cho font chữ cao (ví dụ Impact)
    out.append(text_el("< BACK TO MAIN MENU", 30, 752, 56, 515, uid="lbl-optback"))
    out.append(text_el(GAME_TITLE, 28, 893, 140, 1087))
    out.append(text_el(SUBTITLE, 30, 958, 50, 490))
    out.append(text_el(SITE_URL, 30, 1018, 50, 642))
    out.append("</g>")
    return "".join(out)


def options_html() -> str:
    """Các ô chọn On/Off, Normal/More... (HTML để bấm được)."""
    out = ['<div class="opt-html v-opt">']
    for _label, base, items in OPT_ROWS:
        for group, val, text, x, w, sel in items:
            cls = "opt sel" if sel else "opt"
            out.append(
                f'<button type="button" class="{cls}" data-group="{group}" data-val="{val}" '
                f'style="left:calc(var(--u) * {x});top:calc(var(--u) * {base - 22});'
                f'width:calc(var(--u) * {w});height:calc(var(--u) * 31);'
                f'line-height:calc(var(--u) * 31)">{esc(text)}</button>'
            )
    out.append("</div>")
    return "".join(out)


def svg_body() -> str:
    first_label, _, fx, fw, fb = MENU[0]
    back_label, _, bx, bw, bb = BACK
    main = (
        f'<text class="cr t-main" x="12" y="26">{esc(CREDIT_LEFT)}</text>'
        f'<text class="cr t-main" x="{W - 10}" y="26" text-anchor="end">{esc(CREDIT_RIGHT)}</text>'
        + text_el(GAME_TITLE, 28, 900, 124, 960, cls="lbl t-main title")
        + text_el(SUBTITLE, 30, 975, 56, 490, cls="lbl t-main", delay=0.1)
        + text_el(SITE_URL, 30, 1057, 56, 705, cls="lbl t-main", delay=0.1)
    )
    menu = "".join(
        marker(x - 44, base, key, "right")
        + text_el(label, x, base, 50, w, uid=f"lbl-{key}", delay=0.2 + 0.1 * i)
        for i, (label, key, x, w, base) in enumerate(MENU)
    )
    sub = (
        text_el(first_label, fx, fb, 50, fw, uid="heading")
        + text_el("COMING SOON", MENU_RIGHT - 302, 892, 50, 302, uid="soon", delay=0.2)
        + marker(bx - 44, bb, "back", "left")
        + text_el(back_label, bx, bb, 50, bw, uid="lbl-back", delay=0.3)
    )
    return (
        '<g class="v-scene">' + scene_svg() + "</g>"
        + main
        + f'<g class="menu-view v-menu">{menu}</g>'
        + f'<g class="sub-view v-sub">{sub}</g>'
        + options_svg()
        + choose_svg()
        + diff_svg()
        + fight_svg()
    )


def box(x, y, w, h) -> str:
    return (f"left:calc(var(--u) * {x});top:calc(var(--u) * {y});"
            f"width:calc(var(--u) * {w});height:calc(var(--u) * {h})")


def hit(key, go, label, x, base, css_class) -> str:
    """Vùng bấm trong suốt nằm đúng trên chữ và mũi tên của mục đó."""
    left, top = x - 56, base - 48
    return (f'<button type="button" class="hit {css_class}" id="hit-{key}" data-go="{go}" '
            f'aria-label="{esc(label)}" style="{box(left, top, HIT_RIGHT - left, 62)}"></button>')


def hits_html() -> str:
    items = [hit(key, key, label, x, base, "v-menu") for label, key, x, _w, base in MENU]
    label, key, x, _w, base = BACK
    items.append(hit(key, "menu", label, x, base, "v-sub"))
    x, y, w, h = OPT_BACK_HIT
    items.append(
        f'<button type="button" class="hit v-opt" id="hit-optback" data-go="menu" '
        f'aria-label="BACK TO MAIN MENU" style="{box(x, y, w, h)}"></button>'
    )
    items.append(
        f'<button type="button" class="hit v-choose" id="hit-chooseback" data-go="menu" '
        f'aria-label="BACK TO MAIN MENU" style="{box(x, y, w, h)}"></button>'
    )
    items.append(
        f'<button type="button" class="hit v-diff" id="hit-diffback" data-back="choose" '
        f'aria-label="BACK TO CHARACTERS" style="{box(x, y, w, h)}"></button>'
    )
    return "\n".join(items)


def hover_rules() -> str:
    glow = "filter:drop-shadow(0 0 6px rgba(255,190,70,.55))"
    rules = []
    for key in [m[1] for m in MENU] + [BACK[1]]:
        sel = f"#stage:has(#hit-{key}:is(:hover,:focus-visible))"
        rules.append(f"{sel} #lbl-{key}{{fill:var(--hover);transform:translateX(-10px);{glow}}}")
        rules.append(f"{sel} #mark-{key}{{opacity:1}}")
    sel = "#stage:has(#hit-optback:is(:hover,:focus-visible))"
    rules.append(f"{sel} #lbl-optback{{fill:var(--hover);{glow}}}")
    sel = "#stage:has(#hit-chooseback:is(:hover,:focus-visible))"
    rules.append(f"{sel} #lbl-chooseback{{fill:var(--hover);{glow}}}")
    sel = "#stage:has(#hit-diffback:is(:hover,:focus-visible))"
    rules.append(f"{sel} #lbl-diffback{{fill:var(--hover);{glow}}}")
    for key, lbl in (("menu", "flbl-menu"), ("options", "flbl-options"), ("pause", "flbl-pause")):
        sel = f"#stage:has(#fhit-{key}:is(:hover,:focus-visible))"
        rules.append(f"{sel} #{lbl}{{fill:var(--hover)}}")
    return "\n".join(rules)


def build_component_html() -> str:
    return (
        HTML_TEMPLATE.replace("__TITLE__", esc(GAME_TITLE))
        .replace("__FONT_URL__", FONT.replace(" ", "+"))
        .replace("__FONT__", FONT)
        .replace("__TEXT__", TEXT_COLOR)
        .replace("__HOVER__", HOVER_COLOR)
        .replace("__INK__", INK_COLOR)
        .replace("__HOVER_RULES__", hover_rules())
        .replace("__SVG__", svg_body())
        .replace("__OPT_HTML__", options_html())
        .replace("__CHOOSE_HTML__", choose_html())
        .replace("__DIFF_HTML__", diff_html())
        .replace("__FIGHT_HTML__", fight_html())
        .replace("__HITS__", hits_html())
        .replace("__MUSIC_TAG__", music_tag())
        .replace("__SFX_CONFIG__", sfx_config())
    )


def to_seconds(value) -> float:
    """Đổi '1:23' (hoặc số giây) thành số giây."""
    if isinstance(value, (int, float)):
        return float(value)
    total = 0.0
    for part in str(value).split(":"):
        total = total * 60 + float(part)
    return total


def music_source():
    """Đường dẫn file nhạc nếu có, không thì trả về None."""
    if not MUSIC_FILE:
        return None
    path = BASE_DIR / MUSIC_FILE
    return path if path.exists() else None


def sync_music() -> None:
    """Chép file nhạc vào thư mục giao diện để trình duyệt phát được."""
    src = music_source()
    if src is None:
        return
    COMPONENT_DIR.mkdir(parents=True, exist_ok=True)
    dst = COMPONENT_DIR / ("music" + src.suffix.lower())
    needs_copy = (not dst.exists()
                  or dst.stat().st_size != src.stat().st_size
                  or dst.stat().st_mtime_ns != src.stat().st_mtime_ns)
    if needs_copy:
        shutil.copy2(src, dst)


def music_tag() -> str:
    """Thẻ <audio> cho giao diện; rỗng nếu chưa có file nhạc."""
    src = music_source()
    if src is None:
        return ""
    end = "" if MUSIC_END is None else str(to_seconds(MUSIC_END))
    return (f'<audio id="bgm" data-src="music{src.suffix.lower()}" preload="none" '
            f'data-start="{to_seconds(MUSIC_START)}" data-end="{end}" '
            f'data-volume="{MUSIC_VOLUME}"></audio>')


def sfx_config() -> str:
    """Cấu hình âm thanh đưa vào giao diện (dạng JSON)."""
    return json.dumps({"on": SFX_ON, "volume": SFX_VOLUME,
                       "buttons": SFX_BUTTONS, "option": SFX_OPTION})


# ═══════════════ MÀN HÌNH CHỌN NHÂN VẬT (SINGLE PLAYER) ═══════════════
# Chân dung là hình tự vẽ bằng SVG, tạo ngẫu nhiên với hạt giống cố định.
SKIN_TONES = ["#f2c9a0", "#e3b286", "#c98e5e", "#a86f45", "#8d5a36", "#f7d9bf"]
HAIR_TONES = ["#1d1a17", "#3a2a1c", "#6b4a2a", "#c9b38a", "#e7e1d6", "#8a1f1f", "#2f3e5c"]
BAND_TONES = ["#c73030", "#2f6fd1", "#f4f0e6", "#2e9e4f", "#e0a526"]
HAIR_STYLES = ["short", "spiky", "long", "topknot", "bald", "ponytail", "mohawk", "fringe"]
HEADWEAR = ["none", "none", "none", "headband", "bandana", "helmet", "straw", "cone", "cap"]
FACIAL = ["none", "none", "mustache", "beard", "goatee"]
MOUTHS = {
    "smile": '<path d="M43 67 Q50 72 57 67" stroke="#6b2b1e" stroke-width="2" fill="none" stroke-linecap="round"/>',
    "neutral": '<path d="M44 67 L56 67" stroke="#6b2b1e" stroke-width="2" stroke-linecap="round"/>',
    "frown": '<path d="M43 69 Q50 64 57 69" stroke="#6b2b1e" stroke-width="2" fill="none" stroke-linecap="round"/>',
    "shout": '<path d="M44 66 Q50 74 56 66 Z" fill="#5a1d17"/>',
}
EYE_RY = {"normal": 2.6, "narrow": 1.8, "wide": 3.4}
EDGE = 'stroke="#2a1a06" stroke-width="1"'


def random_look(rng: random.Random) -> dict:
    """Bộ đặc điểm ngẫu nhiên của một nhân vật (áo lấy từ KITS)."""
    return {
        "skin": rng.choice(SKIN_TONES),
        "hair_c": rng.choice(HAIR_TONES),
        "hair": rng.choice(HAIR_STYLES),
        "headwear": rng.choice(HEADWEAR),
        "band": rng.choice(BAND_TONES),
        "facial": rng.choice(FACIAL),
        "mouth": rng.choice(list(MOUTHS)),
        "eyes": rng.choice(list(EYE_RY)),
        "shades": rng.random() < 0.15,
        "scar": rng.random() < 0.15,
    }


def look(**kw) -> dict:
    """Bộ đặc điểm cho nhân vật có tên riêng: mặc định rồi ghi đè theo kw."""
    base = {"skin": "#e3b286", "hair_c": "#1d1a17", "hair": "short", "headwear": "none",
            "band": "#c73030", "facial": "none", "mouth": "smile", "eyes": "normal",
            "shades": False, "scar": False, "shell": False}
    base.update(kw)
    return base


# Ba nhân vật có tên riêng (vẽ mới, không phỏng theo nhân vật có sẵn)
NAMED = [
    ("BỦ NGAY", look(skin="#a86f45", hair="bald", facial="mustache", eyes="wide")),
    ("SU NGHI", look(skin="#f2c9a0", hair_c="#3a2a1c", hair="ponytail", headwear="headband",
                     band="#2e9e4f", mouth="neutral", eyes="narrow", scar=True)),
    ("MBABY TIRTLE", look(skin="#8fbf5a", hair="bald", eyes="wide", shell=True)),
]


def hair_top(style: str, color: str, band: str) -> str:
    """Tóc phía trên đầu (khi không đội mũ)."""
    cap = f'<path d="M27 46 Q26 22 50 21 Q74 22 73 46 Q70 35 50 35 Q30 35 27 46 Z" fill="{color}"/>'
    if style == "spiky":
        return (f'<path d="M27 42 L23 22 L35 30 L37 12 L46 26 L52 9 L58 26 L65 12 L67 30 L77 22 L73 42 '
                f'Q50 34 27 42 Z" fill="{color}"/>')
    if style == "topknot":
        return (cap + f'<ellipse cx="50" cy="15" rx="8" ry="7" fill="{color}"/>'
                f'<rect x="42" y="20" width="16" height="3.5" rx="1.5" fill="{band}"/>')
    if style == "bald":
        return '<ellipse cx="42" cy="32" rx="6" ry="3" fill="#ffffff" opacity=".25"/>'
    if style == "mohawk":
        return f'<path d="M43 40 L40 12 L50 18 L60 12 L57 40 Z" fill="{color}"/>'
    if style == "fringe":
        return (cap + f'<path d="M30 40 L33 46 L37 40 L41 46 L45 40 L49 46 L53 40 L57 46 '
                f'L61 40 L65 46 L69 40 Z" fill="{color}"/>')
    return cap  # short, long, ponytail: phần tóc đỉnh đầu giống nhau


def headwear(kind: str, band: str) -> str:
    """Mũ hoặc băng đầu."""
    if kind == "headband":
        return (f'<path d="M27 40 Q50 31 73 40 L73 46 Q50 37 27 46 Z" fill="{band}"/>'
                f'<path d="M71 42 L84 49 L82 54 L70 47 Z" fill="{band}"/>')
    if kind == "bandana":
        return (f'<path d="M26 39 Q50 27 74 39 L74 47 Q50 35 26 47 Z" fill="{band}"/>'
                f'<circle cx="75" cy="45" r="4.5" fill="{band}"/>')
    if kind == "helmet":
        return ('<path d="M24 47 Q22 15 50 13 Q78 15 76 47 Q50 41 24 47 Z" fill="#9aa3ad" '
                'stroke="#2a1a06" stroke-width="1"/>'
                '<path d="M22 47 Q50 41 78 47 L79 51 Q50 45 21 51 Z" fill="#5c6670"/>')
    if kind == "straw":
        return ('<ellipse cx="50" cy="36" rx="39" ry="7" fill="#d8b26a" stroke="#2a1a06" stroke-width="1"/>'
                '<path d="M31 36 Q33 15 50 14 Q67 15 69 36 Z" fill="#d8b26a" stroke="#2a1a06" stroke-width="1"/>')
    if kind == "cone":
        return ('<path d="M50 3 L86 38 L14 38 Z" fill="#2a2a2a"/>'
                '<ellipse cx="50" cy="38" rx="38" ry="5" fill="#2a2a2a"/>')
    if kind == "cap":
        return (f'<path d="M26 46 Q28 21 50 20 Q72 21 74 46 Z" fill="{band}"/>'
                '<path d="M22 44 Q50 36 78 44 L82 49 Q50 41 20 49 Z" fill="#2a1a06"/>')
    return ""


def face(p: dict) -> str:
    """Mắt, lông mày, mũi, miệng, râu và các chi tiết khuôn mặt."""
    out = []
    if p["shades"]:
        out.append('<rect x="34" y="45" width="14" height="9" rx="3" fill="#111"/>'
                   '<rect x="52" y="45" width="14" height="9" rx="3" fill="#111"/>'
                   '<path d="M48 48 L52 48" stroke="#111" stroke-width="1.6"/>')
    else:
        ry = EYE_RY[p["eyes"]]
        for cx in (41, 59):
            out.append(f'<ellipse cx="{cx}" cy="50" rx="4" ry="{ry}" fill="#fff"/>')
            out.append(f'<circle cx="{cx + 0.8}" cy="50" r="1.8" fill="#1c1c1c"/>')
        out.append('<path d="M36 43 Q41 40 46 43" stroke="#2a1a0e" stroke-width="2.2" fill="none" '
                   'stroke-linecap="round"/>'
                   '<path d="M54 43 Q59 40 64 43" stroke="#2a1a0e" stroke-width="2.2" fill="none" '
                   'stroke-linecap="round"/>')
    out.append('<path d="M50 52 Q47 58 51 60" stroke="#5c3a22" stroke-width="1.6" fill="none" '
               'stroke-linecap="round"/>')
    if p["facial"] != "beard":
        out.append(MOUTHS[p["mouth"]])
    if p["facial"] == "mustache":
        out.append(f'<path d="M43 63 Q50 58 57 63 Q50 65 43 63 Z" fill="{p["hair_c"]}"/>')
    elif p["facial"] == "beard":
        out.append(f'<path d="M29 54 Q32 80 50 82 Q68 80 71 54 Q67 70 50 70 Q33 70 29 54 Z" fill="{p["hair_c"]}"/>')
    elif p["facial"] == "goatee":
        out.append(f'<path d="M46 72 Q50 82 54 72 Q50 75 46 72 Z" fill="{p["hair_c"]}"/>')
    if p["scar"]:
        out.append('<path d="M62 54 L66 60" stroke="#7a3b2c" stroke-width="1.2" stroke-linecap="round"/>')
    return "".join(out)


def shell_svg() -> str:
    """Mai rùa nằm sau đầu và vai (vẽ mới)."""
    return ('<path d="M18 76 Q20 26 50 22 Q80 26 82 76 Z" fill="#6b7a3a" stroke="#2a1a06" stroke-width="1"/>'
            '<path d="M50 24 L50 52 M34 32 L42 50 M66 32 L58 50" stroke="#3f4a22" stroke-width="1.6" fill="none"/>')


def shirt_svg(num: str, kit: dict) -> str:
    """Áo đấu: nền, họa tiết (sọc ngang, dọc, chéo hoặc một màu), viền cổ và số áo."""
    base, stripe, pat = kit["base"], kit["stripe"], kit["pattern"]
    defs, fill = "", base
    if pat == "hoops":
        defs = (f'<pattern id="kit-{num}" width="100" height="16" patternUnits="userSpaceOnUse">'
                f'<rect width="100" height="8" fill="{base}"/>'
                f'<rect y="8" width="100" height="8" fill="{stripe}"/></pattern>')
    elif pat == "vertical":
        defs = (f'<pattern id="kit-{num}" width="16" height="100" patternUnits="userSpaceOnUse">'
                f'<rect width="8" height="100" fill="{base}"/>'
                f'<rect x="8" width="8" height="100" fill="{stripe}"/></pattern>')
    elif pat == "sash":
        defs = (f'<pattern id="kit-{num}" width="20" height="20" patternUnits="userSpaceOnUse" '
                f'patternTransform="rotate(-35)"><rect width="20" height="20" fill="{base}"/>'
                f'<rect width="7" height="20" fill="{stripe}"/></pattern>')
    if defs:
        fill = f"url(#kit-{num})"
    out = [f"<defs>{defs}</defs>"] if defs else []
    out.append(f'<path d="M12 100 Q16 78 40 73 L60 73 Q84 78 88 100 Z" fill="{fill}"/>')
    out.append(f'<path d="M40 73 Q50 85 60 73 Z" fill="{kit["trim"]}"/>')
    out.append(f'<text x="50" y="94" text-anchor="middle" font-family="Arial, sans-serif" '
               f'font-size="16" font-weight="700" fill="{kit["trim"]}">{num}</text>')
    return "".join(out)


def portrait_svg(p: dict, num: str, kit: dict) -> str:
    """Chân dung một nhân vật (nhìn thẳng) mặc áo của đội trong KITS."""
    skin, hc = p["skin"], p["hair_c"]
    parts = []
    if p.get("shell"):  # mai rùa nằm sau đầu và vai
        parts.append(shell_svg())
    if p["hair"] == "long":  # tóc dài nằm sau áo
        parts.append(f'<path d="M25 38 Q20 86 33 94 L67 94 Q80 86 75 38 Z" fill="{hc}"/>')
    if p["hair"] == "ponytail":
        parts.append(f'<path d="M69 34 Q94 40 90 74 Q84 58 71 54 Z" fill="{hc}"/>')
    parts.append(shirt_svg(num, kit))
    parts.append(f'<rect x="43" y="64" width="14" height="14" fill="{skin}"/>')
    parts.append(f'<ellipse cx="27" cy="50" rx="4.5" ry="6.5" fill="{skin}" {EDGE}/>')
    parts.append(f'<ellipse cx="73" cy="50" rx="4.5" ry="6.5" fill="{skin}" {EDGE}/>')
    parts.append(f'<ellipse cx="50" cy="48" rx="22" ry="25" fill="{skin}" {EDGE}/>')
    if p["headwear"] == "none":
        parts.append(hair_top(p["hair"], hc, p["band"]))
    else:
        parts.append(headwear(p["headwear"], p["band"]))
    parts.append(face(p))
    return '<svg viewBox="0 0 100 100" xmlns="http://www.w3.org/2000/svg">' + "".join(parts) + "</svg>"


def make_characters(count: int = 32, seed: int = 2026) -> list:
    """32 nhân vật ngẫu nhiên, rồi đến 3 nhân vật có tên riêng (áo lấy theo KITS)."""
    rng = random.Random(seed)
    out = []
    for i in range(count):
        num = f"{i + 1:02d}"
        kit = KITS[i % len(KITS)]
        lk = random_look(rng)
        out.append({"id": num, "team": kit["name"], "label": f"FIGHTER {num}", "look": lk, "kit": kit,
                    "svg": portrait_svg(lk, num, kit)})
    for j, (name, lk) in enumerate(NAMED):
        num = f"{count + j + 1:02d}"
        kit = KITS[(count + j) % len(KITS)]
        out.append({"id": num, "team": kit["name"], "label": name, "look": lk, "kit": kit,
                    "svg": portrait_svg(lk, num, kit)})
    return out


CHARACTERS = make_characters()


def choose_svg() -> str:
    """Chữ của màn hình chọn nhân vật: credit, nút BACK, tiêu đề."""
    return (
        '<g class="v-choose">'
        f'<text class="cr" x="12" y="26">{esc(CREDIT_LEFT)}</text>'
        f'<text class="cr" x="{W - 10}" y="26" text-anchor="end">{esc(CREDIT_RIGHT)}</text>'
        + text_el("< BACK TO MAIN MENU", 30, 752, 56, 515, uid="lbl-chooseback")
        + text_el("CHOOSE PLAYER", 1064, 752, 50, 340, uid="lbl-choose")
        + text_el(GAME_TITLE, 28, 893, 140, 1087)
        + text_el(SUBTITLE, 30, 958, 50, 490)
        + text_el(SITE_URL, 30, 1018, 50, 642)
        + "</g>"
    )


def choose_html() -> str:
    """Lưới chân dung (bấm được) và khung xem trước bên phải."""
    tiles = "".join(
        f'<button type="button" class="tile" data-pick="{c["id"]}" data-team="{esc(c["team"])}" '
        f'aria-label="{esc(c["label"])}">{c["svg"]}</button>'
        for c in CHARACTERS
    )
    return ('<div class="choose-html v-choose">'
            f'<div class="grid">{tiles}</div>'
            '<div class="preview"><div class="pv-art" id="pvArt"></div>'
            '<div class="pv-name" id="pvName"></div>'
            '<div class="pv-team" id="pvTeam"></div></div>'
            "</div>")


# ═══════════════ MÀN CHỌN ĐỘ KHÓ (sau khi chọn nhân vật) ═══════════════
# (mã, tên hiển thị, mô tả ngắn)
DIFFICULTIES = [
    ("easy", "EASY", "Dành cho người mới chơi"),
    ("hard", "HARD", "Đối thủ nhanh và thông minh hơn"),
    ("impossible", "IMPOSSIBLE", "Thử thách dành cho cao thủ"),
]


def diff_svg() -> str:
    """Chữ của màn chọn độ khó: credit, nút BACK, tiêu đề."""
    return (
        '<g class="v-diff">'
        f'<text class="cr" x="12" y="26">{esc(CREDIT_LEFT)}</text>'
        f'<text class="cr" x="{W - 10}" y="26" text-anchor="end">{esc(CREDIT_RIGHT)}</text>'
        + text_el("< BACK TO CHARACTERS", 30, 752, 56, 520, uid="lbl-diffback")
        + text_el("CHOOSE DIFFICULTY", 964, 752, 50, 440, uid="lbl-diff")
        + text_el(GAME_TITLE, 28, 893, 140, 1087)
        + text_el(SUBTITLE, 30, 958, 50, 490)
        + text_el(SITE_URL, 30, 1018, 50, 642)
        + "</g>"
    )


def diff_html() -> str:
    """Ba nút độ khó và khung xem trước nhân vật đã chọn."""
    btns = "".join(
        f'<button type="button" class="diff-btn diff-{key}" data-diff="{key}" aria-label="{label}">'
        f'<span class="dl">{label}</span><span class="dc">{esc(caption)}</span></button>'
        for key, label, caption in DIFFICULTIES
    )
    return ('<div class="diff-html v-diff">' + btns
            + '<div class="preview"><div class="pv-art" id="dfArt"></div>'
            + '<div class="pv-name" id="dfName"></div>'
            + '<div class="pv-team" id="dfTeam"></div></div>'
            + "</div>")


# ═══════════════ MÀN ĐÁNH NHAU (sau khi chọn độ khó) ═══════════════
# Nhân vật và nền đều vẽ mới bằng SVG; đối thủ là nhân vật riêng, không lấy từ game nào.
BRUSH_FONT = f"'{FONT}', Impact, 'Arial Black', sans-serif"
FIGHT_OPP_LOOK = look(skin="#b9824f", hair_c="#111111", hair="spiky", headwear="headband",
                      band="#c94a4a", facial="mustache", mouth="frown", eyes="narrow")
FIGHT_OPP_KIT = {"name": "Black Wind", "base": "#2b2f3a", "stripe": "#444b5a",
                 "pattern": "vertical", "trim": "#c94a4a"}


def kit_pattern(pid: str, kit: dict):
    """Họa tiết áo cho hình người đánh nhau: trả về (phần defs, giá trị fill)."""
    base, stripe, pat = kit["base"], kit["stripe"], kit["pattern"]
    if pat == "hoops":
        d = (f'<pattern id="{pid}" width="60" height="14" patternUnits="userSpaceOnUse">'
             f'<rect width="60" height="7" fill="{base}"/><rect y="7" width="60" height="7" fill="{stripe}"/>'
             "</pattern>")
    elif pat == "vertical":
        d = (f'<pattern id="{pid}" width="18" height="100" patternUnits="userSpaceOnUse">'
             f'<rect width="9" height="100" fill="{base}"/><rect x="9" width="9" height="100" fill="{stripe}"/>'
             "</pattern>")
    elif pat == "sash":
        d = (f'<pattern id="{pid}" width="20" height="20" patternUnits="userSpaceOnUse" '
             f'patternTransform="rotate(-35)"><rect width="20" height="20" fill="{base}"/>'
             f'<rect width="7" height="20" fill="{stripe}"/></pattern>')
    else:
        return "", base
    return d, f"url(#{pid})"


def fighter_hair(style: str, color: str, band: str) -> str:
    """Tóc phía trên đầu của nhân vật đánh nhau (khi không đội mũ)."""
    cap = (f'<path d="M-33,-388 Q-34,-424 0,-426 Q34,-424 33,-388 Q22,-402 0,-402 Q-22,-402 -33,-388 Z" '
           f'fill="{color}"/>')
    if style == "spiky":
        return (f'<path d="M-32,-392 L-38,-420 L-22,-408 L-20,-432 L-6,-412 L4,-436 L12,-412 L26,-430 '
                f'L28,-406 L40,-414 L32,-388 Q0,-402 -32,-392 Z" fill="{color}"/>')
    if style == "topknot":
        return (cap + f'<ellipse cx="0" cy="-432" rx="10" ry="9" fill="{color}"/>'
                f'<rect x="-11" y="-424" width="22" height="4" rx="2" fill="{band}"/>')
    if style == "bald":
        return '<ellipse cx="-14" cy="-404" rx="8" ry="4" fill="#ffffff" opacity=".25"/>'
    if style == "mohawk":
        return f'<path d="M-9,-404 L-13,-446 L4,-420 L12,-444 L10,-402 Z" fill="{color}"/>'
    if style == "fringe":
        return (cap + f'<path d="M-30,-392 L-26,-380 L-20,-392 L-14,-380 L-8,-392 L-2,-380 L4,-392 L10,-380 '
                f'L16,-392 L22,-380 L28,-392 Z" fill="{color}"/>')
    return cap  # short, long, ponytail


def fighter_headwear(kind: str, band: str) -> str:
    """Mũ hoặc băng đầu của nhân vật đánh nhau."""
    if kind == "headband":
        return (f'<path d="M-32,-398 Q0,-410 32,-398 L32,-388 Q0,-400 -32,-388 Z" fill="{band}"/>'
                f'<path d="M30,-396 L56,-386 L54,-378 L28,-386 Z" fill="{band}"/>')
    if kind == "bandana":
        return (f'<path d="M-33,-396 Q0,-412 33,-396 L33,-384 Q0,-400 -33,-384 Z" fill="{band}"/>'
                f'<circle cx="34" cy="-390" r="6" fill="{band}"/>')
    if kind == "helmet":
        return ('<path d="M-36,-392 Q-38,-438 0,-440 Q38,-438 36,-392 Q0,-404 -36,-392 Z" fill="#9aa3ad" '
                'stroke="#2a1a06" stroke-width="1.5"/>'
                '<path d="M-38,-392 Q0,-404 38,-392 L39,-386 Q0,-398 -39,-386 Z" fill="#5c6670"/>')
    if kind == "straw":
        return ('<ellipse cx="0" cy="-400" rx="50" ry="10" fill="#d8b26a" stroke="#2a1a06" stroke-width="1.2"/>'
                '<path d="M-27,-400 Q-24,-436 0,-438 Q24,-436 27,-400 Z" fill="#d8b26a" stroke="#2a1a06" '
                'stroke-width="1.2"/>')
    if kind == "cone":
        return ('<path d="M0,-476 L58,-398 L-58,-398 Z" fill="#2a2a2a"/>'
                '<ellipse cx="0" cy="-398" rx="52" ry="7" fill="#2a2a2a"/>')
    if kind == "cap":
        return (f'<path d="M-30,-396 Q-30,-432 0,-434 Q30,-432 30,-396 Z" fill="{band}"/>'
                '<path d="M-34,-396 Q0,-408 36,-396 L40,-390 Q0,-402 -36,-390 Z" fill="#2a1a06"/>')
    return ""


def fighter_face(lk: dict, hc: str) -> str:
    """Mắt, lông mày, mũi, miệng và râu của nhân vật đánh nhau."""
    out = []
    if lk["shades"]:
        out.append('<rect x="-22" y="-393" width="18" height="10" rx="3" fill="#111"/>'
                   '<rect x="4" y="-393" width="18" height="10" rx="3" fill="#111"/>')
    else:
        ry = {"normal": 4.5, "narrow": 3.0, "wide": 5.5}[lk["eyes"]]
        for cx in (-12, 12):
            out.append(f'<ellipse cx="{cx}" cy="-388" rx="6" ry="{ry}" fill="#fff"/>')
            out.append(f'<circle cx="{cx + 1}" cy="-388" r="2.6" fill="#1c1c1c"/>')
        out.append('<path d="M-19,-398 Q-12,-403 -5,-398 M5,-398 Q12,-403 19,-398" stroke="#2a1a0e" '
                   'stroke-width="2.4" fill="none" stroke-linecap="round"/>')
    out.append('<path d="M2,-386 Q0,-379 4,-377" stroke="#5c3a22" stroke-width="1.6" fill="none" '
               'stroke-linecap="round"/>')
    if lk["mouth"] == "shout":
        out.append('<ellipse cx="0" cy="-364" rx="5" ry="4" fill="#5a1d17"/>')
    else:
        d = {"smile": "M-7,-366 Q0,-361 7,-366", "neutral": "M-7,-365 L7,-365",
             "frown": "M-7,-363 Q0,-368 7,-363"}[lk["mouth"]]
        out.append(f'<path d="{d}" stroke="#6b2b1e" stroke-width="2.4" fill="none" stroke-linecap="round"/>')
    if lk["facial"] == "mustache":
        out.append(f'<path d="M-8,-370 Q0,-375 8,-370 Q0,-368 -8,-370 Z" fill="{hc}"/>')
    elif lk["facial"] == "beard":
        out.append(f'<path d="M-22,-374 Q-20,-346 0,-342 Q20,-346 22,-374 Q14,-356 0,-356 Q-14,-356 -22,-374 Z" '
                   f'fill="{hc}"/>')
    elif lk["facial"] == "goatee":
        out.append(f'<path d="M-5,-358 Q0,-346 5,-358 Q0,-355 -5,-358 Z" fill="{hc}"/>')
    if lk["scar"]:
        out.append('<path d="M14,-380 L20,-372" stroke="#7a3b2c" stroke-width="1.4" stroke-linecap="round"/>')
    return "".join(out)


def shade(hex_color: str, k: float) -> str:
    """Làm tối (k < 1) hoặc sáng (k > 1) một màu hex, dùng cho tay áo."""
    h = hex_color.lstrip("#")
    vals = [min(255, max(0, round(int(h[i:i + 2], 16) * k))) for i in (0, 2, 4)]
    return "#" + "".join(f"{v:02x}" for v in vals)


OPP_SWORD = (
    '<g class="osw" transform="translate(160 -306) rotate(-55) scale(1.25)">'
    '<rect x="-22" y="-3" width="22" height="6" fill="#5a3418"/>'
    '<rect x="-3" y="-11" width="6" height="22" fill="#c9a227"/>'
    '<path class="osw-blade" d="M3,-4 L100,-4 L118,0 L100,4 L3,4 Z" fill="#dfe6ea" '
    'stroke="#37474f" stroke-width="1.2"/>'
    "</g>"
)


def fighter_parts(fid: str, lk: dict, kit: dict, weapon: bool = False):
    """Các phần của hình người đánh nhau, quay mặt về phải; gốc toạ độ ở giữa hai bàn chân.
    Trả về (phần defs, nội dung SVG)."""
    skin, hc, band = lk["skin"], lk["hair_c"], lk["band"]
    base, trim = kit["base"], kit["trim"]
    pdefs, gi = kit_pattern(f"fp-{fid}", kit)
    leg = f'fill="none" stroke="{base}" stroke-width="40" stroke-linecap="round" stroke-linejoin="round"'
    sleeve = shade(base, 0.78)
    arm = f'fill="none" stroke="{sleeve}" stroke-width="28" stroke-linecap="round" stroke-linejoin="round"'
    p = ['<ellipse cx="0" cy="6" rx="112" ry="16" fill="#000" opacity=".28"/>']
    if lk["hair"] == "ponytail":
        p.append(f'<path d="M26,-404 Q66,-398 58,-350 Q46,-372 30,-380 Z" fill="{hc}"/>')
    if lk["hair"] == "long":
        p.append(f'<path d="M-27,-392 Q-42,-340 -30,-318 L30,-318 Q42,-340 27,-392 Z" fill="{hc}"/>')
    # Chân sau (leg-b) và chân trước (leg-f)
    p.append(f'<g class="leg-b"><path d="M-16,-196 L-48,-108 L-86,-26" {leg}/>'
             '<ellipse cx="-102" cy="-12" rx="30" ry="12" fill="#1d1d1d"/></g>')
    p.append(f'<g class="leg-f"><path d="M18,-196 L66,-110 L84,-26" {leg}/>'
             '<ellipse cx="102" cy="-12" rx="30" ry="12" fill="#1d1d1d"/></g>')
    if lk.get("shell"):  # mai rùa trên lưng, nằm sau thân
        p.append('<path d="M-30,-330 Q-112,-340 -108,-262 Q-106,-200 -46,-192 L-20,-192 Z" fill="#2f8f3a" '
                 'stroke="#1d5c25" stroke-width="2"/>')
        p.append('<path d="M-96,-292 L-66,-270 L-92,-244 M-66,-270 L-40,-262 M-68,-236 L-46,-214" fill="none" '
                 'stroke="#1d5c25" stroke-width="2.5" stroke-linejoin="round"/>')
    p.append(f'<path d="M-60,-330 Q0,-342 60,-330 L66,-236 Q56,-206 42,-196 L-42,-196 Q-56,-206 -66,-236 Z" '
             f'fill="{gi}"/>')
    p.append(f'<path d="M-14,-334 L4,-262 L18,-334 Z" fill="{trim}"/>')
    p.append(f'<rect x="-50" y="-206" width="100" height="16" fill="{trim}"/>')
    p.append(f'<path d="M28,-200 L48,-160 L36,-156 L20,-196 Z" fill="{trim}"/>')
    # Tay sau (arm-b) và tay trước (arm-f), kèm bàn tay
    p.append(f'<g class="arm-b"><path d="M-46,-318 L-62,-258 L-28,-296" {arm}/>'
             f'<circle cx="-24" cy="-300" r="17" fill="{skin}"/></g>')
    p.append(f'<g class="arm-f"><path d="M48,-318 L104,-302 L150,-306" {arm}/>'
             f'<circle cx="160" cy="-306" r="19" fill="{skin}"/>' + (OPP_SWORD if weapon else "") + '</g>')
    p.append(f'<rect x="-12" y="-352" width="24" height="26" fill="{skin}"/>')
    p.append(f'<ellipse cx="-31" cy="-380" rx="6" ry="9" fill="{skin}"/>')
    p.append(f'<ellipse cx="31" cy="-380" rx="6" ry="9" fill="{skin}"/>')
    p.append(f'<ellipse cx="0" cy="-384" rx="31" ry="37" fill="{skin}" stroke="#2a1a06" stroke-width="1.5"/>')
    p.append(fighter_hair(lk["hair"], hc, band))
    if lk["headwear"] != "none":
        p.append(fighter_headwear(lk["headwear"], band))
    p.append(fighter_face(lk, hc))
    return pdefs, "".join(p)


def fighter_group(fid: str, lk: dict, kit: dict, cls: str, style: str, transform: str, weapon: bool = False):
    """Một nhân vật đánh nhau dưới dạng nhóm SVG có id ft-<fid>."""
    pdefs, inner = fighter_parts(fid, lk, kit, weapon)
    return pdefs, f'<g id="ft-{fid}" class="{cls}" style="{style}" transform="{transform}">{inner}</g>'


def arena_svg() -> str:
    """Nền rừng xanh: các cột tròn, mặt đất và đá nhỏ (vẽ mới)."""
    rng = random.Random(77)
    out = [
        '<defs>'
        '<linearGradient id="fsky" x1="0" y1="0" x2="0" y2="1">'
        '<stop offset="0" stop-color="#2fa89a"/><stop offset=".55" stop-color="#8fe0c8"/>'
        '<stop offset="1" stop-color="#d9f7e6"/></linearGradient>'
        '<linearGradient id="fground" x1="0" y1="0" x2="0" y2="1">'
        '<stop offset="0" stop-color="#2d8f62"/><stop offset="1" stop-color="#17583d"/></linearGradient>'
        '<linearGradient id="fpillar" x1="0" y1="0" x2="1" y2="0">'
        '<stop offset="0" stop-color="#1f9c96"/><stop offset=".5" stop-color="#58d1a8"/>'
        '<stop offset="1" stop-color="#1f8f7e"/></linearGradient></defs>',
        f'<rect width="{W}" height="{H}" fill="url(#fsky)"/>',
    ]
    for _ in range(26):
        x = rng.uniform(-30, W)
        w = rng.uniform(26, 62)
        top = rng.uniform(-40, 260)
        base = rng.uniform(470, 560)
        op = rng.uniform(0.55, 0.95)
        out.append(f'<rect x="{x:.1f}" y="{top:.1f}" width="{w:.1f}" height="{base - top:.1f}" '
                   f'rx="{w / 2:.1f}" fill="url(#fpillar)" opacity="{op:.2f}"/>')
    out.append(f'<rect y="470" width="{W}" height="{H - 470}" fill="url(#fground)"/>')
    out.append('<ellipse cx="724" cy="478" rx="820" ry="70" fill="#e8fff4" opacity=".25"/>')
    for _ in range(14):
        x = rng.uniform(40, 1400)
        y = rng.uniform(900, 1020)
        out.append(f'<ellipse cx="{x:.1f}" cy="{y:.1f}" rx="{rng.uniform(10, 22):.1f}" '
                   f'ry="{rng.uniform(5, 9):.1f}" fill="#e8e8dc" opacity=".9"/>')
    return "".join(out)


def fight_ui_svg() -> str:
    """Thanh máu, tên, tia đỏ, chữ GAME OVER và thanh nút dưới."""
    return (
        '<g id="rays"></g>'
        '<polygon points="0,0 700,0 700,56 0,56" fill="#1e1e1e"/>'
        '<polygon points="748,0 1448,0 1448,56 748,56" fill="#1e1e1e"/>'
        '<rect x="0" y="56" width="700" height="8" fill="#c62828"/>'
        '<rect x="748" y="56" width="700" height="8" fill="#c62828"/>'
        '<rect x="20" y="30" width="670" height="20" fill="#4a0d0d"/>'
        '<rect id="hpP" x="20" y="30" width="670" height="20" fill="#ffd54f"/>'
        '<rect x="758" y="30" width="670" height="20" fill="#4a0d0d"/>'
        '<rect id="hpB" x="758" y="30" width="670" height="20" fill="#ff8f3a"/>'
        '<text id="fnameL" x="22" y="22" font-family="Arial, Helvetica, sans-serif" font-size="20" '
        'font-weight="700" fill="#f4f4f4">FIGHTER</text>'
        '<text id="fnameR" x="1426" y="22" text-anchor="end" font-family="Arial, Helvetica, sans-serif" '
        'font-size="20" font-weight="700" fill="#f4f4f4">BLACK WIND</text>'
        '<polygon points="540,64 908,64 886,108 562,108" fill="#1b1b1b" opacity=".92"/>'
        f'<text id="fbanner" x="724" y="96" text-anchor="middle" font-family="{BRUSH_FONT}" '
        'font-size="34" font-weight="700" fill="#fbf5a4">HARD</text>'
        '<polygon points="0,1030 1448,1030 1448,1086 0,1086" fill="#0d0d0d" opacity=".92"/>'
        '<text id="flbl-menu" x="36" y="1068" font-family="Arial, Helvetica, sans-serif" font-size="34" '
        'fill="#f4f4f4">MAIN MENU</text>'
        '<text id="flbl-options" x="1270" y="1068" text-anchor="end" font-family="Arial, Helvetica, sans-serif" '
        'font-size="34" fill="#f4f4f4">OPTIONS</text>'
        '<text id="flbl-pause" x="1420" y="1068" text-anchor="end" font-family="Arial, Helvetica, sans-serif" '
        'font-size="34" fill="#f4f4f4">PAUSE</text>'
        '<g id="ko" style="display:none"><rect width="1448" height="1086" fill="#000" opacity=".55"/>'
        f'<text id="kotxt" x="724" y="560" text-anchor="middle" font-family="{BRUSH_FONT}" font-size="150" '
        'font-weight="700" fill="#ff3b2f">GAME OVER</text></g>'
        '<g id="win" style="display:none"><rect width="1448" height="1086" fill="#000" opacity=".45"/>'
        f'<text id="wintxt" x="724" y="560" text-anchor="middle" font-family="{BRUSH_FONT}" font-size="150" '
        'font-weight="700" fill="#ffd54f">YOU WIN</text></g>'
        '<g class="pause-layer"><rect width="1448" height="1086" fill="#000" opacity=".5"/>'
        f'<text x="724" y="580" text-anchor="middle" font-family="{BRUSH_FONT}" font-size="110" '
        'font-weight="700" fill="#fbf5a4">PAUSE</text></g>'
    )


def fight_svg() -> str:
    """Màn đánh nhau: nền rừng, hai nhân vật (chỉ hiện nhân vật đã chọn) và các thanh trên, dưới."""
    pdefs, groups = [], []
    for c in CHARACTERS:
        d, g = fighter_group(c["id"], c["look"], c["kit"], "ft", "display:none",
                             "translate(360 880) scale(1.2 1.2)")
        pdefs.append(d)
        groups.append(g)
    d, g = fighter_group("opp", FIGHT_OPP_LOOK, FIGHT_OPP_KIT, "", "",
                         "translate(1090 880) scale(-1.2 1.2)", weapon=True)
    pdefs.append(d)
    groups.append(g)
    return ('<g class="v-fight"><defs>' + "".join(pdefs) + "</defs>"
            + arena_svg() + "".join(groups) + fight_ui_svg() + "</g>")


def fight_html() -> str:
    """Ba nút dưới màn đánh nhau: MAIN MENU, OPTIONS, PAUSE."""
    return ('<div class="fight-html v-fight">'
            f'<button type="button" class="fight-btn" id="fhit-menu" data-fight="menu" '
            f'aria-label="MAIN MENU" style="{box(20, 1030, 260, 56)}"></button>'
            f'<button type="button" class="fight-btn" id="fhit-options" data-fight="options" '
            f'aria-label="OPTIONS" style="{box(1040, 1030, 240, 56)}"></button>'
            f'<button type="button" class="fight-btn" id="fhit-pause" data-fight="pause" '
            f'aria-label="PAUSE" style="{box(1290, 1030, 140, 56)}"></button>'
            "</div>")


def write_component() -> None:
    COMPONENT_DIR.mkdir(parents=True, exist_ok=True)
    target = COMPONENT_DIR / "index.html"
    page = build_component_html()
    if not target.exists() or target.read_text(encoding="utf-8") != page:
        target.write_text(page, encoding="utf-8")


# ═══════════════ CHẠY ỨNG DỤNG ═══════════════
st.set_page_config(
    page_title=GAME_TITLE, page_icon="⚽", layout="wide", initial_sidebar_state="collapsed"
)
st.markdown("<style>" + " ".join(PAGE_CSS.split()) + "</style>", unsafe_allow_html=True)

sync_music()
write_component()
menu_component = components.declare_component("wcf_menu", path=str(COMPONENT_DIR))

# "single" | "two" | "editor" | "options" khi bấm mục menu, "menu" khi bấm BACK
choice = menu_component(key="wcf_menu", default=None)

# Ví dụ: if choice == "single": chạy chế độ 1 người ở đây
# Ví dụ: if choice and choice.startswith("pick:"): nhân vật = choice[5:]
# Ví dụ: if choice and choice.startswith("start:"): _, ma, do = choice.split(":")