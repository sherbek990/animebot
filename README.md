import logging
import sqlite3
from datetime import datetime
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, MessageHandler, filters, ContextTypes
import asyncio

# Log sozlamalari
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

# Database sozlamalari
DB_NAME = "bot_database.db"

def init_db():
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    
    # Adminlar jadvali
    cursor.execute('''CREATE TABLE IF NOT EXISTS admins
                     (user_id INTEGER PRIMARY KEY, username TEXT, added_date TEXT)''')
    
    # Kanallar jadvali
    cursor.execute('''CREATE TABLE IF NOT EXISTS channels
                     (channel_id TEXT PRIMARY KEY, 
                      channel_username TEXT, 
                      channel_type TEXT, 
                      added_date TEXT)''')
    
    # Kinolar jadvali
    cursor.execute('''CREATE TABLE IF NOT EXISTS movies
                     (id INTEGER PRIMARY KEY AUTOINCREMENT,
                      title TEXT,
                      description TEXT,
                      file_id TEXT,
                      movie_type TEXT,
                      parts INTEGER DEFAULT 1,
                      added_date TEXT)''')
    
    # Foydalanuvchilar jadvali
    cursor.execute('''CREATE TABLE IF NOT EXISTS users
                     (user_id INTEGER PRIMARY KEY, 
                      username TEXT, 
                      first_name TEXT, 
                      joined_date TEXT)''')
    
    conn.commit()
    conn.close()
    
    # Bitta asosiy admin qo'shamiz (123456789 o'rniga o'zingizning ID ingizni qo'ying)
    add_admin(8382153274, "Asosiy Admin")

# Database funksiyalari
def add_admin(user_id, username):
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("INSERT OR REPLACE INTO admins (user_id, username, added_date) VALUES (?, ?, ?)",
                   (user_id, username, datetime.now().isoformat()))
    conn.commit()
    conn.close()

def is_admin(user_id):
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("SELECT user_id FROM admins WHERE user_id=?", (user_id,))
    result = cursor.fetchone()
    conn.close()
    return result is not None

def add_channel(channel_id, channel_username, channel_type):
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("INSERT OR REPLACE INTO channels (channel_id, channel_username, channel_type, added_date) VALUES (?, ?, ?, ?)",
                   (channel_id, channel_username, channel_type, datetime.now().isoformat()))
    conn.commit()
    conn.close()

def get_all_channels():
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("SELECT channel_id, channel_username, channel_type FROM channels")
    channels = cursor.fetchall()
    conn.close()
    return channels

def add_movie(title, description, file_id, movie_type, parts=1):
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("INSERT INTO movies (title, description, file_id, movie_type, parts, added_date) VALUES (?, ?, ?, ?, ?, ?)",
                   (title, description, file_id, movie_type, parts, datetime.now().isoformat()))
    conn.commit()
    conn.close()

def get_all_movies():
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("SELECT id, title, description, movie_type FROM movies")
    movies = cursor.fetchall()
    conn.close()
    return movies

def add_user(user_id, username, first_name):
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("INSERT OR REPLACE INTO users (user_id, username, first_name, joined_date) VALUES (?, ?, ?, ?)",
                   (user_id, username, first_name, datetime.now().isoformat()))
    conn.commit()
    conn.close()

def get_all_users():
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("SELECT user_id FROM users")
    users = cursor.fetchall()
    conn.close()
    return [user[0] for user in users]

