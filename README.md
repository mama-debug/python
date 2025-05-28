import telebot
import requests

# 🔐 Токены и настройки
TELEGRAM_BOT_TOKEN = '7605354776:AAEwR4_CUh02a9aQluXznNbCB1hXCRYFoK0'
CRYPTOPAY_API_TOKEN = '405654:AAkMBKRSkuQCjG1fsjtG4czgSR7iX3IS0An'
BOT_USERNAME = 'PaymentAdelMarin_Bot'
CHANNEL_INVITE_LINK = 'https://t.me/+IeInqEaDoBE2NmNi'
PRICE_USD = 150

bot = telebot.TeleBot(TELEGRAM_BOT_TOKEN)

# invoice_id -> user_id
invoice_users = {}
# user_id -> 'ru' or 'en'
user_languages = {}

# Тексты на двух языках
TEXTS = {
    "ru": {
        "welcome": "👋 Привет! Это бот для оплаты доступа к приватному каналу.\n\n🔞 Более 120 порно видео и 200 фото с нами!",
        "pay": "💳 Оплатить приватный канал за $150",
        "after_pay": "После оплаты напиши /check, чтобы получить ссылку на канал.",
        "paid": "✅ Оплата подтверждена!\nВот ссылка на канал:",
        "not_paid": "❌ Оплата пока не найдена. Попробуй позже.",
        "choose_lang": "Выберите язык / Choose your language:",
        "pay_error": "Произошла ошибка при создании ссылки. Попробуй позже.",
    },
    "en": {
        "welcome": "👋 Hi! This bot lets you pay for access to the private channel.\n\n🔞 Over 120 adult videos and 200 photos available!",
        "pay": "💳 Pay $150 for private channel access",
        "after_pay": "After payment, type /check to get the invite link.",
        "paid": "✅ Payment confirmed!\nHere is your link to the channel:",
        "not_paid": "❌ Payment not found yet. Try again later.",
        "choose_lang": "Choose your language / Выберите язык:",
        "pay_error": "Error while creating payment link. Try again later.",
    }
}

# Обработка команды /start
@bot.message_handler(commands=['start'])
def start(message):
    chat_id = message.chat.id
    # Кнопки выбора языка
    markup = telebot.types.ReplyKeyboardMarkup(resize_keyboard=True, one_time_keyboard=True)
    markup.add('🇷🇺 Русский', '🇬🇧 English')
    bot.send_message(chat_id, TEXTS["ru"]["choose_lang"], reply_markup=markup)

# Выбор языка
@bot.message_handler(func=lambda msg: msg.text in ['🇷🇺 Русский', '🇬🇧 English'])
def choose_language(message):
    chat_id = message.chat.id
    lang = 'ru' if 'Рус' in message.text else 'en'
    user_languages[chat_id] = lang

    bot.send_message(chat_id, TEXTS[lang]["welcome"], parse_mode="Markdown", reply_markup=telebot.types.ReplyKeyboardRemove())

    pay_url, invoice_id = create_payment_link(PRICE_USD, chat_id)
    if invoice_id:
        invoice_users[invoice_id] = chat_id
        bot.send_message(chat_id, f"{TEXTS[lang]['pay']}: {pay_url}")
        bot.send_message(chat_id, TEXTS[lang]['after_pay'])
    else:
        bot.send_message(chat_id, TEXTS[lang]['pay_error'])

# Проверка оплаты
@bot.message_handler(commands=['check'])
def check_payment(message):
    chat_id = message.chat.id
    lang = user_languages.get(chat_id, 'ru')
    found = False

    for invoice_id, user_id in invoice_users.items():
        if user_id != chat_id:
            continue
        status = check_invoice_status(invoice_id)
        if status == 'paid':
            bot.send_message(chat_id, f"{TEXTS[lang]['paid']}\n{CHANNEL_INVITE_LINK}")
            found = True
            break

    if not found:
        bot.send_message(chat_id, TEXTS[lang]['not_paid'])

# Генерация ссылки на оплату
def create_payment_link(amount_usd, chat_id):
    headers = {
        "Crypto-Pay-API-Token": CRYPTOPAY_API_TOKEN
    }

    payload = {
        "asset": "USDT",
        "amount": amount_usd,
        "description": "Private channel access",
        "hidden_message": "Thank you for your payment!",
        "paid_btn_name": "openBot",
        "paid_btn_url": f"https://t.me/{BOT_USERNAME}",
        "payload": f"user_{chat_id}"
    }

    response = requests.post("https://pay.crypt.bot/api/createInvoice", json=payload, headers=headers)
    if response.status_code == 200:
        result = response.json()['result']
        return result['pay_url'], result['invoice_id']
    else:
        return None, None

# Проверка статуса оплаты
def check_invoice_status(invoice_id):
    headers = {"Crypto-Pay-API-Token": CRYPTOPAY_API_TOKEN
    }

    response = requests.get("https://pay.crypt.bot/api/getInvoices", headers=headers)
    if response.status_code == 200:
        items = response.json().get("result", {}).get("items", [])
        for invoice in items:
            if invoice['invoice_id'] == invoice_id:
                return invoice['status']
    return "unknown"

bot.polling()
