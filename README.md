# family_bot.io
import telebot
import google.generativeai as genai
import random
from telebot import types

# ВАШИ КЛЮЧИ
TOKEN = ''
GOOGLE_API_KEY = ''


# === Инициализация ===
bot = telebot.TeleBot(TOKEN)

try:
    bot_info = bot.get_me()
    BOT_USERNAME = f"@{bot_info.username}"
    print(f"Информация о боте получена: ID: {bot_info.id}, Имя: {bot_info.first_name}, Username: {BOT_USERNAME}")
except Exception as e:
    print(f"Не удалось получить информацию о боте. Ошибка: {e}")
    exit()

genai.configure(api_key=GOOGLE_API_KEY)
model = genai.GenerativeModel('gemini-1.5-flash-latest')

# Словарь для хранения данных игры для каждого пользователя
user_game_data = {}


# === Настройка меню команд (/) ===
def set_main_menu_commands(bot):
    commands = [
        types.BotCommand("/start", "🚀 Перезапустить бота"),
        types.BotCommand("/menu", "📖 Открыть главное меню"),
        types.BotCommand("/help", "ℹ️ Помощь по командам"),
        types.BotCommand("/describe_image", "🖼️ Описать картинку для ИИ"),
        types.BotCommand("/write_post", "✍️ Написать пост для соцсетей"),
        types.BotCommand("/brainstorm", "💡 Провести мозговой штурм"),
        types.BotCommand("/game", "🎲 Сыграть в 'Угадай число'"),
        types.BotCommand("/fact", "💡 Узнать случайный факт"),
        types.BotCommand("/summarize", "📝 Краткий пересказ текста"),
    ]
    bot.set_my_commands(commands)


# === Функции для создания клавиатур ===
def create_main_menu_keyboard():
    keyboard = types.InlineKeyboardMarkup(row_width=2)
    
    btn_describe_image = types.InlineKeyboardButton(text="🖼️ Описать картинку", callback_data="describe_image")
    btn_write_post = types.InlineKeyboardButton(text="✍️ Написать пост", callback_data="write_post")
    btn_brainstorm = types.InlineKeyboardButton(text="💡 Мозговой штурм", callback_data="brainstorm_ideas")
    btn_ask_ai = types.InlineKeyboardButton(text="❓ Общий вопрос к ИИ", callback_data="ask_ai")
    btn_summarize = types.InlineKeyboardButton(text="📝 Пересказ текста", callback_data="summarize_text")
    btn_fact = types.InlineKeyboardButton(text="💡 Случайный факт", callback_data="get_fact")
    btn_game = types.InlineKeyboardButton(text="🎲 Простая игра", callback_data="play_game")
    btn_help = types.InlineKeyboardButton(text="ℹ️ Помощь", callback_data="help")

    keyboard.add(btn_ask_ai, btn_describe_image, btn_write_post, btn_brainstorm, btn_summarize, btn_fact, btn_game, btn_help)
    return keyboard

# === Обработчики команд ===

@bot.message_handler(commands=['start', 'menu'])
def show_menu(message):
    user_name = message.from_user.first_name
    menu_text = f"Привет, {user_name}! Я твой ИИ-ассистент. Выбери, чем займемся:"
    keyboard = create_main_menu_keyboard()
    bot.send_message(message.chat.id, menu_text, reply_markup=keyboard)

@bot.message_handler(commands=['help'])
def send_help(message):
    help_text = (
        "🤖 *Список доступных команд:*\n\n"
        "/start, /menu - Открыть главное меню.\n"
        "/help - Показать это сообщение.\n\n"
        "--- Творчество и Контент ---\n"
        "/describe_image - Помогает создать промпт (описание) для нейросетей, рисующих картинки.\n"
        "/write_post - Пишет готовый пост для соцсетей на вашу тему.\n"
        "/brainstorm - Генерирует креативные идеи по вашему запросу.\n\n"
        "--- Утилиты и Развлечения ---\n"
        "/summarize - Делает краткий пересказ вашего текста.\n"
        "/fact - Рассказывает интересный случайный факт.\n"
        "/game - Предлагает сыграть в 'Угадай число'.\n\n"
        "💡 Для общего вопроса к ИИ просто напишите его в чат (в группах — с упоминанием меня)."
    )
    bot.send_message(message.chat.id, help_text, parse_mode='Markdown')

# --- Команды, использующие register_next_step_handler ---

@bot.message_handler(commands=['describe_image'])
def ask_for_image_to_describe(message):
    msg = bot.send_message(message.chat.id, "Отлично! Опишите простыми словами, какую картинку вы хотите создать, а я превращу это в профессиональный промпт.")
    bot.register_next_step_handler(msg, process_image_description)

@bot.message_handler(commands=['write_post'])
def ask_for_post_topic(message):
    msg = bot.send_message(message.chat.id, "Какова тема вашего будущего поста? Опишите ее, и я подготовлю текст.")
    bot.register_next_step_handler(msg, generate_social_media_post)

