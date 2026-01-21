import telebot
import random
import sqlite3
from datetime import datetime
from telebot import types

# Настройки
BOT_TOKEN = '8370874571:AAHWeKrWiDRYkmKt6Ijf-mEJX41iWwAAuH8'  # Замените на токен вашего бота от @BotFather
DATABASE_NAME = 'stars_casino.db'

# Инициализация бота
bot = telebot.TeleBot(BOT_TOKEN)


# Инициализация базы данных
def init_db():
    conn = sqlite3.connect(DATABASE_NAME)
    cursor = conn.cursor()

    # Таблица пользователей
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS users (
        user_id INTEGER PRIMARY KEY,
        username TEXT,
        stars INTEGER DEFAULT 1000,
        daily_last_claimed TEXT,
        created_at TEXT
    )
    ''')

    conn.commit()
    conn.close()


# Получение или создание пользователя
def get_user(user_id, username=None):
    conn = sqlite3.connect(DATABASE_NAME)
    cursor = conn.cursor()

    cursor.execute('SELECT * FROM users WHERE user_id = ?', (user_id,))
    user = cursor.fetchone()

    if not user:
        cursor.execute('''
        INSERT INTO users (user_id, username, stars, created_at) 
        VALUES (?, ?, ?, ?)
        ''', (user_id, username, 1000, datetime.now().isoformat()))
        conn.commit()
        conn.close()
        return {
            'user_id': user_id,
            'username': username,
            'stars': 1000,
            'daily_last_claimed': None,
            'created_at': datetime.now().isoformat()
        }

    conn.close()
    return {
        'user_id': user[0],
        'username': user[1],
        'stars': user[2],
        'daily_last_claimed': user[3],
        'created_at': user[4]
    }


# Обновление звезд
def update_stars(user_id, amount):
    conn = sqlite3.connect(DATABASE_NAME)
    cursor = conn.cursor()

    cursor.execute('SELECT stars FROM users WHERE user_id = ?', (user_id,))
    current = cursor.fetchone()[0]
    new_amount = current + amount

    if new_amount < 0:
        conn.close()
        return False

    cursor.execute('UPDATE users SET stars = ? WHERE user_id = ?', (new_amount, user_id))
    conn.commit()
    conn.close()
    return True


# Клавиатура главного меню
def main_menu():
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
    btn1 = types.KeyboardButton('🎰 Крутить слоты')
    btn2 = types.KeyboardButton('⭐️ Мой баланс')
    btn3 = types.KeyboardButton('🎁 Ежедневный бонус')
    btn4 = types.KeyboardButton('🏆 Топ игроков')
    btn5 = types.KeyboardButton('ℹ️ Помощь')
    markup.add(btn1, btn2, btn3, btn4, btn5)
    return markup


# Ежедневный бонус
def claim_daily_bonus(user_id):
    conn = sqlite3.connect(DATABASE_NAME)
    cursor = conn.cursor()

    cursor.execute('SELECT daily_last_claimed FROM users WHERE user_id = ?', (user_id,))
    last_claimed = cursor.fetchone()[0]

    today = datetime.now().date().isoformat()

    if last_claimed == today:
        conn.close()
        return False

    cursor.execute('UPDATE users SET daily_last_claimed = ?, stars = stars + 500 WHERE user_id = ?',
                   (today, user_id))
    conn.commit()
    conn.close()
    return True


# Игровые символы
SYMBOLS = ['🍒', '🍋', '⭐️', '🔔', '7️⃣', '🍉', '💎', '🍀']


# Функция вращения слотов
def spin_slots(bet_amount):
    reels = [
        random.choices(SYMBOLS, weights=[20, 18, 5, 15, 3, 17, 2, 20], k=3),
        random.choices(SYMBOLS, weights=[20, 18, 5, 15, 3, 17, 2, 20], k=3),
        random.choices(SYMBOLS, weights=[20, 18, 5, 15, 3, 17, 2, 20], k=3)
    ]

    # Отображение для вывода
    display = f"{reels[0][0]} | {reels[1][0]} | {reels[2][0]}\n"
    display += f"{reels[0][1]} | {reels[1][1]} | {reels[2][1]}\n"
    display += f"{reels[0][2]} | {reels[1][2]} | {reels[2][2]}"

    # Проверка выигрышей (горизонтальные линии)
    win_multiplier = 0

    # Проверка центральной линии (самая важная)
    if reels[0][1] == reels[1][1] == reels[2][1]:
        multipliers = {
            '7️⃣': 100,
            '💎': 50,
            '⭐️': 25,
            '🔔': 10,
            '🍀': 8,
            '🍒': 5,
            '🍋': 4,
            '🍉': 3
        }
        win_multiplier = multipliers.get(reels[0][1], 0)

    # Другие линии (упрощенная логика)
    elif reels[0][0] == reels[1][0] == reels[2][0]:
        win_multiplier = 5
    elif reels[0][2] == reels[1][2] == reels[2][2]:
        win_multiplier = 5

    win_amount = bet_amount * win_multiplier

    return {
        'display': display,
        'reels': reels,
        'win': win_amount,
        'multiplier': win_multiplier
    }


# Топ игроков
def get_top_players(limit=10):
    conn = sqlite3.connect(DATABASE_NAME)
    cursor = conn.cursor()

    cursor.execute('SELECT username, stars FROM users ORDER BY stars DESC LIMIT ?', (limit,))
    top = cursor.fetchall()
    conn.close()

    return top


# Обработчик команды /start
@bot.message_handler(commands=['start'])
def start_message(message):
    user = get_user(message.from_user.id, message.from_user.username)

    welcome_text = f"""
👋 Привет, {message.from_user.first_name}!

🎰 Добро пожаловать в Star Slots! 

У тебя есть ⭐️ {user['stars']} звезд для игры.
Получай ежедневный бонус и выигрывай больше звезд!

🎯 Правила:
- Крути слоты со ставкой от 10 звезд
- 3 одинаковых символа в центре = ДЖЕКПОТ!
- Другие линии тоже приносят выигрыш

📌 Используй меню ниже для навигации
    """

    bot.send_message(message.chat.id, welcome_text, reply_markup=main_menu())


# Обработчик кнопки баланса
@bot.message_handler(func=lambda message: message.text == '⭐️ Мой баланс')
def show_balance(message):
    user = get_user(message.from_user.id)
    bot.send_message(message.chat.id, f"💰 Твой баланс: ⭐️ {user['stars']} звезд")


# Обработчик ежедневного бонуса
@bot.message_handler(func=lambda message: message.text == '🎁 Ежедневный бонус')
def daily_bonus(message):
    user_id = message.from_user.id

    if claim_daily_bonus(user_id):
        user = get_user(user_id)
        bot.send_message(message.chat.id,
                         f"🎉 Ты получил ежедневный бонус: ⭐️ 500 звезд!\n💰 Теперь у тебя: ⭐️ {user['stars']} звезд")
    else:
        bot.send_message(message.chat.id, "⏳ Ты уже получал бонус сегодня. Возвращайся завтра!")


# Обработчик топа игроков
@bot.message_handler(func=lambda message: message.text == '🏆 Топ игроков')
def top_players(message):
    top = get_top_players(10)

    if not top:
        bot.send_message(message.chat.id, "📊 Пока нет данных о игроках")
        return

    top_text = "🏆 ТОП-10 ИГРОКОВ:\n\n"
    for i, (username, stars) in enumerate(top, 1):
        name = username if username else f"Игрок {i}"
        top_text += f"{i}. {name} - ⭐️ {stars}\n"

    bot.send_message(message.chat.id, top_text)


# Обработчик помощи
@bot.message_handler(func=lambda message: message.text == 'ℹ️ Помощь')
def help_message(message):
    help_text = """
🎰 *Star Slots - Помощь*

*Как играть:*
1. Нажми "🎰 Крутить слоты"
2. Выбери ставку (от 10 до 500 звезд)
3. Крути и выигрывай!

*Выигрышные комбинации:*
- 3 одинаковых символа в центре: БОЛЬШОЙ ВЫИГРЫШ
- 3 одинаковых в верхней линии: x5
- 3 одинаковых в нижней линии: x5

*Символы и множители:*
7️⃣ — x100
💎 — x50
⭐️ — x25
🔔 — x10
🍀 — x8
🍒 — x5
🍋 — x4
🍉 — x3

*Другие функции:*
🎁 Ежедневный бонус — 500 звезд каждый день
🏆 Топ игроков — соревнуйся с другими

*Важно:* Это развлекательная игра с виртуальной валютой!
    """

    bot.send_message(message.chat.id, help_text, parse_mode='Markdown')


# Обработчик игры в слоты
@bot.message_handler(func=lambda message: message.text == '🎰 Крутить слоты')
def play_slots(message):
    user = get_user(message.from_user.id)

    if user['stars'] < 10:
        bot.send_message(message.chat.id,
                         "❌ У тебя недостаточно звезд. Минимальная ставка: 10 звезд\n🎁 Получи ежедневный бонус!")
        return

    # Создаем клавиатуру для выбора ставки
    markup = types.InlineKeyboardMarkup(row_width=3)
    bets = [10, 50, 100, 250, 500]

    # Показываем только доступные ставки
    available_bets = [bet for bet in bets if bet <= user['stars']]

    if not available_bets:
        bot.send_message(message.chat.id, "❌ У тебя недостаточно звезд для минимальной ставки")
        return

    for bet in available_bets:
        markup.add(types.InlineKeyboardButton(f"⭐️ {bet}", callback_data=f"bet_{bet}"))

    markup.add(types.InlineKeyboardButton("❌ Отмена", callback_data="cancel"))

    bot.send_message(message.chat.id,
                     f"💰 Твой баланс: ⭐️ {user['stars']}\n\nВыбери ставку:",
                     reply_markup=markup)


# Обработчик колбэков (кнопок)
@bot.callback_query_handler(func=lambda call: True)
def callback_handler(call):
    user = get_user(call.from_user.id)

    if call.data == "cancel":
        bot.delete_message(call.message.chat.id, call.message.message_id)
        bot.send_message(call.message.chat.id, "❌ Игра отменена", reply_markup=main_menu())
        return

    if call.data.startswith("bet_"):
        bet_amount = int(call.data.split("_")[1])

        if bet_amount > user['stars']:
            bot.answer_callback_query(call.id, "❌ Недостаточно звезд!")
            return

        # Снимаем ставку
        if not update_stars(call.from_user.id, -bet_amount):
            bot.answer_callback_query(call.id, "❌ Ошибка при списании!")
            return

        # Крутим слоты
        result = spin_slots(bet_amount)

        # Добавляем выигрыш если есть
        if result['win'] > 0:
            update_stars(call.from_user.id, result['win'])

        # Обновляем информацию о пользователе
        user = get_user(call.from_user.id)

        # Создаем сообщение с результатом
        result_text = f"""
🎰 *РЕЗУЛЬТАТ:*

{result['display']}

{'🎉 ДЖЕКПОТ!' if result['multiplier'] >= 25 else '✅ Выигрыш!' if result['win'] > 0 else '❌ Проигрыш'}

Ставка: ⭐️ {bet_amount}
Множитель: x{result['multiplier']}
Выигрыш: ⭐️ {result['win']}
Новый баланс: ⭐️ {user['stars']}
        """

        # Удаляем старое сообщение с кнопками
        bot.delete_message(call.message.chat.id, call.message.message_id)

        # Отправляем результат
        bot.send_message(call.message.chat.id, result_text, parse_mode='Markdown')

        # Добавляем кнопку для повторной игры
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🎰 Крутить еще раз", callback_data="play_again"))

        bot.send_message(call.message.chat.id, "Хочешь сыграть еще?", reply_markup=markup)

    elif call.data == "play_again":
        bot.delete_message(call.message.chat.id, call.message.message_id)
        play_slots(call.message)


# Запуск бота
if __name__ == '__main__':
    print("🎰 Star Slots Bot запущен!")
    init_db()
    bot.polling(none_stop=True)
