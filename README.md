"""
YDORABot - version finale
Installation :  pip install python-telegram-bot==21.6 httpx
Variables    :  BOT_TOKEN (obligatoire), LLM_KEY (IA), LLM_URL, LLM_MODEL,
                VIDEO_API_URL, VIDEO_API_KEY (optionnelles)
Lancement    :  python ydora_bot_final.py
  Linux/Mac  :  export BOT_TOKEN="ton_token" LLM_KEY="ta_cle"
  PowerShell :  $env:BOT_TOKEN="ton_token"; $env:LLM_KEY="ta_cle"
"""
import os, json, logging, urllib.parse
import httpx
from telegram import (Update, InlineKeyboardButton, InlineKeyboardMarkup,
                      BotCommand)
from telegram.ext import (Application, CommandHandler, MessageHandler,
                          CallbackQueryHandler, ContextTypes, filters)

BOT_TOKEN = os.environ["BOT_TOKEN"]

# --- IA (toute API compatible OpenAI) ---
# Groq   : https://api.groq.com/openai/v1  | llama-3.3-70b-versatile
# Gemini : https://generativelanguage.googleapis.com/v1beta/openai | gemini-2.0-flash
# OpenAI : https://api.openai.com/v1       | gpt-4o-mini
LLM_URL = os.getenv("LLM_URL", "https://api.groq.com/openai/v1")
LLM_KEY = os.getenv("LLM_KEY", "")
LLM_MODEL = os.getenv("LLM_MODEL", "llama-3.3-70b-versatile")

# --- Vidéo (optionnel) : service image -> vidéo ---
VIDEO_API_URL = os.getenv("VIDEO_API_URL", "")
VIDEO_API_KEY = os.getenv("VIDEO_API_KEY", "")

SYSTEM_PROMPT = ("Tu es Ydora, une assistante créative et chaleureuse. "
                 "Réponds en français, de façon claire et concise.")
DATA_FILE = "users.json"
MAX_HIST = 10
logging.basicConfig(level=logging.INFO)


# ---------- stockage (profils, paramètres, historique) ----------
def default_user():
    return {"profil": {}, "genre": "Romance", "longueur": "Moyenne", "hist": []}

def load():
    try:
        with open(DATA_FILE, encoding="utf-8") as f:
            return json.load(f)
    except FileNotFoundError:
        return {}

def save(data):
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

def user(uid):
    data = load()
    if str(uid) not in data:
        data[str(uid)] = default_user()
        save(data)
    return {**default_user(), **data[str(uid)]}

def update(uid, **kw):
    data = load()
    u = {**default_user(), **data.get(str(uid), {})}
    u.update(kw)
    data[str(uid)] = u
    save(data)


# ---------- IA ----------
async def ask_llm(messages):
    async with httpx.AsyncClient(timeout=90) as c:
        r = await c.post(f"{LLM_URL}/chat/completions",
            headers={"Authorization": f"Bearer {LLM_KEY}"},
            json={"model": LLM_MODEL, "messages": messages})
        r.raise_for_status()
        return r.json()["choices"][0]["message"]["content"]

async def send_long(msg, text):
    for i in range(0, len(text), 4000):  # limite Telegram : 4096
        await msg.reply_text(text[i:i + 4000])


# ---------- commandes ----------
async def start(update_: Update, ctx: ContextTypes.DEFAULT_TYPE):
    name = update_.effective_user.first_name
    await update_.message.reply_text(
        f"Bienvenue {name} 🚀\nJe suis Ydora. Je discute avec toi, j'écris des histoires, "
        "je crée des images et j'anime tes visuels.\n\n"
        "Écris-moi simplement, ou tape /help pour voir mes commandes."
    )

async def help_(update_: Update, ctx: ContextTypes.DEFAULT_TYPE):
    await update_.message.reply_text(
        "📖 Comment m'utiliser :\n\n"
        "💬 Écris-moi directement pour discuter\n"
        "/histoire <idée> — génère une histoire\n"
        "/image <description> — crée une image\n"
        "/video — puis envoie une photo à animer\n"
        "/profil — crée ou consulte ta fiche perso\n"
        "/settings — genre, longueur, effacer la conversation"
    )

