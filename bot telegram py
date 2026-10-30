import os, io, re, ast, json, time, wave, base64, secrets, random, asyncio, datetime, operator, requests
from urllib.parse import quote
from telegram import InlineKeyboardButton as B, InlineKeyboardMarkup as M, ReplyKeyboardMarkup
from telegram.ext import (ApplicationBuilder, MessageHandler, CommandHandler,
                          CallbackQueryHandler, filters)

TELEGRAM_TOKEN = os.getenv("TELEGRAM_TOKEN", "9243305663:AAFfd1JA7Ibc94IXDLF56iFXAumBSlEEDFg")
AI_KEY = os.getenv("AI_KEY", "AQ.Ab8RN6JfOEREPvwZLMnMAWkX6xAG6oVdvCEjrr0t25IWKe7q-g")
ADMIN = 8013833941
PUBLIC = False
DAILY_LIMIT = 50
DAILY_HOUR = 8

BASE = "https://generativelanguage.googleapis.com/v1beta/models/"
TEXT_M = "gemini-flash-lite-latest"
IMAGE_M = "gemini-2.5-flash-image"
TTS_M = "gemini-2.5-flash-preview-tts"
FILE = "data.json"

PERSONAS = {
    "friend": ("😊 دوست", "تو یه دستیار دوستانه و باهوشی. فارسی محاوره‌ای و کامل جواب بده."),
    "teacher": ("👨‍🏫 معلم", "تو معلم صبوری هستی. ساده و قدم‌به‌قدم با مثال توضیح بده. فارسی."),
    "translator": ("🌐 مترجم", "تو مترجم حرفه‌ای هستی. فارسی را به انگلیسی و بقیه را به فارسی ترجمه کن. فقط ترجمه."),
    "coder": ("💻 برنامه‌نویس", "تو برنامه‌نویس ارشدی. کد تمیز و توضیح کوتاه بده. فارسی."),
    "funny": ("😂 شوخ", "تو دستیار بامزه و شوخ‌طبعی. با طنز ولی مفید جواب بده. فارسی محاوره‌ای."),
    "english": ("🇬🇧 معلم انگلیسی", "تو معلم انگلیسی هستی. با کاربر انگلیسی ساده حرف بزن، غلط‌هاش را مهربانانه اصلاح کن و معنی فارسی بده."),
    "writer": ("✍️ نویسنده", "تو نویسنده‌ی خلاق و ویراستاری. متن‌های روان، جذاب و بی‌غلط به فارسی بنویس."),
    "marketer": ("📣 بازاریاب", "تو متخصص بازاریابی و تولید محتوای اینستاگرام هستی. کپشن، هشتگ و ایده‌ی خلاقانه بده. فارسی."),
}

QUICK = {
    "joke": "یه جوک کوتاه و بامزه‌ی فارسی بگو.",
    "fal": "یه فال حافظ با بیت شعر و تفسیر کوتاه بگو.",
    "quote": "یه جمله‌ی انگیزشی کوتاه و تازه به فارسی بگو.",
    "price": "قیمت لحظه‌ای دلار، یورو، طلای ۱۸ عیار، سکه امامی و بیت‌کوین رو از اینترنت پیدا کن، کوتاه لیست کن و زمان اطلاعات رو بنویس.",
}

try:
    db = json.load(open(FILE))
except Exception:
    db = {}
for k in ("history", "persona", "notes", "count", "users", "mem", "voice", "game"):
    db.setdefault(k, {})
for k in ("reminders", "allowed", "banned", "daily"):
    db.setdefault(k, [])
db.setdefault("total", 0)
db.setdefault("last_daily", "")
BOT = {"name": ""}
LAST = {"err": ""}

def save():
    json.dump(db, open(FILE, "w"), ensure_ascii=False)

def run(fn, *a):
    return asyncio.to_thread(fn, *a)

def today():
    return datetime.date.today().isoformat()

def is_admin(uid):
    return ADMIN == 0 or uid == ADMIN

def allowed(uid):
    if uid in db["banned"]:
        return False
    return PUBLIC or ADMIN == 0 or uid == ADMIN or uid in db["allowed"]