@bot.message_handler(commands=['brainstorm'])
def ask_for_brainstorm_topic(message):
    msg = bot.send_message(message.chat.id, "Сформулируйте проблему или тему для мозгового штурма, и я накину несколько идей.")
    bot.register_next_step_handler(msg, generate_brainstorm_ideas)

@bot.message_handler(commands=['game'])
def start_game(message):
    user_id = message.from_user.id
    secret_number = random.randint(1, 10)
    user_game_data[user_id] = {'secret_number': secret_number, 'attempts': 3}
    msg = bot.send_message(message.chat.id, "Я загадал число от 1 до 10. У тебя 3 попытки. Какое число?")
    bot.register_next_step_handler(msg, process_game_guess)

@bot.message_handler(commands=['summarize'])
def ask_for_text_to_summarize(message):
    msg = bot.send_message(message.chat.id, "Отправьте мне текст или ссылку на статью для краткого пересказа.")
    bot.register_next_step_handler(msg, generate_summary)

# --- Команды, работающие сразу ---

@bot.message_handler(commands=['fact'])
def send_random_fact(message):
    prompt = "Расскажи один интересный и неожиданный научный или исторический факт, о котором мало кто знает. Подай его в увлекательной манере."
    bot.send_message(message.chat.id, "Ищу интересный факт...")
    try:
        response = model.generate_content(prompt)
        bot.send_message(message.chat.id, response.text)
    except Exception as e:
        print(f"ERROR in /fact: {e}")
        bot.send_message(message.chat.id, "Не смог найти факт, попробуйте еще раз.")

# === ПОЛНЫЙ Обработчик нажатий на inline-кнопки ===
@bot.callback_query_handler(func=lambda call: True)
def handle_callback_query(call):
    actions = {
        "describe_image": lambda: ask_for_image_to_describe(call.message),
        "write_post": lambda: ask_for_post_topic(call.message),
        "brainstorm_ideas": lambda: ask_for_brainstorm_topic(call.message),
        "ask_ai": lambda: bot.send_message(call.message.chat.id, f"В этом чате я буду отвечать, только если вы упомянете меня по имени: {BOT_USERNAME}\n\nПросто напишите ваш вопрос в этот чат."),
        "summarize_text": lambda: ask_for_text_to_summarize(call.message),
        "get_fact": lambda: send_random_fact(call.message),
        "play_game": lambda: start_game(call.message),
        "help": lambda: send_help(call.message)
    }
    
    action = actions.get(call.data)
    if action:
        action()

    bot.answer_callback_query(call.id)
    try:
        bot.edit_message_reply_markup(chat_id=call.message.chat.id, message_id=call.message.message_id, reply_markup=None)
    except telebot.apihelper.ApiTelegramException:
        pass # Игнорируем ошибку, если клавиатуру уже нельзя отредактировать

# === Логика для функций, вызываемых через register_next_step_handler ===

def process_image_description(message):
    user_idea = message.text
    if user_idea.startswith('/'): return # Игнорируем случайные команды
    prompt_for_gemini = f"""Преврати простую идею пользователя в высококачественный, детализированный и креативный промпт для нейросети, генерирующей изображения (типа Stable Diffusion или Midjourney). Промпт должен быть на английском языке. Включи в него стиль, освещение, ракурс и живые детали. ВАЖНО: В ответе верни ТОЛЬКО готовый английский промпт, без лишних слов. Идея пользователя: "{user_idea}" """
    thinking_msg = bot.send_message(message.chat.id, "Придумываю идеальный промпт для художника-нейросети...")
    
    try:
        response = model.generate_content(prompt_for_gemini)
        image_prompt = response.text.strip()
        service_url = "https://huggingface.co/spaces/stabilityai/stable-diffusion-3-medium"
        keyboard = types.InlineKeyboardMarkup()
        url_button = types.InlineKeyboardButton(text="🎨 Создать по этому описанию", url=service_url)
        keyboard.add(url_button)
        result_text = (f"Вот профессиональный промпт для нейросети:\n\n```\n{image_prompt}\n```\n\n"
                       f"👆 **Скопируйте** этот текст.\n👇 **Нажмите кнопку**, вставьте его в поле 'Prompt' и творите магию!")
        bot.edit_message_text(result_text, chat_id=thinking_msg.chat.id, message_id=thinking_msg.message_id, 
                              reply_markup=keyboard, parse_mode='Markdown')
    except Exception as e:
        print(f"ERROR in process_image_description: {e}")
        bot.edit_message_text("Что-то пошло не так, не смог придумать промпт.", 
                              chat_id=thinking_msg.chat.id, message_id=thinking_msg.message_id)