async def histoire(update_: Update, ctx: ContextTypes.DEFAULT_TYPE):
    if not LLM_KEY:
        await update_.message.reply_text("⚠️ Clé IA manquante (variable LLM_KEY).")
        return
    idee = " ".join(ctx.args) or "une histoire originale de ton choix"
    u = user(update_.effective_user.id)
    longueurs = {"Courte": "environ 200 mots", "Moyenne": "environ 500 mots", "Longue": "environ 900 mots"}
    await update_.message.reply_text("✍️ J'écris ton histoire...")
    prompt = (f"Écris en français une histoire de genre {u['genre']}, "
              f"{longueurs[u['longueur']]}, sur ce thème : {idee}. Donne-lui un titre.")
    try:
        texte = await ask_llm([{"role": "user", "content": prompt}])
    except Exception as e:
        logging.exception(e)
        await update_.message.reply_text("❌ Impossible de générer l'histoire pour l'instant.")
        return
    await send_long(update_.message, texte)

async def image(update_: Update, ctx: ContextTypes.DEFAULT_TYPE):
    desc = " ".join(ctx.args)
    if not desc:
        await update_.message.reply_text("Utilise : /image un lion dans une forêt magique")
        return
    await update_.message.reply_text("🎨 Je dessine...")
    url = f"https://image.pollinations.ai/prompt/{urllib.parse.quote(desc)}?width=1024&height=1024&nologo=true"
    try:
        async with httpx.AsyncClient(timeout=120, follow_redirects=True) as c:
            r = await c.get(url)
            r.raise_for_status()
        await update_.message.reply_photo(r.content, caption=f"🖼️ {desc}")
    except Exception as e:
        logging.exception(e)
        await update_.message.reply_text("❌ Génération d'image impossible pour l'instant.")

async def video(update_: Update, ctx: ContextTypes.DEFAULT_TYPE):
    ctx.user_data["attend_video"] = True
    await update_.message.reply_text("🎬 Envoie-moi l'image à transformer en vidéo.")

async def on_photo(update_: Update, ctx: ContextTypes.DEFAULT_TYPE):
    if not ctx.user_data.pop("attend_video", False):
        await update_.message.reply_text("Pour animer une image, tape d'abord /video.")
        return
    if not VIDEO_API_URL:
        await update_.message.reply_text(
            "🚧 La génération vidéo n'est pas encore connectée (variable VIDEO_API_URL).")
        return
    await update_.message.reply_text("⏳ Création de la vidéo...")
    photo = await update_.message.photo[-1].get_file()
    img_bytes = bytes(await photo.download_as_bytearray())
    try:
        async with httpx.AsyncClient(timeout=300) as c:
            r = await c.post(VIDEO_API_URL,
                headers={"Authorization": f"Bearer {VIDEO_API_KEY}"},
                files={"image": ("image.jpg", img_bytes, "image/jpeg")})
            r.raise_for_status()
        await update_.message.reply_video(r.content)
    except Exception as e:
        logging.exception(e)
        await update_.message.reply_text("❌ La vidéo n'a pas pu être créée.")


# ---------- profil ----------
QUESTIONS = [("nom", "Ton nom ou pseudo ?"), ("age", "Ton âge ?"),
             ("ville", "Ta ville ?"), ("bio", "Une courte présentation de toi ?")]

async def start_profil(msg, ctx):
    ctx.user_data["profil_step"] = 0
    ctx.user_data["profil_tmp"] = {}
    await msg.reply_text(QUESTIONS[0][1])

async def profil(update_: Update, ctx: ContextTypes.DEFAULT_TYPE):
    p = user(update_.effective_user.id)["profil"]
    if p:
        txt = "👤 Ta fiche :\n\n" + "\n".join(f"• {k.capitalize()} : {v}" for k, v in p.items())
        kb = InlineKeyboardMarkup([[InlineKeyboardButton("✏️ Modifier", callback_data="profil_edit")]])
        await update_.message.reply_text(txt, reply_markup=kb)
    else:
        await start_profil(update_.message, ctx)