# ---------- تاریخ شمسی ----------
def g2j(gy, gm, gd):
    gdm = [0, 31, 59, 90, 120, 151, 181, 212, 243, 273, 304, 334]
    gy2 = gy + 1 if gm > 2 else gy
    days = 355666 + 365 * gy + (gy2 + 3) // 4 - (gy2 + 99) // 100 + (gy2 + 399) // 400 + gd + gdm[gm - 1]
    jy = -1595 + 33 * (days // 12053)
    days %= 12053
    jy += 4 * (days // 1461)
    days %= 1461
    if days > 365:
        jy += (days - 1) // 365
        days = (days - 1) % 365
    if days < 186:
        jm, jd = 1 + days // 31, 1 + days % 31
    else:
        jm, jd = 7 + (days - 186) // 30, 1 + (days - 186) % 30
    return jy, jm, jd

MONTHS = ["فروردین", "اردیبهشت", "خرداد", "تیر", "مرداد", "شهریور", "مهر", "آبان", "آذر", "دی", "بهمن", "اسفند"]
DAYS = ["دوشنبه", "سه‌شنبه", "چهارشنبه", "پنجشنبه", "جمعه", "شنبه", "یکشنبه"]

def today_text():
    n = datetime.datetime.now()
    y, m, d = g2j(n.year, n.month, n.day)
    return f"{DAYS[n.weekday()]} {d} {MONTHS[m-1]} {y} | {n.strftime('%Y-%m-%d %H:%M')}"

# ---------- گوگل ----------
def gcall(model, body, timeout=60):
    r = requests.post(BASE + model + ":generateContent",
                      headers={"x-goog-api-key": AI_KEY}, json=body, timeout=timeout)
    return r.status_code, r.json()

def ask(cid, uid, parts, saved, system=None, mem=True):
    cid = str(cid)
    msgs = db["history"].setdefault(cid, [])
    if system is None:
        system = PERSONAS[db["persona"].get(cid, "friend")][1]
        facts = db["mem"].get(str(uid), [])
        if facts:
            system += "\nاطلاعات کاربر: " + "؛ ".join(facts)
        system += "\nتاریخ امروز: " + today_text()
    tools = [{"google_search": {}}]
    if any("http" in (p.get("text") or "") for p in parts):
        tools.append({"url_context": {}})
    body = {"system_instruction": {"parts": [{"text": system}]},
            "contents": (msgs[-20:] if mem else []) + [{"role": "user", "parts": parts}],
            "tools": tools}
    j = None
    for i in range(3):
        try:
            code, j = gcall(TEXT_M, body, 45)
            if code == 429:
                return "سهمیه‌ی رایگان گوگل فعلاً تموم شده، کمی بعد امتحان کن."
            ps = j["candidates"][0]["content"]["parts"]
            ans = "".join(x.get("text", "") for x in ps).strip() or "جوابی نیومد، دوباره بپرس."
            if mem:
                msgs.append({"role": "user", "parts": [{"text": saved}]})
                msgs.append({"role": "model", "parts": [{"text": ans}]})
                save()
            return ans
        except Exception as e:
            LAST["err"] = f"{type(e).__name__}: {str(j)[:200]}"
            time.sleep(3)
    return "گوگل الان جواب نمی‌ده، چند دقیقه دیگه امتحان کن."

def ai(text, system):
    return ask(0, 0, [{"text": text}], "", system, False)

def make_image(prompt, img_b64=None):
    parts = [{"text": prompt}]
    if img_b64:
        parts.insert(0, {"inline_data": {"mime_type": "image/jpeg", "data": img_b64}})
    body = {"contents": [{"parts": parts}],
            "generationConfig": {"responseModalities": ["TEXT", "IMAGE"]}}
    try:
        code, j = gcall(IMAGE_M, body, 90)
        for x in j["candidates"][0]["content"]["parts"]:
            d = x.get("inlineData") or x.get("inline_data")
            if d:
                return base64.b64decode(d["data"]), None
        return None, "عکسی ساخته نشد. توضیح دیگه‌ای امتحان کن."
    except Exception:
        return None, "ساخت عکس ناموفق بود (سهمیه یا اسم مدل عکس)."

def make_tts(text):
    body = {"contents": [{"parts": [{"text": text[:1500]}]}],
            "generationConfig": {"responseModalities": ["AUDIO"],
                                 "speechConfig": {"voiceConfig": {"prebuiltVoiceConfig": {"voiceName": "Kore"}}}}}
    code, j = gcall(TTS_M, body, 90)
    d = j["candidates"][0]["content"]["parts"][0]
    d = (d.get("inlineData") or d.get("inline_data"))["data"]
    buf = io.BytesIO()
    with wave.open(buf, "wb") as w:
        w.setnchannels(1)
        w.setsampwidth(2)
        w.setframerate(24000)
        w.writeframes(base64.b64decode(d))
    buf.seek(0)
    buf.name = "voice.wav"
    return buf

# ---------- آب‌وهوا ----------
WC = {0: "☀️ صاف", 1: "🌤 کم‌ابر", 2: "⛅ نیمه‌ابری", 3: "☁️ ابری", 45: "🌫 مه", 48: "🌫 مه",
      51: "🌦 نم‌نم", 53: "🌦 نم‌نم", 55: "🌦 نم‌نم", 61: "🌧 باران", 63: "🌧 باران", 65: "🌧 باران شدید",
      71: "🌨 برف", 73: "🌨 برف", 75: "❄️ برف شدید", 80: "🌦 رگبار", 81: "🌦 رگبار", 82: "⛈ رگبار شدید",
      95: "⛈ رعد و برق", 96: "⛈ رعد و تگرگ", 99: "⛈ رعد و تگرگ"}

def geo(name):
    g = requests.get("https://geocoding-api.open-meteo.com/v1/search",
                     params={"name": name, "count": 1, "language": "fa"}, timeout=20).json()
    return g["results"][0] if g.get("results") else None

def forecast(lat, lon, title):
    w = requests.get("https://api.open-meteo.com/v1/forecast", params={
        "latitude": lat, "longitude": lon, "timezone": "auto",
        "current": "temperature_2m,relative_humidity_2m,wind_speed_10m,weather_code",
        "daily": "temperature_2m_max,temperature_2m_min"}, timeout=20).json()
    c, d = w["current"], w["daily"]
    return (f"📍 {title}\n{WC.get(c['weather_code'], '🌡')}\n"
            f"🌡 دما: {c['temperature_2m']}° (امروز {d['temperature_2m_min'][0]}° تا {d['temperature_2m_max'][0]}°)\n"
            f"💧 رطوبت: {c['relative_humidity_2m']}٪\n💨 باد: {c['wind_speed_10m']} km/h")

def weather(city):
    r = geo(city)
    if not r:
        eng = ai(city, "فقط نام انگلیسی این شهر را بنویس، بدون هیچ توضیح اضافه.").strip()
        r = geo(eng)
    if not r:
        return "شهر پیدا نشد."
    return forecast(r["latitude"], r["longitude"], r["name"])

# ---------- ماشین‌حساب ----------
OPS = {ast.Add: operator.add, ast.Sub: operator.sub, ast.Mult: operator.mul,
       ast.Div: operator.truediv, ast.Pow: operator.pow, ast.Mod: operator.mod, ast.USub: operator.neg}

def ev(n):
    if isinstance(n, ast.Constant) and isinstance(n.value, (int, float)):
        return n.value
    if isinstance(n, ast.BinOp):
        a, b = ev(n.left), ev(n.right)
        if isinstance(n.op, ast.Pow) and abs(b) > 100:
            raise ValueError
        return OPS[type(n.op)](a, b)
    if isinstance(n, ast.UnaryOp):
        return OPS[type(n.op)](ev(n.operand))
    raise ValueError

# ---------- ابزارهای عمومی ----------
async def gate(update, count=True):
    u = update.effective_user
    if not u:
        return False
    if not allowed(u.id):
        if update.effective_chat.type == "private":
            await update.effective_message.reply_text("این ربات خصوصیه 🔒")
        return False
    if count:
        c = db["count"].get(str(u.id))
        if not c or c["d"] != today():
            c = {"d": today(), "n": 0}
            db["count"][str(u.id)] = c
        if not is_admin(u.id) and c["n"] >= DAILY_LIMIT:
            await update.effective_message.reply_text("سهمیه‌ی امروزت تموم شد. فردا برگرد 🙏")
            return False
        c["n"] += 1
        db["total"] += 1
        db["users"][str(u.id)] = u.first_name or "?"
        save()
    return True

async def send(update, text):
    for i in range(0, max(len(text), 1), 4000):
        await update.effective_message.reply_text(text[i:i + 4000])

async def reply_ans(update, ans):
    await send(update, ans)
    if db["voice"].get(str(update.effective_chat.id)):
        try:
            buf = await run(make_tts, ans)
            await update.effective_message.reply_audio(audio=buf)
        except Exception:
            pass

def addressed(update):
    m = update.message
    if m.chat.type == "private":
        return True
    r = m.reply_to_message
    if r and r.from_user and r.from_user.username == BOT["name"]:
        return True
    return ("@" + BOT["name"]) in (m.text or m.caption or "")

def menu(cid):
    ps = [B(v[0], callback_data="p:" + k) for k, v in PERSONAS.items()]
    rows = [ps[i:i + 2] for i in range(0, len(ps), 2)]
    vo = "🔊 صدا: روشن" if db["voice"].get(str(cid)) else "🔇 صدا: خاموش"
    rows += [
        [B("😂 جوک", callback_data="x:joke"), B("🔮 فال", callback_data="x:fal"), B("💪 انگیزه", callback_data="x:quote")],
        [B("🧠 کوییز", callback_data="x:quiz"), B("💵 قیمت‌ها", callback_data="x:price"), B("📅 تاریخ", callback_data="x:date")],
        [B(vo, callback_data="vo"), B("📝 یادداشت‌ها", callback_data="notes")],
        [B("🗑 گفتگوی تازه", callback_data="new"), B("❓ راهنما", callback_data="help")],
    ]
    return M(rows)

# دکمه‌های پایین صفحه
KB = ReplyKeyboardMarkup(
    [["🎭 منو", "🗑 گفتگوی تازه"],
     ["😂 جوک", "🔮 فال", "💪 انگیزه"],
     ["💵 قیمت‌ها", "📅 تاریخ", "❓ راهنما"]],
    resize_keyboard=True)

HELP = """📖 راهنما
💬 هر چی بنویسی جواب می‌دم (عکس، ویس، ویدیو، PDF و فایل متنی هم می‌فهمم)
/menu منو و شخصیت‌ها
/new گفتگوی تازه
/image توضیح  ساخت عکس
عکس با کپشن «/edit توضیح»  ویرایش عکس
عکس با کپشن «/ocr»  استخراج متن عکس
/say متن  تبدیل متن به صدا
/voice  روشن/خاموش کردن جواب صوتی
/translate /summary /fix متن
/search سؤال  جستجوی اینترنت
/weather شهر  (یا لوکیشن بفرست)
/price قیمت دلار و طلا و...
/date تاریخ شمسی
/calc 2+2 یا تبدیل واحد
/password 16  ساخت رمز
/qr متن  ساخت QR
/remember متن  حافظه‌ی شخصی
/memory ، /forget
/note ، /notes ، /delnote 1
/remind 10m متن  (s m h d)
/remindlist ، /delremind 1
/joke /fal /quote /quiz /guess
/daily  پیام روزانه
ادمین: /stats /broadcast /ban /unban /adduser /deluser /backup"""

# ---------- دستورات ----------
async def cmd_id(update, ctx):
    await send(update, f"آیدی تو: {update.effective_user.id}")

async def cmd_start(update, ctx):
    if await gate(update, False):
        await update.effective_message.reply_text(
            "سلام! 👋 از دکمه‌های پایین استفاده کن یا مستقیم بپرس.\n/help راهنما", reply_markup=KB)
        await update.effective_message.reply_text("شخصیت‌ها:", reply_markup=menu(update.effective_chat.id))

async def cmd_menu(update, ctx):
    if await gate(update, False):
        await update.effective_message.reply_text("منو:", reply_markup=menu(update.effective_chat.id))

async def cmd_help(update, ctx):
    if await gate(update, False):
        await send(update, HELP)

async def cmd_new(update, ctx):
    if await gate(update, False):
        db["history"][str(update.effective_chat.id)] = []
        save()
        await send(update, "حافظه پاک شد ✅")

async def cmd_voice(update, ctx):
    if await gate(update, False):
        c = str(update.effective_chat.id)
        db["voice"][c] = not db["voice"].get(c)
        save()
        await send(update, "جواب صوتی " + ("روشن شد 🔊" if db["voice"][c] else "خاموش شد 🔇"))

async def cmd_date(update, ctx):
    if await gate(update, False):
        await send(update, "📅 " + today_text())

def arg_or_reply(update, ctx):
    t = " ".join(ctx.args)
    r = update.effective_message.reply_to_message
    if not t and r:
        t = r.text or r.caption or ""
    return t

async def tool(update, ctx, system, hint="بعد از دستور متن بنویس، یا روی یه پیام ریپلای کن."):
    if not await gate(update):
        return
    t = arg_or_reply(update, ctx)
    if not t:
        await send(update, hint)
        return
    ans = await run(ai, t, system)
    await reply_ans(update, ans)

async def cmd_translate(update, ctx):
    await tool(update, ctx, "متن را ترجمه کن: فارسی به انگلیسی، غیر فارسی به فارسی. فقط ترجمه.")

async def cmd_summary(update, ctx):
    await tool(update, ctx, "متن (یا لینک) را به فارسی در چند خط کوتاه خلاصه کن.")

async def cmd_fix(update, ctx):
    await tool(update, ctx, "غلط‌های املایی و نگارشی متن را اصلاح کن. فقط متن اصلاح‌شده را بنویس.")

async def cmd_search(update, ctx):
    await tool(update, ctx, "با جستجوی اینترنت جواب به‌روز و دقیق بده و منبع‌ها را کوتاه نام ببر. فارسی.",
               "مثال: /search آخرین خبر فوتبال")

async def cmd_calc(update, ctx):
    if not await gate(update):
        return
    t = " ".join(ctx.args)
    if not t:
        await send(update, "مثال: /calc 12*(3+4)  یا  /calc 5 مایل چند کیلومتره")
        return
    try:
        e = t.replace("^", "**").replace("×", "*").replace("÷", "/")
        await send(update, f"🧮 {ev(ast.parse(e, mode='eval').body)}")
    except Exception:
        await send(update, await run(ai, t, "ماشین‌حساب و مبدل واحد هستی. فقط جواب نهایی کوتاه بده. فارسی."))

async def cmd_weather(update, ctx):
    if not await gate(update):
        return
    c = " ".join(ctx.args)
    if not c:
        await send(update, "مثال: /weather تهران\nیا لوکیشنت رو بفرست.")
        return
    await send(update, await run(weather, c))

async def on_location(update, ctx):
    if await gate(update):
        l = update.message.location
        await send(update, await run(forecast, l.latitude, l.longitude, "موقعیت شما"))

async def cmd_password(update, ctx):
    if await gate(update, False):
        try:
            n = max(8, min(64, int(ctx.args[0])))
        except Exception:
            n = 16
        chars = "abcdefghijkmnopqrstuvwxyzABCDEFGHJKLMNPQRSTUVWXYZ23456789!@#$%&*"
        await send(update, "🔐 " + "".join(secrets.choice(chars) for _ in range(n)))

async def cmd_qr(update, ctx):
    if await gate(update, False):
        t = arg_or_reply(update, ctx)
        if not t:
            await send(update, "مثال: /qr https://example.com")
            return
        await update.effective_message.reply_photo(
            "https://api.qrserver.com/v1/create-qr-code/?size=500x500&data=" + quote(t))

async def cmd_say(update, ctx):
    if not await gate(update):
        return
    t = arg_or_reply(update, ctx)
    if not t:
        await send(update, "مثال: /say سلام چطوری؟")
        return
    try:
        await update.effective_message.reply_audio(audio=await run(make_tts, t))
    except Exception:
        await send(update, "تبدیل به صدا ناموفق بود (سهمیه یا اسم مدل صدا).")

async def cmd_image(update, ctx):
    if not await gate(update):
        return
    p = " ".join(ctx.args)
    if not p:
        await send(update, "مثال: /image گربه‌ی فضانورد روی ماه")
        return
    await send(update, "دارم می‌سازم... 🎨")
    img, err = await run(make_image, p)
    if img:
        await update.effective_message.reply_photo(photo=img)
    else:
        await send(update, err)

# ---------- حافظه‌ی شخصی و یادداشت ----------
async def cmd_remember(update, ctx):
    if await gate(update, False):
        t = " ".join(ctx.args)
        if not t:
            await send(update, "مثال: /remember اسم من امیره و برنامه‌نویسم")
            return
        db["mem"].setdefault(str(update.effective_user.id), []).append(t)
        save()
        await send(update, "یادم موند 🧠")

async def cmd_memory(update, ctx):
    if await gate(update, False):
        m = db["mem"].get(str(update.effective_user.id), [])
        await send(update, "\n".join(f"{i+1}. {t}" for i, t in enumerate(m)) or "چیزی ذخیره نشده.")

async def cmd_forget(update, ctx):
    if await gate(update, False):
        db["mem"][str(update.effective_user.id)] = []
        save()
        await send(update, "حافظه‌ی شخصی پاک شد ✅")

def notes_text(cid):
    n = db["notes"].get(str(cid), [])
    return "\n".join(f"{i+1}. {t}" for i, t in enumerate(n)) or "یادداشتی نداری."

async def cmd_note(update, ctx):
    if await gate(update, False):
        t = " ".join(ctx.args)
        if not t:
            await send(update, "مثال: /note فردا خرید نون")
            return
        db["notes"].setdefault(str(update.effective_chat.id), []).append(t)
        save()
        await send(update, "ذخیره شد 📝")

async def cmd_notes(update, ctx):
    if await gate(update, False):
        await send(update, notes_text(update.effective_chat.id))

async def cmd_delnote(update, ctx):
    if await gate(update, False):
        try:
            db["notes"].get(str(update.effective_chat.id), []).pop(int(ctx.args[0]) - 1)
            save()
            await send(update, "حذف شد ✅")
        except Exception:
            await send(update, "مثال: /delnote 1")

# ---------- یادآور ----------
async def cmd_remind(update, ctx):
    if not await gate(update, False):
        return
    try:
        w = ctx.args[0].lower()
        sec = int(w[:-1]) * {"s": 1, "m": 60, "h": 3600, "d": 86400}[w[-1]]
        text = " ".join(ctx.args[1:]) or "یادآوری!"
    except Exception:
        await send(update, "مثال: /remind 10m قرص بخور")
        return
    db["reminders"].append({"t": time.time() + sec, "chat": update.effective_chat.id, "text": text})
    save()
    await send(update, "باشه، یادآوری می‌کنم ⏰")

def my_rem(cid):
    return [r for r in db["reminders"] if r["chat"] == cid]

async def cmd_remindlist(update, ctx):
    if await gate(update, False):
        rs = my_rem(update.effective_chat.id)
        await send(update, "\n".join(f"{i+1}. {r['text']} ({int((r['t']-time.time())/60)} دقیقه‌ی دیگه)"
                                     for i, r in enumerate(rs)) or "یادآوری فعالی نداری.")

async def cmd_delremind(update, ctx):
    if await gate(update, False):
        try:
            db["reminders"].remove(my_rem(update.effective_chat.id)[int(ctx.args[0]) - 1])
            save()
            await send(update, "حذف شد ✅")
        except Exception:
            await send(update, "مثال: /delremind 1")

async def cmd_daily(update, ctx):
    if await gate(update, False):
        c = 