# Start komandasi
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    add_user(user.id, user.username, user.first_name)
    
    if is_admin(user.id):
        keyboard = [
            [InlineKeyboardButton("🎬 Kino qo'shish", callback_data="add_movie")],
            [InlineKeyboardButton("📢 Xabar yuborish", callback_data="broadcast")],
            [InlineKeyboardButton("📺 Kanallar", callback_data="channels")],
            [InlineKeyboardButton("📊 Statistika", callback_data="stats")]
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        await update.message.reply_text(
            f"Salom Admin {user.first_name}! 🎉\n\n"
            "Bot boshqaruv paneliga xush kelibsiz!",
            reply_markup=reply_markup
        )
    else:
        await update.message.reply_text(
            f"Salom {user.first_name}! 👋\n\n"
            "Bu bot orqali yangi filmlar va seriallarni topishingiz mumkin!"
        )

# Kanal qo'shish
async def add_channel_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_admin(update.effective_user.id):
        await update.message.reply_text("❌ Sizda admin huquqi yo'q!")
        return
    
    if len(context.args) < 2:
        await update.message.reply_text(
            "❌ Foydalanish: /addchannel <channel_id> <type>\n\n"
            "📝 Typelar: public, request, optional\n"
            "📌 Misol: /addchannel -100123456789 public"
        )
        return
    
    channel_id = context.args[0]
    channel_type = context.args[1].lower()
    
    if channel_type not in ['public', 'request', 'optional']:
        await update.message.reply_text("❌ Noto'g'ri type! public, request yoki optional bo'lishi kerak")
        return
    
    add_channel(channel_id, "unknown", channel_type)
    await update.message.reply_text(f"✅ Kanal qo'shildi!\nID: {channel_id}\nTuri: {channel_type}")

# Kino qo'shish - boshlanishi
async def add_movie_start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    keyboard = [
        [InlineKeyboardButton("🎥 Oddiy Kino", callback_data="movie_single")],
        [InlineKeyboardButton("📺 Serial", callback_data="movie_series")],
        [InlineKeyboardButton("🔙 Orqaga", callback_data="main_menu")]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    
    await query.edit_message_text(
        "🎬 Kino qo'shish turini tanlang:",
        reply_markup=reply_markup
    )

# Oddiy kino qo'shish
async def add_single_movie(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    await query.edit_message_text(
        "🎥 Oddiy kino qo'shish:\n\n"
        "Iltimos, kino haqida ma'lumotlarni quyidagi formatda yuboring:\n\n"
        "Nomi\n"
        "Tavsifi\n\n"
        "📌 Misol:\n"
        "Inception\n"
        "Dom Cobb - eng yaxshi xo'rov, odamlarning ongiga kirib, ularning sirlarini o'g'irlaydi."
    )
    context.user_data['waiting_for'] = 'single_movie_info'

# Serial qo'shish
async def add_series_movie(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    await query.edit_message_text(
        "📺 Serial qo'shish:\n\n"
        "Iltimos, serial haqida ma'lumotlarni quyidagi formatda yuboring:\n\n"
        "Nomi\n"
        "Tavsifi\n"
        "Qismlar soni\n\n"
        "📌 Misol:\n"
        "Breaking Bad\n"
        "Kimyo o'qituvchisi Walter White saraton kasalligiga chalingach, metamfetamin ishlab chiqarishni boshlaydi.\n"
        "5"
    )
    context.user_data['waiting_for'] = 'series_movie_info'

# Xabar yuborish
async def broadcast_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    await query.edit_message_text(
        "📢 Barcha foydalanuvchilarga xabar yuborish:\n\n"
        "Iltimos, yubormoqchi bo'lgan xabaringizni yuboring:"
    )
    context.user_data['waiting_for'] = 'broadcast_message'

# Kanallar ro'yxati
async def channels_list(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    channels = get_all_channels()
    if not channels:
        text = "📺 Hozircha hech qanday kanal qo'shilmagan"
    else:
        text = "📺 Qo'shilgan kanallar:\n\n"
        for i, (channel_id, username, channel_type) in enumerate(channels, 1):
            text += f"{i}. ID: {channel_id}\n   Turi: {channel_type}\n\n"
    
    keyboard = [[InlineKeyboardButton("🔙 Orqaga", callback_data="main_menu")]]
    reply_markup = InlineKeyboardMarkup(keyboard)
    
    await query.edit_message_text(text, reply_markup=reply_markup)

# Statistika
async def show_stats(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    users_count = len(get_all_users())
    movies_count = len(get_all_movies())
    channels_count = len(get_all_channels())
    
    text = (
        f"📊 Bot statistikasi:\n\n"
        f"👥 Foydalanuvchilar: {users_count}\n"
        f"🎬 Kinolar: {movies_count}\n"
        f"📺 Kanallar: {channels_count}\n"
        f"🕐 Sana: {datetime.now().strftime('%Y-%m-%d %H:%M')}"
    )
    
    keyboard = [[InlineKeyboardButton("🔙 Orqaga", callback_data="main_menu")]]
    reply_markup = InlineKeyboardMarkup(keyboard)
    
    await query.edit_message_text(text, reply_markup=reply_markup)

# Asosiy menyu
async def main_menu(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    keyboard = [
        [InlineKeyboardButton("🎬 Kino qo'shish", callback_data="add_movie")],
        [InlineKeyboardButton("📢 Xabar yuborish", callback_data="broadcast")],
        [InlineKeyboardButton("📺 Kanallar", callback_data="channels")],
        [InlineKeyboardButton("📊 Statistika", callback_data="stats")]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    
    await query.edit_message_text(
        "🏠 Bosh menyu:",
        reply_markup=reply_markup
    )

# Xabarlarni qayta ishlash
async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_admin(update.effective_user.id):
        return
    
    user_data = context.user_data
    waiting_for = user_data.get('waiting_for')
    
    if waiting_for == 'broadcast_message':
        message_text = update.message.text
        users = get_all_users()
        
        await update.message.reply_text(f"📤 Xabar {len(users)} ta foydalanuvchiga yuborilmoqda...")
        
        success = 0
        failed = 0
        
        for i in range(0, len(users), 100):
            batch = users[i:i + 100]
            tasks = []
            
            for user_id in batch:
                try:
                    task = context.bot.send_message(
                        chat_id=user_id,
                        text=message_text
                    )
                    tasks.append(task)
                except Exception:
                    failed += 1
                    continue
            
            results = await asyncio.gather(*tasks, return_exceptions=True)
            
            for result in results:
                if isinstance(result, Exception):
                    failed += 1
                else:
                    success += 1
            
            await asyncio.sleep(0.5)
        
        await update.message.reply_text(
            f"✅ Xabar yuborish yakunlandi!\n\n"
            f"✅ Muvaffaqiyatli: {success}\n"
            f"❌ Xatolik: {failed}"
        )
        
        user_data['waiting_for'] = None
    
    elif waiting_for in ['single_movie_info', 'series_movie_info']:
        lines = update.message.text.split('\n')
        
        if len(lines) < 2:
            await update.message.reply_text("❌ Noto'g'ri format! Iltimos, ko'rsatilgan formatda yuboring.")
            return
        
        title = lines[0].strip()
        description = lines[1].strip()
        parts = 1
        
        if waiting_for == 'series_movie_info':
            if len(lines) < 3:
                await update.message.reply_text("❌ Qismlar soni ko'rsatilmagan!")
                return
            try:
                parts = int(lines[2].strip())
            except ValueError:
                await update.message.reply_text("❌ Noto'g'ri qismlar soni!")
                return
        
        user_data['movie_data'] = {
            'title': title,
            'description': description,
            'parts': parts,
            'type': 'single' if waiting_for == 'single_movie_info' else 'series'
        }
        
        await update.message.reply_text(
            f"✅ Ma'lumotlar qabul qilindi!\n\n"
            f"📝 Nomi: {title}\n"
            f"📄 Tavsifi: {description}\n"
            f"🎬 Turi: {'Oddiy kino' if waiting_for == 'single_movie_info' else 'Serial'}\n"
            f"📦 Qismlar: {parts}\n\n"
            f"Iltimos, endi video faylini yuboring:"
        )
        
        user_data['waiting_for'] = 'movie_file'
    
    elif waiting_for == 'movie_file':
        if update.message.video:
            file_id = update.message.video.file_id
            movie_data = user_data.get('movie_data')
            
            if movie_data:
                add_movie(
                    title=movie_data['title'],
                    description=movie_data['description'],
                    file_id=file_id,
                    movie_type=movie_data['type'],
                    parts=movie_data['parts']
                )
                
                await update.message.reply_text(
                    f"✅ Kino muvaffaqiyatli qo'shildi!\n\n"
                    f"📝 Nomi: {movie_data['title']}\n"
                    f"🎬 Turi: {movie_data['type']}\n"
                    f"📦 Qismlar: {movie_data['parts']}"
                )
                
                user_data['waiting_for'] = None
                user_data['movie_data'] = None
        else:
            await update.message.reply_text("❌ Iltimos, video fayl yuboring!")

# Callback query handler
async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    data = query.data
    
    handlers = {
        'add_movie': add_movie_start,
        'movie_single': add_single_movie,
        'movie_series': add_series_movie,
        'broadcast': broadcast_message,
        'channels': channels_list,
        'stats': show_stats,
        'main_menu': main_menu
    }
    
    if data in handlers:
        await handlers[data](update, context)

# Botni ishga tushirish
def main():
    # Database ni ishga tushirish (avtomatik bitta admin qo'shiladi)
    init_db()
    
    # Bot tokenini o'rnating
    TOKEN = "8028224751:AAEFTNP3yJyG4f_RS7NsJ9dtjCZjmsbLRHc"
    
    application = Application.builder().token(TOKEN).build()
    
    # Command handlers
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("addchannel", add_channel_cmd))
    
    # Callback query handler
    application.add_handler(CallbackQueryHandler(button_handler))
    
    # Message handler
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    application.add_handler(MessageHandler(filters.VIDEO, handle_message))
    
    # Botni ishga tushirish
    application.run_polling()
    print("Bot ishga tushdi...")
    print("Admin ID: 123456789 (o'zgartiring)")

if __name__ == '__main__':
    main()