# ---------- paramètres ----------
def settings_kb(u):
    def b(label, data, current):
        return InlineKeyboardButton(("✅ " if current == label else "") + label, callback_data=data)
    return InlineKeyboardMarkup([
        [b("Romance", "genre:Romance", u["genre"]), b("Fantastique", "genre:Fantastique", u["genre"])],
        [b("Policier", "genre:Policier", u["genre"]), b("Horreur", "genre:Horreur", u["genre"])],
        [b(l, f"long:{l}", u["longueur"]) for l in ("Courte", "Moyenne", "Longue")],
        [InlineKeyboardButton("🗑️ Effacer la conversation", callback_data="clear_hist")],
    ])

async def settings(update_: Update, ctx: ContextTypes.DEFAULT_TYPE):
    u = user(update_.effective_user.id)
    await update_.message.reply_text("⚙️ Paramètres :", reply_markup=settings_kb(u))

async def on_button(update_: Update, ctx: ContextTypes.DEFAULT_TYPE):
    q = update_.callback_query
    await q.answer()
    uid = q.from_user.id
    if q.data == "profil_edit":
        await start_profil(q.message, ctx)
        return
    if q.data == "clear_hist":
        update(uid, hist=[])
        await q.message.reply_text("🗑️ Conversation effacée.")
        return
    kind, val = q.data.split(":", 1)
    update(uid, **{"genre" if kind == "genre" else "longueur": val})
    await q.edit_message_reply_markup(settings_kb(user(uid)))


# ---------- texte libre : réponses du profil, sinon discussion IA ----------
async def on_text(update_: Update, ctx: ContextTypes.DEFAULT_TYPE):
    step = ctx.user_data.get("profil_step")

    # 1) on est en train de remplir la fiche
    if step is not None:
        key, _ = QUESTIONS[step]
        ctx.user_data["profil_tmp"][key] = update_.message.text
        step += 1
        if step < len(QUESTIONS):
            ctx.user_data["profil_step"] = step
            await update_.message.reply_text(QUESTIONS[step][1])
        else:
            update(update_.effective_user.id, profil=ctx.user_data.pop("profil_tmp"))
            ctx.user_data.pop("profil_step")
            await update_.message.reply_text("✅ Fiche enregistrée ! Tape /profil pour la voir.")
        return

    # 2) discussion avec l'IA
    if not LLM_KEY:
        await update_.message.reply_text("⚠️ Clé IA manquante (variable LLM_KEY). Tape /help pour mes commandes.")
        return
    uid = update_.effective_user.id
    hist = user(uid)["hist"]
    hist.append({"role": "user", "content": update_.message.text})
    hist = hist[-MAX_HIST:]
    await update_.message.chat.send_action("typing")
    try:
        rep = await ask_llm([{"role": "system", "content": SYSTEM_PROMPT}] + hist)
    except Exception as e:
        logging.exception(e)
        await update_.message.reply_text("❌ IA indisponible pour l'instant, réessaie dans un instant.")
        return
    hist.append({"role": "assistant", "content": rep})
    update(uid, hist=hist[-MAX_HIST:])
    await send_long(update_.message, rep)


async def on_error(update_: object, ctx: ContextTypes.DEFAULT_TYPE):
    logging.error("Erreur non gérée", exc_info=ctx.error)

async def post_init(app: Application):
    await app.bot.set_my_commands([
        BotCommand("start", "Démarrer Ydora 🚀"),
        BotCommand("help", "Voir comment utiliser le bot"),
        BotCommand("histoire", "Générer une nouvelle histoire"),
        BotCommand("image", "Créer une image à partir d'un texte"),
        BotCommand("video", "Transformer une image en vidéo"),
        BotCommand("profil", "Créer / voir ma fiche perso"),
        BotCommand("settings", "Paramètres du bot"),
    ])

def main():
    app = Application.builder().token(BOT_TOKEN).post_init(post_init).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("help", help_))
    app.add_handler(CommandHandler("histoire", histoire))
    app.add_handler(CommandHandler("image", image))
    app.add_handler(CommandHandler("video", video))
    app.add_handler(CommandHandler("profil", profil))
    app.add_handler(CommandHandler("settings", settings))
    app.add_handler(CallbackQueryHandler(on_button))
    app.add_handler(MessageHandler(filters.PHOTO, on_photo))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, on_text))
    app.add_error_handler(on_error)
    app.run_polling()

if __name__ == "__main__":
    main()
