# brotherwatch.py - Screen Time Management System

```python
import argparse
import sqlite3
import datetime
import os
import subprocess
import sys

DB_PATH = "/var/lib/brotherwatch/brotherwatch.db"
DEFAULT_MAX_SESSION_MIN = 45
DEFAULT_COOLDOWN_MIN = 240

def get_db():
    if os.geteuid() != 0:
        sys.exit("Run this with sudo — it needs access to the shared database.")
    os.makedirs(os.path.dirname(DB_PATH), exist_ok=True)
    conn = sqlite3.connect(DB_PATH)
    conn.execute("""
        CREATE TABLE IF NOT EXISTS users (
            username TEXT PRIMARY KEY,
            daily_allowance_min INTEGER NOT NULL,
            remaining_min REAL NOT NULL,
            last_refill_date TEXT NOT NULL,
            warned INTEGER NOT NULL DEFAULT 0,
            max_session_min INTEGER NOT NULL DEFAULT 45,
            cooldown_min INTEGER NOT NULL DEFAULT 240,
            session_active_since TEXT,
            cooldown_until TEXT
        )
    """)
    conn.execute("""
        CREATE TABLE IF NOT EXISTS daily_usage (
            username TEXT NOT NULL,
            date TEXT NOT NULL,
            minutes_used REAL NOT NULL DEFAULT 0,
            PRIMARY KEY (username, date)
        )
    """)
    
    existing_cols = {row[1] for row in conn.execute("PRAGMA table_info(users)")}
    migrations = {
        "max_session_min": "ALTER TABLE users ADD COLUMN max_session_min INTEGER NOT NULL DEFAULT (DEFAULT_MAX_SESSION_MIN)",
        "cooldown_min": "ALTER TABLE users ADD COLUMN cooldown_min INTEGER NOT NULL DEFAULT (DEFAULT_COOLDOWN_MIN)",
        "session_active_since": "ALTER TABLE users ADD COLUMN session_active_since TEXT",
        "cooldown_until": "ALTER TABLE users ADD COLUMN cooldown_until TEXT",
    }
    for col, ddl in migrations.items():
        if col not in existing_cols:
            conn.execute(ddl)
    conn.commit()
    return conn

def cmd_add_user(args):
    conn = get_db()
    today = datetime.date.today().isoformat()
    conn.execute("""
        INSERT INTO users (username, daily_allowance_min, remaining_min, last_refill_date,
            warned, max_session_min, cooldown_min)
        VALUES (?, ?, ?, ?, 0, ?, 0)
        ON CONFLICT(username) DO UPDATE SET
            daily_allowance_min=excluded.daily_allowance_min,
            max_session_min=excluded.max_session_min,
            cooldown_min=excluded.cooldown_min
    """, (args.username, args.minutes, args.minutes, today, args.session_min, args.cooldown_min))
    conn.commit()
    print(f"{args.username}: daily allowance {args.minutes} min, session cap {args.session_min} min, cooldown {args.cooldown_min} min.")

def cmd_status(args):
    conn = get_db()
    rows = conn.execute("""
        SELECT username, daily_allowance_min, remaining_min, last_refill_date,
            max_session_min, cooldown_min, cooldown_until
        FROM users
    """).fetchall()
    if not rows:
        print("No monitored users configured yet. Use: brotherwatch add-user <name> <minutes>")
        return
    now = datetime.datetime.now()
    print(f"{'User':<12}{'Allowance':<12}{'Remaining':<12}{'Last Refill':<12}{'State':<18}")
    print("=" * 66)
    for username, allowance, remaining, last_refill, max_session, cooldown_min, cooldown_until in rows:
        if cooldown_until:
            until = datetime.datetime.fromisoformat(cooldown_until)
            if now < until:
                mins_left = (until - now).total_seconds() / 60
                state = f"locked ({mins_left:.0f}m left)"
            else:
                state = "active"
        else:
            state = "active"
        print(f"{username:<12}{allowance:<12}{remaining:.0f}m left{last_refill:<12}{state:<18}")

def cmd_add_time(args):
    conn = get_db()
    cur = conn.execute(
        "UPDATE users SET remaining_min=remaining_min + ? WHERE username=?",
        (args.minutes, args.username)
    )
    if cur.rowcount == 0:
        print(f"No such monitored user: {args.username}")
        return
    conn.commit()
    row = conn.execute(
        "SELECT remaining_min FROM users WHERE username=?", (args.username,)
    ).fetchone()
    print(f"{args.username} now has {row[0]:.1f} minutes remaining today.")

def cmd_set_daily(args):
    conn = get_db()
    cur = conn.execute(
        "UPDATE users SET daily_allowance_min=? WHERE username=?",
        (args.minutes, args.username)
    )
    if cur.rowcount == 0:
        print(f"No such monitored user: {args.username}")
        return
    conn.commit()
    print(f"{args.username}'s daily allowance set to {args.minutes} minutes (applies from next refill).")

def cmd_set_session(args):
    conn = get_db()
    cur = conn.execute(
        "UPDATE users SET max_session_min=?, cooldown_min=? WHERE username=?",
        (args.session_min, args.cooldown_min, args.username)
    )
    if cur.rowcount == 0:
        print(f"No such monitored user: {args.username}")
        return
    conn.commit()
    print(f"{args.username}: session cap {args.session_min} min, cooldown {args.cooldown_min} min.")

def cmd_unlock(args):
    conn = get_db()
    cur = conn.execute(
        "UPDATE users SET cooldown_until=NULL, session_active_since=NULL WHERE username=?",
        (args.username,)
    )
    if cur.rowcount == 0:
        print(f"No such monitored user: {args.username}")
        return
    conn.commit()
    subprocess.run(["usermod", "-U", args.username], check=False)
    print(f"{args.username} unlocked.")

def cmd_reset(args):
    conn = get_db()
    row = conn.execute(
        "SELECT daily_allowance_min FROM users WHERE username=?", (args.username,)
    ).fetchone()
    if not row:
        print(f"No such monitored user: {args.username}")
        return
    today = datetime.date.today().isoformat()
    conn.execute("""
        UPDATE users
        SET remaining_min=?, last_refill_date=?, warned=0,
            cooldown_until=NULL, session_active_since=NULL
        WHERE username=?
    """, (row[0], today, args.username))
    conn.commit()
    subprocess.run(["usermod", "-U", args.username], check=False)
    print(f"{args.username} reset to full daily allowance ({row[0]} minutes) and unlocked.")

def cmd_history(args):
    conn = get_db()
    rows = conn.execute("""
        SELECT date, minutes_used FROM daily_usage
        WHERE username=? ORDER BY date DESC LIMIT ?
    """, (args.username, args.days)).fetchall()
    if not rows:
        print(f"No usage history for {args.username} yet.")
        return
    print(f"Usage history for {args.username}:")
    for date, minutes in rows:
        print(f"  {date}: {minutes:.1f} min")

def main():
    parser = argparse.ArgumentParser(prog="brotherwatch", description="Manage screen time budgets.")
    sub = parser.add_subparsers(dest="command", required=True)

    p = sub.add_parser("add-user", help="Add or update a monitored user's daily allowance")
    p.add_argument("username")
    p.add_argument("minutes", type=int)
    p.add_argument("--session-min", type=int, default=DEFAULT_MAX_SESSION_MIN,
                   help=f"Continuous session cap in minutes (default {DEFAULT_MAX_SESSION_MIN})")
    p.add_argument("--cooldown-min", type=int, default=DEFAULT_COOLDOWN_MIN,
                   help=f"Cooldown break length in minutes (default {DEFAULT_COOLDOWN_MIN})")
    p.set_defaults(func=cmd_add_user)

    p = sub.add_parser("status", help="Show all monitored users' current status")
    p.set_defaults(func=cmd_status)

    p = sub.add_parser("add-time", help="Add bonus minutes to a user's remaining time today")
    p.add_argument("username")
    p.add_argument("minutes", type=float)
    p.set_defaults(func=cmd_add_time)

    p = sub.add_parser("set-daily", help="Change a user's daily allowance")
    p.add_argument("username")
    p.add_argument("minutes", type=int)
    p.set_defaults(func=cmd_set_daily)

    p = sub.add_parser("set-session", help="Change a user's session cap and cooldown length")
    p.add_argument("username")
    p.add_argument("--session-min", type=int, required=True)
    p.add_argument("--cooldown-min", type=int, required=True)
    p.set_defaults(func=cmd_set_session)

    p = sub.add_parser("unlock", help="Manually clear a lock/cooldown right now")
    p.add_argument("username")
    p.set_defaults(func=cmd_unlock)

    p = sub.add_parser("reset", help="Reset a user to full daily allowance and unlock, right now")
    p.add_argument("username")
    p.set_defaults(func=cmd_reset)

    p = sub.add_parser("history", help="Show recent daily usage for a user")
    p.add_argument("username")
    p.add_argument("--days", type=int, default=14)
    p.set_defaults(func=cmd_history)

    args = parser.parse_args()
    args.func(args)

if __name__ == "__main__":
    main()
```

