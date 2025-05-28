import telebot
import requests

TELEGRAM_BOT_TOKEN = '7605354776:AAEwR4_CUh02a9aQluXznNbCB1hXCRYFoK0'
CRYPTOPAY_API_TOKEN = '405654:AAkMBKRSkuQCjG1fsjtG4czgSR7iX3IS0An'
BOT_USERNAME = 'https://t.me/PaymentAdelMarin_Bot'
PRICE_USD = 150  # Новая цена

bot = telebot.TeleBot(TELEGRAM_BOT_TOKEN)

@bot.message_handler(commands=['start'])
def start(message):
    chat_id = message.chat.id
    bot.send_message(chat_id, "Привет! Для оплаты нажми на кнопку ниже.")
    payment_link = create_payment_link(PRICE_USD, chat_id)
    bot.send_message(chat_id, f"💸 Оплатить $150: {payment_link}")

def create_payment_link(amount_usd, chat_id):
    headers = {
        "Content-Type": "application/json",
        "Crypto-Pay-API-Token": CRYPTOPAY_API_TOKEN
    }

    payload = {
        "asset": "USDT",
        "amount": amount_usd,
        "currency": "USD",
        "description": "Оплата товара",
        "hidden_message": f"Спасибо за оплату!",
        "callback_url": "https://example.com/payment-callback",
        "payload": f"user_{chat_id}"
    }

    response = requests.post("https://pay.crypt.bot/api/createInvoice", json=payload, headers=headers)

    if response.status_code == 200:
        invoice = response.json()['result']
        return invoice['pay_url']
    else:
        return "Ошибка при создании ссылки на оплату"

bot.polling()
