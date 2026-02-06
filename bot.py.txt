import os
import sqlite3
from datetime import datetime, timedelta
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup, KeyboardButton, ReplyKeyboardMarkup
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, ContextTypes, MessageHandler, filters
from enum import Enum

# ==================== НАСТРОЙКИ ====================
BOT_TOKEN = os.environ.get('TELEGRAM_BOT_TOKEN', '')  # ИЗМЕНИЛ НАЗВАНИЕ
MAX_USERS = 30

# Типы перерывов
class BreakType(Enum):
    LUNCH = ("🍽 Обед", 45, 5)
    SMOKE = ("🚬 Перекур", 10, 3)

# ==================== БАЗА ДАННЫХ ====================
def init_db():
    conn = sqlite3.connect('queue.db')
    c = conn.cursor()
    
    c.execute('''CREATE TABLE IF NOT EXISTS users
                 (user_id INTEGER PRIMARY KEY,
                  username TEXT,
                  full_name TEXT,
                  registered_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP)''')
    
    c.execute('''CREATE TABLE IF NOT EXISTS bookings
                 (id INTEGER PRIMARY KEY AUTOINCREMENT,
                  user_id INTEGER,
                  break_type TEXT,
                  start_time TEXT,
                  end_time TEXT,
                  status TEXT DEFAULT 'active',
                  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP)''')
    
    conn.commit()
    conn.close()
    print("✅ База данных инициализирована")

# ==================== КЛАВИАТУРЫ ====================
def get_main_keyboard():
    keyboard = [
        [KeyboardButton("📋 Мои записи"), KeyboardButton("📊 Очередь")],
        [KeyboardButton("🍽 Записать на обед"), KeyboardButton("🚬 Записать на перекур")],
        [KeyboardButton("❌ Отменить запись"), KeyboardButton("📈 Статистика")]
    ]
    return ReplyKeyboardMarkup(keyboard, resize_keyboard=True, one_time_keyboard=False)

# ==================== ВСПОМОГАТЕЛЬНЫЕ ФУНКЦИИ ====================
def register_user(user_id, username, full_name):
    conn = sqlite3.connect('queue.db')
    c = conn.cursor()
    c.execute('''INSERT OR IGNORE INTO users (user_id, username, full_name)
                 VALUES (?, ?, ?)''', (user_id, username, full_name))
    conn.commit()
    conn.close()

def get_current_bookings_count(break_type, start_time):
    conn = sqlite3.connect('queue.db')
    c = conn.cursor()
    c.execute('''SELECT COUNT(*) FROM bookings 
                 WHERE break_type = ? AND start_time = ? AND status = 'active' ''',
              (break_type, start_time))
    count = c.fetchone()[0]
    conn.close()
    return count

# ==================== ОСНОВНЫЕ КОМАНДЫ ====================
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    register_user(user.id, user.username, user.full_name)
    
    await update.message.reply_text(
        f"👋 Привет, {user.first_name}!\n\n"
        "🤖 Я бот для записи на перерывы\n"
        "Используйте кнопки ниже 👇",
        reply_markup=get_main_keyboard()
    )

async def show_my_bookings(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    conn = sqlite3.connect('queue.db')
    c = conn.cursor()
    c.execute('''SELECT break_type, start_time, end_time FROM bookings 
                 WHERE user_id = ? AND status = 'active' ''', (user.id,))
    bookings = c.fetchall()
    conn.close()
    
    if not bookings:
        await update.message.reply_text("📭 У вас нет активных записей")
    else:
        text = "📋 Ваши записи:\n\n"
        for br_type, start_t, end_t in bookings:
            text += f"{br_type}: {start_t}-{end_t}\n"
        await update.message.reply_text(text)

async def show_queue(update: Update, context: ContextTypes.DEFAULT_TYPE):
    conn = sqlite3.connect('queue.db')
    c = conn.cursor()
    c.execute('''SELECT break_type, start_time, end_time, u.full_name 
                 FROM bookings b
                 JOIN users u ON b.user_id = u.user_id
                 WHERE b.status = 'active'
                 ORDER BY start_time''')
    bookings = c.fetchall()
    conn.close()
    
    if not bookings:
        await update.message.reply_text("📭 Очередь пуста")
    else:
        text = "📊 Текущая очередь:\n\n"
        for br_type, start_t, end_t, name in bookings:
            text += f"{br_type}: {start_t}-{end_t} ({name})\n"
        await update.message.reply_text(text)

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    user = update.effective_user
    
    if text == "📋 Мои записи":
        await show_my_bookings(update, context)
    elif text == "📊 Очередь":
        await show_queue(update, context)
    elif text == "🍽 Записать на обед":
        await update.message.reply_text("Функция записи на обед в разработке")
    elif text == "🚬 Записать на перекур":
        await update.message.reply_text("Функция записи на перекур в разработке")
    elif text == "❌ Отменить запись":
        await update.message.reply_text("Функция отмены в разработке")
    elif text == "📈 Статистика":
        conn = sqlite3.connect('queue.db')
        c = conn.cursor()
        c.execute("SELECT COUNT(*) FROM users")
        users = c.fetchone()[0]
        c.execute("SELECT COUNT(*) FROM bookings WHERE status='active'")
        active = c.fetchone()[0]
        conn.close()
        await update.message.reply_text(f"📈 Статистика:\n👥 Пользователей: {users}\n📋 Активных записей: {active}")
    else:
        await update.message.reply_text("Используйте кнопки ниже 👇", reply_markup=get_main_keyboard())

# ==================== ЗАПУСК БОТА ====================
def main():
    init_db()
    
    if not BOT_TOKEN:
        print("❌ ОШИБКА: Токен не найден!")
        print("Добавьте TELEGRAM_BOT_TOKEN в переменные окружения Render")
        return
    
    app = Application.builder().token(BOT_TOKEN).build()
    
    app.add_handler(CommandHandler("start", start))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    
    print("=" * 50)
    print("🤖 БОТ ЗАПУСКАЕТСЯ...")
    print(f"Токен: {'Есть' if BOT_TOKEN else 'НЕТ!'}")
    print("=" * 50)
    
    app.run_polling()

if __name__ == '__main__':
    main() 