## Qisqa tavsif

Bu Python CLI — **screen time (ekran vaqti) monitoring tizimi** ota-ona/kattalar uchun:

- **`add-user`** — yangi user qo'shish, kunlik limit o'rnatish
- **`status`** — barcha user'larning hozirgi holati (qolgan vaqt, locked/active)
- **`add-time`** — bugungi qolgan vaqtga daqiqa qo'shish
- **`set-daily`** — kunlik limitni o'zgartirish
- **`set-session`** — bir o'tirishdagi cap va tanaffus vaqti
- **`unlock`** / **`reset`** — qulfni ochish, to'liq reset
- **`history`** — so'nggi kunlik sarflarni ko'rish

**Storage:** SQLite database `/var/lib/brotherwatch/brotherwatch.db` — faqat root o'z kirita oladi.

**Muhim:** `usermod -L/-U` orqali Linux account'ni lock/unlock qiladi, demak bu faqat shunchaki vaqt app emas — tizim darajasidagi nazorat.

<!DOCTYPE html>
<html lang="uz">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>brotherwatch // control</title>
<style>
  @import url('https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@400;500;700;800&family=Space+Grotesk:wght@500;700&display=swap');

  :root{
    --bg: #0b0f0d;
    --panel: #101512;
    --line: #223028;
    --amber: #ffb454;
    --green: #7ee787;
    --green-dim: #2e5d3f;
    --red: #ff6b6b;
    --text: #d7e0da;
    --text-dim: #6f8378;
    --mono: 'JetBrains Mono', monospace;
    --disp: 'Space Grotesk', sans-serif;
  }

  *{box-sizing:border-box; margin:0; padding:0;}
  html,body{ background:var(--bg); color:var(--text); font-family:var(--mono); min-height:100vh; }
  body{
    background-image:
      linear-gradient(rgba(126,231,135,0.025) 1px, transparent 1px),
      linear-gradient(90deg, rgba(126,231,135,0.025) 1px, transparent 1px);
    background-size: 28px 28px;
  }

  .wrap{ max-width: 640px; margin:0 auto; padding: 36px 20px 90px; }

  header{
    display:flex; align-items:center; gap:10px;
    border-bottom:1px solid var(--line); padding-bottom:16px; margin-bottom:24px;
  }
  .dot{ width:9px; height:9px; border-radius:50%; background:var(--green); box-shadow:0 0 10px var(--green); animation:pulse 2s infinite; }
  @keyframes pulse{ 0%,100%{opacity:1;} 50%{opacity:.35;} }
  h1{ font-family:var(--disp); font-size:20px; letter-spacing:-0.02em; font-weight:700; color:#eef5f0; }
  h1 span{ color:var(--text-dim); font-weight:500; }

  /* USER PICKER */
  .userpicker{ margin-bottom:26px; }
  .userpicker .lbl{ font-size:11px; text-transform:uppercase; letter-spacing:.08em; color:var(--text-dim); margin-bottom:8px; }
  .user-tabs{ display:flex; gap:8px; flex-wrap:wrap; }
  .user-tab{
    flex:1; min-width:100px; background:var(--panel); border:1px solid var(--line);
    color:var(--text-dim); font-family:var(--mono); font-size:13px; padding:12px 10px;
    border-radius:7px; cursor:pointer; text-align:center; transition:all .15s;
  }
  .user-tab:hover{ border-color:#33473b; color:var(--text); }
  .user-tab.active{
    background:rgba(126,231,135,0.08); border-color:var(--green); color:var(--green); font-weight:700;
  }
  .user-tab .uname{ display:block; font-family:var(--disp); font-size:15px; }
  .user-tab .ualias{ display:block; font-size:10px; opacity:.7; margin-top:2px; }

  .custom-user input{
    width:100%; background:#0d1210; border:1px solid var(--line); color:var(--text);
    font-family:var(--mono); font-size:12px; padding:8px 10px; border-radius:6px; outline:none;
    margin-top:8px; display:none;
  }
  .custom-user input.show{ display:block; }

  /* section label */
  .seclabel{
    display:flex; align-items:center; gap:10px; margin:28px 0 12px;
    font-size:11px; text-transform:uppercase; letter-spacing:.1em; color:var(--text-dim);
  }
  .seclabel::after{ content:''; flex:1; height:1px; background:var(--line); }

  /* action grid */
  .grid{ display:grid; grid-template-columns:repeat(auto-fit,minmax(220px,1fr)); gap:12px; }
  .card{
    background:var(--panel); border:1px solid var(--line); border-radius:8px;
    padding:16px; transition:border-color .15s ease;
  }
  .card:hover{ border-color:#33473b; }
  .card h3{ font-family:var(--disp); font-size:14px; margin-bottom:4px; color:#eef5f0;}
  .card p{ font-size:11px; color:var(--text-dim); line-height:1.45; margin-bottom:12px; min-height:28px;}

  .inline-field{ display:flex; gap:6px; margin-bottom:10px; }
  .inline-field input{
    flex:1; width:0; background:#0d1210; border:1px solid var(--line); color:var(--text);
    font-family:var(--mono); font-size:13px; padding:7px 9px; border-radius:5px; outline:none;
  }
  .inline-field input:focus{ border-color:var(--green-dim); }
  .inline-field input[type="date"]::-webkit-calendar-picker-indicator {
    filter: invert(0.8); cursor: pointer;
  }
  .inline-field .unit{ font-size:11px; color:var(--text-dim); align-self:center; white-space:nowrap; }

  .add-btn{
    width:100%; background:transparent; border:1px solid var(--green-dim);
    color:var(--green); font-family:var(--mono); font-size:12px; padding:9px; border-radius:5px;
    cursor:pointer; text-transform:uppercase; letter-spacing:.05em; transition:all .15s;
  }
  .add-btn:hover{ background:rgba(126,231,135,0.08); border-color:var(--green); }
  .add-btn:active{ transform:scale(.98); }

  .danger .add-btn{ border-color:#5d2e2e; color:var(--red); }
  .danger .add-btn:hover{ background:rgba(255,107,107,0.08); border-color:var(--red); }

  .warning .add-btn{ border-color:#685025; color:var(--amber); }
  .warning .add-btn:hover{ background:rgba(255,180,84,0.08); border-color:var(--amber); }

  /* advanced toggle */
  .adv-toggle{
    width:100%; display:flex; align-items:center; justify-content:space-between;
    background:var(--panel); border:1px solid var(--line); border-radius:8px;
    padding:14px 16px; cursor:pointer; margin-top:30px; transition:border-color .15s;
  }
  .adv-toggle:hover{ border-color:#33473b; }
  .adv-toggle .lbl{ font-family:var(--disp); font-size:14px; color:var(--text); display:flex; align-items:center; gap:8px; }
  .adv-toggle .lbl .icon{ color:var(--amber); font-size:12px; }
  .adv-toggle .chev{ color:var(--text-dim); font-size:12px; transition:transform .2s; }
  .adv-toggle.open .chev{ transform:rotate(180deg); }

  .adv-panel{
    max-height:0; overflow:hidden; transition:max-height .25s ease;
  }
  .adv-panel.open{ max-height:1600px; }
  .adv-inner{ padding-top:16px; }

  /* terminal */
  .term{ background:#070a08; border:1px solid var(--line); border-radius:8px; margin-top:30px; overflow:hidden; }
  .term-head{ display:flex; justify-content:space-between; align-items:center; padding:10px 14px; border-bottom:1px solid var(--line); background:#0c110e; }
  .term-head .lights{ display:flex; gap:6px; }
  .term-head .lights span{ width:10px; height:10px; border-radius:50%; display:block; }
  .term-head .lights span:nth-child(1){ background:#ff5f56;}
  .term-head .lights span:nth-child(2){ background:#ffbd2e;}
  .term-head .lights span:nth-child(3){ background:#27c93f;}
  .term-title{ font-size:11px; color:var(--text-dim); }
  .copy-btn{
    background:transparent; border:1px solid var(--line); color:var(--text-dim);
    font-family:var(--mono); font-size:11px; padding:6px 12px; border-radius:5px; cursor:pointer; transition:all .15s;
  }
  .copy-btn:hover{ border-color:var(--green); color:var(--green); }
  .copy-btn.copied{ border-color:var(--green); color:var(--green); }

  #output{ padding:16px 14px; min-height:100px; font-size:13px; line-height:1.9; }
  .cmd-line{ display:flex; gap:8px; align-items:flex-start; margin-bottom:2px; animation:appear .2s ease; }
  @keyframes appear{ from{opacity:0; transform:translateY(-3px);} to{opacity:1; transform:translateY(0);} }
  .cmd-line .prompt{ color:var(--green); flex-shrink:0; }
  .cmd-line .cmdtext{ color:#eef5f0; word-break:break-all; }
  .cmd-line .rm{ margin-left:auto; color:var(--text-dim); cursor:pointer; flex-shrink:0; font-size:12px; padding:0 4px; user-select:none; }
  .cmd-line .rm:hover{ color:var(--red); }
  .empty-hint{ color:var(--text-dim); font-size:12px; font-style:italic; }
  .cursor-line{ color:var(--text-dim); }
  .cursor-line::after{ content:'▍'; color:var(--green); animation:blink 1s step-end infinite; }
  @keyframes blink{ 50%{opacity:0;} }

  footer{ margin-top:30px; text-align:center; font-size:11px; color:var(--text-dim); }

  @media (prefers-reduced-motion: reduce){
    .dot, .cursor-line::after, .cmd-line{ animation:none !important; }
  }
</style>
</head>
<body>

<div class="wrap">

  <header>
    <span class="dot"></span>
    <h1>brotherwatch <span>// control panel</span></h1>
  </header>

  <!-- USER PICKER -->
  <div class="userpicker">
    <div class="lbl">Kim uchun?</div>
    <div class="user-tabs" id="userTabs">
      <div class="user-tab active" data-user="muhammad">
        <span class="uname">muhammad</span>
        <span class="ualias">MR / bratan</span>
      </div>
      <div class="user-tab" data-user="komoliddin">
        <span class="uname">komoliddin</span>
        <span class="ualias">ukam</span>
      </div>
      <div class="user-tab" data-user="__custom">
        <span class="uname">boshqa...</span>
      </div>
    </div>
    <div class="custom-user">
      <input type="text" id="customUserInput" placeholder="username yoz">
    </div>
  </div>

  <!-- ===== ASOSIY ===== -->
  <div class="seclabel">asosiy</div>
  <div class="grid">

    <div class="card">
      <h3>Holatni ko'rish</h3>
      <p>Barcha userlar bo'yicha to'liq status jadvali.</p>
      <button class="add-btn" onclick="addCmd(BIN + ' status')">status</button>
    </div>

    <div class="card">
      <h3>Vaqt qo'shish / ayirish</h3>
      <p>Bugungi qolgan vaqtga daqiqa qo'sh (ayirish uchun: -30).</p>
      <div class="inline-field">
        <input type="text" id="at-min" placeholder="60" value="60">
        <span class="unit">min</span>
      </div>
      <button class="add-btn" onclick="addTime()">qo'sh</button>
    </div>

    <div class="card">
      <h3>Kunlik limit (Daily)</h3>
      <p>Har kungi umumiy daqiqa limitini o'zgartiradi.</p>
      <div class="inline-field">
        <input type="text" id="sd-min" placeholder="120" value="120">
        <span class="unit">min/kun</span>
      </div>
      <button class="add-btn" onclick="addSetDaily()">o'rnat</button>
    </div>

    <div class="card">
      <h3>Tanaffus (Cooldown)</h3>
      <p>Session tugagach kutish vaqti. Session cap avvalgi qiymatida qoladi.</p>
      <div class="inline-field">
        <input type="text" id="cd-min" placeholder="240" value="240">
        <span class="unit">min</span>
      </div>
      <button class="add-btn" onclick="addCooldown()">o'rnat</button>
    </div>

  </div>

  <!-- ===== ADVANCED TOGGLE ===== -->
  <div class="adv-toggle" id="advToggle" onclick="toggleAdv()">
    <span class="lbl"><span class="icon">▲</span> Advanced</span>
    <span class="chev">▾</span>
  </div>

  <div class="adv-panel" id="advPanel">
    <div class="adv-inner">
      <div class="grid">

        <div class="card">
          <h3>Qulfni ochish (unlock)</h3>
          <p>Joriy cooldown/lockni hozir tozalaydi.</p>
          <button class="add-btn" onclick="addCmd(BIN + ' unlock ' + currentUser())">unlock</button>
        </div>

        <div class="card danger">
          <h3>To'liq reset</h3>
          <p>Kunlik hisobni to'liq to'ldiradi va qulfni ochadi.</p>
          <button class="add-btn" onclick="addCmd(BIN + ' reset ' + currentUser())">reset</button>
        </div>

        <div class="card danger">
          <h3>Barcha sarfni tozalash</h3>
          <p>Barcha userlar uchun bugungi tarixni o'chiradi va limitlarni noldan boshlaydi.</p>
          <button class="add-btn" onclick="addClearTodayAll()">tozalash</button>
        </div>

        <div class="card warning">
          <h3>O'tgan kunni tahrirlash</h3>
          <p>Xato ketgan sanadagi vaqtni to'g'ridan-to'g'ri SQlite orqali yangilaydi.</p>
          <div class="inline-field">
            <input type="date" id="ed-date" style="flex: 1.5;">
            <input type="text" id="ed-min" placeholder="95">
            <span class="unit">min</span>
          </div>
          <button class="add-btn" onclick="addEditHistory()">to'g'rilash</button>
        </div>

        <div class="card">
          <h3>Session cap</h3>
          <p>Bir o'tirishdagi uzluksiz ishlatish chegarasi.</p>
          <div class="inline-field">
            <input type="text" id="ss-sess" placeholder="40" value="40">
            <span class="unit">min</span>
          </div>
          <button class="add-btn" onclick="addSessionOnly()">o'rnat</button>
        </div>

        <div class="card">
          <h3>Tarixni ko'rish</h3>
          <p>Oxirgi necha kunlik sarfni ko'rsatadi.</p>
          <div class="inline-field">
            <input type="text" id="hs-days" placeholder="14" value="14">
            <span class="unit">kun</span>
          </div>
          <button class="add-btn" onclick="addHistory()">history</button>
        </div>

        <div class="card">
          <h3>Yangi user qo'shish</h3>
          <p>Kuzatuvga yangi foydalanuvchi qo'shadi.</p>
          <div class="inline-field">
            <input type="text" id="au-min" placeholder="kunlik min" value="120">
            <span class="unit">min/kun</span>
          </div>
          <button class="add-btn" onclick="addUser()">add-user</button>
        </div>

      </div>
    </div>
  </div>

  <!-- TERMINAL OUTPUT -->
  <div class="term">
    <div class="term-head">
      <div class="lights"><span></span><span></span><span></span></div>
      <div class="term-title">tayyor buyruqlar</div>
      <button class="copy-btn" id="copyBtn" onclick="copyAll()">nusxa olish</button>
    </div>
    <div id="output">
      <div class="empty-hint">— hozircha buyruq qo'shilmagan —</div>
    </div>
  </div>

  <footer>faqat matn tayyorlaydi, hech qayerga ulanmaydi — o'zing terminalga yozib ishga tushirasan</footer>

</div>

<script>
  const BIN = '/usr/local/bin/brotherwatch';
  let commands = [];
  let selectedUser = 'muhammad';
  // remember last known session/cooldown so single-field edits don't clobber the other
  let lastSession = '40';
  let lastCooldown = '240';

  const tabs = document.querySelectorAll('.user-tab');
  const customInput = document.getElementById('customUserInput');

  // Sahifa yuklanganda sana qatoriga bugungi sanani qo'yib qo'yamiz
  window.addEventListener('DOMContentLoaded', () => {
    const today = new Date().toISOString().split('T')[0];
    document.getElementById('ed-date').value = today;
  });

  tabs.forEach(tab => {
    tab.addEventListener('click', () => {
      tabs.forEach(t => t.classList.remove('active'));
      tab.classList.add('active');
      const u = tab.dataset.user;
      if(u === '__custom'){
        customInput.classList.add('show');
        customInput.focus();
        selectedUser = customInput.value.trim() || '';
      } else {
        customInput.classList.remove('show');
        selectedUser = u;
      }
    });
  });

  customInput.addEventListener('input', () => {
    selectedUser = customInput.value.trim();
  });

  function currentUser(){
    return selectedUser || '???';
  }

  function toggleAdv(){
    document.getElementById('advToggle').classList.toggle('open');
    document.getElementById('advPanel').classList.toggle('open');
  }

  function render(){
    const out = document.getElementById('output');
    if(commands.length === 0){
      out.innerHTML = '<div class="empty-hint">— hozircha buyruq qo\'shilmagan —</div>';
      return;
    }
    out.innerHTML = commands.map((c,i) => `
      <div class="cmd-line">
        <span class="prompt">#</span>
        <span class="cmdtext">${escapeHtml(c)}</span>
        <span class="rm" onclick="removeCmd(${i})" title="olib tashlash">✕</span>
      </div>
    `).join('') + '<div class="cursor-line"></div>';
  }

  function escapeHtml(s){
    return s.replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;');
  }

  function addCmd(cmd){
    if(currentUser() === '???' || currentUser() === ''){
      alert('Avval userni tanla yoki yoz');
      return;
    }
    commands.push(cmd);
    render();
    document.getElementById('copyBtn').classList.remove('copied');
    document.getElementById('copyBtn').textContent = 'nusxa olish';
  }

  function removeCmd(i){
    commands.splice(i,1);
    render();
  }

  function val(id){ return document.getElementById(id).value.trim(); }

  function addTime(){
    const m = val('at-min');
    if(!m){ alert('Minutlar kerak'); return; }
    if(m.startsWith('-')){
      addCmd(`${BIN} add-time -- ${currentUser()} ${m}`);
    } else {
      addCmd(`${BIN} add-time ${currentUser()} ${m}`);
    }
  }

  function addSetDaily(){
    const m = val('sd-min');
    if(!m){ alert('Minutlar kerak'); return; }
    addCmd(`${BIN} set-daily ${currentUser()} ${m}`);
  }

  function addCooldown(){
    const c = val('cd-min');
    if(!c){ alert('Cooldown kerak'); return; }
    lastCooldown = c;
    addCmd(`${BIN} set-session ${currentUser()} --session-min ${lastSession} --cooldown-min ${c}`);
  }

  function addSessionOnly(){
    const s = val('ss-sess');
    if(!s){ alert('Session kerak'); return; }
    lastSession = s;
    addCmd(`${BIN} set-session ${currentUser()} --session-min ${s} --cooldown-min ${lastCooldown}`);
  }

  function addHistory(){
    const d = val('hs-days');
    addCmd(`${BIN} history ${currentUser()}${d ? ' --days ' + d : ''}`);
  }

  function addUser(){
    const m = val('au-min');
    if(!m){ alert('Kunlik minutlar kerak'); return; }
    addCmd(`${BIN} add-user ${currentUser()} ${m}`);
  }

  // To'g'ridan to'g'ri SQLite orqali tahrirlash
  function addEditHistory(){
    const d = val('ed-date');
    const m = val('ed-min');
    if(!d){ alert('Sanani tanlang'); return; }
    if(!m){ alert('Minut miqdorini kiriting'); return; }
    
    const cmd = `sudo sqlite3 /var/lib/brotherwatch/brotherwatch.db "UPDATE daily_usage SET minutes_used = ${m} WHERE username = '${currentUser()}' AND date = '${d}';"`;
    addCmd(cmd);
  }

  // Barcha sarflarni nolga tushirish funksiyasi
  function addClearTodayAll(){
    const cmd = `sudo sqlite3 /var/lib/brotherwatch/brotherwatch.db "UPDATE users SET remaining_min = daily_allowance_min, session_active_since = NULL, cooldown_until = NULL, warned = 0; DELETE FROM daily_usage WHERE date = date('now');"`;
    // Eslatma: buyruqni barcha foydalanuvchilar qatori (usernamesiz) ishlatish uchun push qilamiz.
    // addCmd () da currentUser tekshiruvi borligi uchun, nom tanlangan bo'lishi kerak. Shuning uchun to'g'ridan to'g'ri push qilamiz:
    commands.push(cmd);
    render();
    document.getElementById('copyBtn').classList.remove('copied');
    document.getElementById('copyBtn'
).textContent = 'nusxa olish';
  }

  function copyAll(){
    if(commands.length === 0) return;
    const text = commands.join('\n');
    navigator.clipboard.writeText(text).then(()=>{
      const btn = document.getElementById('copyBtn');
      btn.textContent = 'nusxa olindi ✓';
      btn.classList.add('copied');
      setTimeout(()=>{ btn.textContent = 'nusxa olish'; btn.classList.remove('copied'); }, 1800);
    });
  }

  render();
</script>

</body>
</html>
