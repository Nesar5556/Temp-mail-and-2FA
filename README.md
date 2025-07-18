# from telegram import Update, ReplyKeyboardMarkup
from telegram.ext import (
    ApplicationBuilder, CommandHandler, ContextTypes,
    MessageHandler, filters, ConversationHandler
)
import requests, random, re, time, pyotp

from keep_alive import keep_alive

BOT_TOKEN = "7566975820:AAFJ4uDF6r4qM0HLPK03Q-_2qUEUApAAYME"  # এখানে তোমার Token বসানো হলো

ASK_SECRET = range(1)
user_emails = {}

first_names = ["rakibul", "tamim", "nasima", "sumon", "nodi", "tanvir", "mehedi", "raisa"]
last_names = ["rahman", "islam", "khan", "ahmed", "sarker", "hasan", "uddin", "faruk"]

def generate_email():
    name = random.choice(first_names)
    lname = random.choice(last_names)
    sep = random.choice([".", "_", ""])
    digits = str(random.randint(10, 99))
    return f"{name}{sep}{lname}{digits}@mailto.plus"

def get_tempmail_inbox(email):
    url = f"https://tempmail.plus/api/mails?email={email}"
    try:
        res = requests.get(url)
        return res.json()
    except:
        return []

def read_mail_and_extract_code(mail_id):
    url = f"https://tempmail.plus/api/mail/{mail_id}"
    try:
        res = requests.get(url).json()
        content = res.get("text", "")
        match = re.search(r"\b\d{6}\b", content)
        if match:
            return match.group()
        else:
            return "❗Code পাওয়া যায়নি।"
    except:
        return "📭 মেইল পড়তে সমস্যা হয়েছে।"

keyboard = [
    ["📧 Temp Mail", "🔐 Get 2FA Code"],
    ["📥 Inbox"]
]
reply_markup = ReplyKeyboardMarkup(keyboard, resize_keyboard=True)

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("👋 স্বাগতম! নিচের অপশন বেছে নিন:", reply_markup=reply_markup)

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.message.from_user.id
    text = update.message.text

    if text == "📧 Temp Mail":
        email = generate_email()
        user_emails[user_id] = email
        await update.message.reply_text(f"📧 Temp Mail:\n`{email}`\n⏳ মেইল চেক হচ্ছে...", parse_mode="Markdown")
        time.sleep(10)
        inbox = get_tempmail_inbox(email)
        if inbox:
            mail_id = inbox[0].get("id")
            code = read_mail_and_extract_code(mail_id)
            await update.message.reply_text(f"✅ মেইল এসেছে!\n📨 কোড: `{code}`", parse_mode="Markdown")
        else:
            await update.message.reply_text("📭 কোনো মেইল পাওয়া যায়নি। পরে '📥 Inbox' দিয়ে রিফ্রেশ করো।")

    elif text == "📥 Inbox":
        if user_id in user_emails:
            email = user_emails[user_id]
            inbox = get_tempmail_inbox(email)
            if inbox:
                mail_id = inbox[0].get("id")
                code = read_mail_and_extract_code(mail_id)
                await update.message.reply_text(f"📨 আপডেটেড কোড: `{code}`", parse_mode="Markdown")
            else:
                await update.message.reply_text("📭 এখনো কোনো নতুন মেইল নেই।")
        else:
            await update.message.reply_text("⚠️ আগে '📧 Temp Mail' চাপো।")

    elif text == "🔐 Get 2FA Code":
        await update.message.reply_text(
            "✅ আপনার 2FA Secret Key send করুন!\n(Example - `ABCD EFGH IGK84 LM44 NSER3`)",
            parse_mode="Markdown"
        )
        return ASK_SECRET

async def handle_secret(update: Update, context: ContextTypes.DEFAULT_TYPE):
    secret = update.message.text.strip().replace(" ", "")
    try:
        code = pyotp.TOTP(secret).now()
        await update.message.reply_text(f"🔐 আপনার 2FA কোড: `{code}`", parse_mode="Markdown")
    except:
        await update.message.reply_text("❌ Invalid Secret Key! আবার চেষ্টা করো।")
    return ConversationHandler.END

keep_alive()

app = ApplicationBuilder().token(BOT_TOKEN).build()

conv_handler = ConversationHandler(
    entry_points=[MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message)],
    states={ASK_SECRET: [MessageHandler(filters.TEXT & ~filters.COMMAND, handle_secret)]},
    fallbacks=[]
)

app.add_handler(CommandHandler("start", start))
app.add_handler(conv_handler)

print("🤖 Bot is running...")
app.run_polling()