def generate_social_media_post(message):
    topic = message.text
    if topic.startswith('/'): return
    prompt = f"Напиши увлекательный и структурированный пост для социальной сети (например, Instagram) на тему '{topic}'. Включи в него: броский заголовок, основной текст из 2-3 абзацев, используй подходящие по смыслу эмодзи, и закончи пост вопросом к аудитории или призывом к действию."
    bot.send_message(message.chat.id, f"✍️ Готовлю пост на тему '{topic}'...")
    try:
        response = model.generate_content(prompt)
        bot.send_message(message.chat.id, response.text, reply_to_message_id=message.message_id)
    except Exception as e:
        print(f"ERROR in generate_social_media_post: {e}")
        bot.send_message(message.chat.id, "Не смог написать пост, попробуйте сформулировать тему иначе.")

def generate_brainstorm_ideas(message):
    topic = message.text
    if topic.startswith('/'): return
    prompt = f"Проведи сессию мозгового штурма на тему '{topic}'. Предложи 5-7 нестандартных, креативных и практичных идей, связанных с этой темой. Оформи их в виде нумерованного списка с кратким пояснением для каждой идеи."
    bot.send_message(message.chat.id, f"💡 Устраиваю мозговой штурм по теме '{topic}'...")
    try:
        response = model.generate_content(prompt)
        bot.send_message(message.chat.id, response.text, reply_to_message_id=message.message_id)
    except Exception as e:
        print(f"ERROR in generate_brainstorm_ideas: {e}")
        bot.send_message(message.chat.id, "Не удалось сгенерировать идеи, попробуйте другую тему.")
        
def generate_summary(message):
    user_text = message.text
    if user_text.startswith('/'): return
    prompt = f"Сделай краткий, но емкий пересказ следующего текста: '{user_text}'"
    bot.send_message(message.chat.id, "Анализирую текст...")
    try:
        response = model.generate_content(prompt)
        bot.send_message(message.chat.id, f"**Краткий пересказ:**\n\n{response.text}", parse_mode='Markdown')
    except Exception as e:
        print(f"ERROR in generate_summary: {e}")
        bot.send_message(message.chat.id, "Не удалось обработать текст.")
        
def process_game_guess(message):
    user_id = message.from_user.id
    if user_id not in user_game_data:
        bot.send_message(message.chat.id, "Сначала начните игру командой /game.")
        return
    if message.text.startswith('/'): return

    try:
        guess = int(message.text)
    except ValueError:
        msg = bot.send_message(message.chat.id, "Это не похоже на число. Пожалуйста, введите число.")
        bot.register_next_step_handler(msg, process_game_guess)
        return

    game = user_game_data[user_id]
    game['attempts'] -= 1

    if guess == game['secret_number']:
        bot.send_message(message.chat.id, f"🎉 Поздравляю, вы угадали! Это было число {game['secret_number']}.")
        del user_game_data[user_id]
    elif game['attempts'] > 0:
        hint = "больше" if guess < game['secret_number'] else "меньше"
        msg = bot.send_message(message.chat.id, f"Неверно. Загаданное число {hint}. Осталось попыток: {game['attempts']}.")
        bot.register_next_step_handler(msg, process_game_guess)
    else:
        bot.send_message(message.chat.id, f"😔 Увы, попытки закончились. Я загадал число {game['secret_number']}.")
        del user_game_data[user_id]

# === ГЛАВНЫЙ ОБРАБОТЧИК ТЕКСТА (ДЛЯ AI) --- ДОЛЖЕН БЫТЬ ПОСЛЕДНИМ ===
@bot.message_handler(content_types=["text"])
def handle_ai_message(message):
    # Эта функция теперь ловит только текст, который не был пойман командами
    is_private_chat = message.chat.type == 'private'
    is_mentioned = BOT_USERNAME in message.text

    # Отвечаем, только если это личка ИЛИ если бота упомянули в группе
    if not (is_private_chat or is_mentioned):
        return

    prompt = message.text.replace(BOT_USERNAME, "").strip() if is_mentioned else message.text
    if not prompt:
        bot.send_message(message.chat.id, "Да, я здесь! Чем могу помочь?", reply_to_message_id=message.message_id)
        return
    
    thinking_message = bot.send_message(message.chat.id, "🤔 Думаю над вашим вопросом...", reply_to_message_id=message.message_id)
    try:
        response = model.generate_content(prompt)
        bot.edit_message_text(response.text, chat_id=thinking_message.chat.id, message_id=thinking_message.message_id)
    except Exception as e:
        print(f"ERROR in handle_ai_message: {e}")
        bot.edit_message_text("Извините, произошла ошибка при обращении к ИИ.", chat_id=thinking_message.chat.id, message_id=thinking_message.message_id)


# === Запуск бота ===
if __name__ == '__main__':
    print("Настройка команд меню...")
    set_main_menu_commands(bot)
    print("Бот запущен...")
    bot.polling(non_stop=True)
