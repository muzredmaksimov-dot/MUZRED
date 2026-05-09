import os
import uuid
import json
import logging
from datetime import datetime, timedelta
from dotenv import load_dotenv
import gspread
from oauth2client.service_account import ServiceAccountCredentials
from telegram import (
    Update, ReplyKeyboardMarkup, KeyboardButton,
    LabeledPrice
)
from telegram.ext import (
    Application, CommandHandler, MessageHandler,
    PreCheckoutQueryHandler, ContextTypes, filters
)

load_dotenv()

logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

BOT_TOKEN = os.getenv('BOT_TOKEN')
EDITOR_CHAT_ID = os.getenv('EDITOR_CHAT_ID')
GOOGLE_CREDENTIALS = os.getenv('GOOGLE_CREDENTIALS')  # Одна переменная со всем JSON
GOOGLE_SHEET_ID = os.getenv('GOOGLE_SHEET_ID')
PDF_FILE_PATH = os.getenv('PDF_FILE_PATH', './files/18-kriteriev.pdf')
CHECKLIST_PRICE_XTR = int(os.getenv('CHECKLIST_PRICE_XTR', '250'))
REVIEW_PRICE_XTR = int(os.getenv('REVIEW_PRICE_XTR', '500'))

# Google Sheets — инициализация из одной переменной
scope = ['https://spreadsheets.google.com/feeds', 'https://www.googleapis.com/auth/drive']
creds_dict = json.loads(GOOGLE_CREDENTIALS)  # Парсим JSON из переменной
creds = ServiceAccountCredentials.from_json_keyfile_dict(creds_dict, scope)
client = gspread.authorize(creds)
sheet = client.open_by_key(GOOGLE_SHEET_ID)
orders_sheet = sheet.worksheet('Заказы')
stats_sheet = sheet.worksheet('Статистика')

user_states = {}

WELCOME_TEXT = """Привет. Я музыкальный редактор. За годы работы я сравнил больше 1000 хитов и выделил закономерности, которые повторяются из трека в трек.

Я упаковал их в систему проверки. Выберите, что нужно:

🆓 Топ-5 ошибок — почему трек не берут. Бесплатно.
📋 Чек-лист редактора — 18 критериев.
🎧 Заказать разбор — я лично проверю ваш трек.

Оплата через Telegram Stars. Точную цену в вашей валюте покажет Telegram."""

FREE_GUIDE_TEXT = """5 ПРИЧИН, ПОЧЕМУ ТРЕК НЕ БЕРУТ В ПЛЕЙЛИСТЫ

1. Пустое интро.
Трек начинается с тишины, шума или долгой раскачки. У вас есть 5 секунд, чтобы редактор не переключил.

2. Нет хука.
После прослушивания в голове пусто. Если трек не напевают — его не распространяют.

3. Вокал не разобрать.
На студийных мониторах слышно. На телефоне — каша.

4. Трек безликий.
Его можно перепутать с любым другим. Нет уникального маркера.

5. Прислали сырую демку.
Без сведения, без мастеринга. Редактор не доделывает — он закрывает.

---

Эти пять ошибок — начало. Полный список из 18 критериев, по которым я проверяю треки, — в платном чек-листе.

📋 Нажмите «Чек-лист», чтобы получить полный документ.
🎧 Нажмите «Разбор трека», чтобы я проверил ваш трек лично."""


def get_main_keyboard():
    keyboard = [
        [KeyboardButton("📉 Топ-5 ошибок")],
        [KeyboardButton("📋 Чек-лист")],
        [KeyboardButton("🎧 Разбор трека")]
    ]
    return ReplyKeyboardMarkup(keyboard, resize_keyboard=True)


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(WELCOME_TEXT, reply_markup=get_main_keyboard())


async def get_id(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(f"Ваш chat_id: {update.effective_user.id}")


async def stats_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if str(update.effective_user.id) != EDITOR_CHAT_ID:
        await update.message.reply_text("⛔ Доступ запрещён")
        return
    try:
        records = orders_sheet.get_all_records()
        total_checklists = sum(1 for r in records if r.get('order_type') == 'checklist' and r.get('status') == 'paid')
        total_reviews = sum(1 for r in records if r.get('order_type') == 'review' and r.get('status') in ['paid', 'file_uploaded', 'in_progress', 'completed'])
        rev_checklists = sum(int(r.get('amount', 0)) for r in records if r.get('order_type') == 'checklist' and r.get('status') == 'paid')
        rev_reviews = sum(int(r.get('amount', 0)) for r in records if r.get('order_type') == 'review' and r.get('status') in ['paid', 'file_uploaded', 'in_progress', 'completed'])
        await update.message.reply_text(
            f"📊 Статистика:\n\n📋 Чек-листов: {total_checklists}\n🎧 Разборов: {total_reviews}\n💰 Выручка: {rev_checklists + rev_reviews} XTR"
        )
    except Exception as e:
        logger.error(f"Stats error: {e}")


async def pending_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if str(update.effective_user.id) != EDITOR_CHAT_ID:
        await update.message.reply_text("⛔ Доступ запрещён")
        return
    try:
        records = orders_sheet.get_all_records()
        pending = []
        for order in records:
            if order.get('status') in ['file_uploaded', 'in_progress']:
                order_id = order.get('order_id', '')
                username = order.get('username', '')
                file_link = order.get('file_link', '')
                created = order.get('created_at', '')
                deadline = order.get('deadline', '')
                overdue = ""
                if deadline:
                    try:
                        if datetime.now() > datetime.fromisoformat(deadline):
                            overdue = " ⚠️ ПРОСРОЧЕН"
                    except:
                        pass
                pending.append(f"🔹 ID: {order_id}\n   @{username}\n   Файл: {file_link}\n   Создан: {created}\n   Дедлайн: {deadline}{overdue}\n")
        if not pending:
            await update.message.reply_text("✅ Нет активных заказов")
        else:
            text = "🔔 АКТИВНЫЕ ЗАКАЗЫ:\n\n" + "\n".join(pending) + "\nОтвет: /reply [ID] [текст]"
            if len(text) > 4000:
                for i in range(0, len(text), 4000):
                    await update.message.reply_text(text[i:i+4000])
            else:
                await update.message.reply_text(text)
    except Exception as e:
        logger.error(f"Pending error: {e}")


async def reply_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if str(update.effective_user.id) != EDITOR_CHAT_ID:
        await update.message.reply_text("⛔ Доступ запрещён")
        return
    try:
        parts = update.message.text.split(' ', 2)
        if len(parts) < 3:
            await update.message.reply_text("Использование: /reply [ID_заказа] [текст]")
            return
        order_id = parts[1]
        feedback = parts[2]
        records = orders_sheet.get_all_records()
        for i, order in enumerate(records, start=2):
            if str(order.get('order_id')) == order_id:
                user_id = order.get('user_id')
                if user_id:
                    await context.bot.send_message(
                        chat_id=int(user_id),
                        text=f"✅ Готов разбор вашего трека:\n\n{feedback}\n\n---\nВопросы? Напишите сюда."
                    )
                    orders_sheet.update(f'I{i}', feedback)
                    orders_sheet.update(f'F{i}', 'completed')
                    orders_sheet.update(f'L{i}', datetime.now().isoformat())
                    await update.message.reply_text(f"✅ Фидбек по заказу {order_id} отправлен")
                    return
        await update.message.reply_text(f"❌ Заказ {order_id} не найден")
    except Exception as e:
        logger.error(f"Reply error: {e}")


async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    user_id = update.effective_user.id

    if text == "📉 Топ-5 ошибок":
        await update.message.reply_text(FREE_GUIDE_TEXT, reply_markup=get_main_keyboard())
    elif text == "📋 Чек-лист":
        await context.bot.send_invoice(
            chat_id=update.effective_chat.id,
            title="Чек-лист музыкального редактора",
            description="18 критериев проверки трека. PDF.",
            payload="checklist",
            provider_token="",
            currency="XTR",
            prices=[LabeledPrice("Чек-лист", CHECKLIST_PRICE_XTR)]
        )
    elif text == "🎧 Разбор трека":
        await context.bot.send_invoice(
            chat_id=update.effective_chat.id,
            title="Разбор трека редактором",
            description="Личный фидбек по 18 критериям. Срок — до 48 часов.",
            payload="review",
            provider_token="",
            currency="XTR",
            prices=[LabeledPrice("Разбор трека", REVIEW_PRICE_XTR)]
        )
    elif user_id in user_states and user_states[user_id] == 'waiting_for_file':
        if text.startswith('http'):
            await process_file_link(update, context, text)
        else:
            await update.message.reply_text("❌ Неподдерживаемый формат. Пришлите WAV, MP3 или ссылку на облако.", reply_markup=get_main_keyboard())
    else:
        await update.message.reply_text("Используйте кнопки меню.", reply_markup=get_main_keyboard())


async def precheckout_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.pre_checkout_query
    if query.invoice_payload not in ["checklist", "review"]:
        await query.answer(ok=False, error_message="Ошибка. Попробуйте снова.")
        return
    await query.answer(ok=True)


async def successful_payment_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    payment = update.message.successful_payment
    user_id = update.effective_user.id
    username = update.effective_user.username or "no_username"
    first_name = update.effective_user.first_name or "no_name"
    payload = payment.invoice_payload
    amount = payment.total_amount
    charge_id = payment.telegram_payment_charge_id
    order_id = str(uuid.uuid4())[:8]
    now = datetime.now().isoformat()

    if payload == "checklist":
        try:
            with open(PDF_FILE_PATH, 'rb') as f:
                await update.message.reply_document(
                    document=f, filename="18-kriteriev.pdf",
                    caption="✅ Готово! Ваш чек-лист.\n\nПройдите по всем 18 пунктам. Сомневаетесь? Закажите разбор — кнопка «🎧 Разбор трека».",
                    reply_markup=get_main_keyboard()
                )
        except FileNotFoundError:
            await update.message.reply_text("✅ Оплата получена! Файл временно недоступен — напишите администратору.", reply_markup=get_main_keyboard())
        save_order(order_id, user_id, username, first_name, "checklist", "paid", amount, "", now, "", charge_id)
        update_stats("checklist", amount)

    elif payload == "review":
        await update.message.reply_text(
            "✅ Оплата получена! Загрузите трек.\n\n• WAV или MP3 (до 20 МБ)\n• Или ссылка на облако\n\nОтправьте файл или ссылку прямо в чат.",
            reply_markup=get_main_keyboard()
        )
        user_states[user_id] = 'waiting_for_file'
        context.user_data['pending_order_id'] = order_id
        context.user_data['pending_amount'] = amount
        context.user_data['pending_charge_id'] = charge_id
        context.user_data['pending_created'] = now


async def handle_document(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    if user_id not in user_states or user_states[user_id] != 'waiting_for_file':
        await update.message.reply_text("⚠️ Сначала оплатите разбор — кнопка «🎧 Разбор трека».", reply_markup=get_main_keyboard())
        return
    file = update.message.document or update.message.audio
    if not file:
        await update.message.reply_text("❌ Пришлите WAV/MP3 или ссылку.", reply_markup=get_main_keyboard())
        return
    file_name = file.file_name or f"track_{user_id}.wav"
    safe_name = f"{user_id}_{datetime.now().strftime('%Y%m%d_%H%M%S')}_{file_name}"
    file_path = f"./uploads/{safe_name}"
    try:
        os.makedirs('./uploads', exist_ok=True)
        tg_file = await context.bot.get_file(file.file_id)
        await tg_file.download_to_drive(file_path)
        await process_uploaded_file(update, context, user_id, file_path)
    except Exception as e:
        logger.error(f"File error: {e}")
        await update.message.reply_text("❌ Ошибка сохранения. Попробуйте ещё раз или отправьте ссылку.", reply_markup=get_main_keyboard())


async def process_file_link(update: Update, context: ContextTypes.DEFAULT_TYPE, link: str):
    await process_uploaded_file(update, context, update.effective_user.id, link)


async def process_uploaded_file(update: Update, context: ContextTypes.DEFAULT_TYPE, user_id: int, file_path: str):
    username = update.effective_user.username or "no_username"
    first_name = update.effective_user.first_name or "no_name"
    order_id = context.user_data.get('pending_order_id', str(uuid.uuid4())[:8])
    amount = context.user_data.get('pending_amount', REVIEW_PRICE_XTR)
    charge_id = context.user_data.get('pending_charge_id', '')
    created = context.user_data.get('pending_created', datetime.now().isoformat())
    deadline = (datetime.now() + timedelta(hours=48)).isoformat()

    save_order(order_id, user_id, username, first_name, "review", "file_uploaded", amount, file_path, created, deadline, charge_id)
    update_stats("review", amount)

    await update.message.reply_text("✅ Файл получен. Ответ придёт в течение 48 часов.\n\nПока ждёте — проверьте трек по чек-листу: кнопка «📋 Чек-лист».", reply_markup=get_main_keyboard())

    editor_msg = f"🔔 НОВЫЙ ЗАКАЗ\n\nID: {order_id}\nПользователь: @{username}\nСумма: {amount} XTR\nФайл: {file_path}\nДедлайн: {deadline}\n\n/reply {order_id} [текст]"
    try:
        await context.bot.send_message(chat_id=EDITOR_CHAT_ID, text=editor_msg)
    except Exception as e:
        logger.error(f"Notify editor error: {e}")

    if user_id in user_states:
        del user_states[user_id]
    context.user_data.clear()


def save_order(order_id, user_id, username, first_name, order_type, status, amount, file_link, created_at, deadline, charge_id=""):
    try:
        orders_sheet.append_row([order_id, user_id, username, first_name, order_type, status, amount, file_link, charge_id, created_at, deadline, ""])
    except Exception as e:
        logger.error(f"Save order error: {e}")


def update_stats(order_type, amount):
    try:
        records = stats_sheet.get_all_records()
        today = datetime.now().strftime('%Y-%m-%d')
        for i, record in enumerate(records, start=2):
            if record.get('Дата') == today:
                if order_type == 'checklist':
                    stats_sheet.update(f'B{i}', int(record.get('Продано чек-листов', 0)) + 1)
                else:
                    stats_sheet.update(f'C{i}', int(record.get('Продано разборов', 0)) + 1)
                stats_sheet.update(f'D{i}', int(record.get('Выручка (XTR)', 0)) + amount)
                return
        stats_sheet.append_row([today, 1 if order_type == 'checklist' else 0, 1 if order_type == 'review' else 0, amount])
    except Exception as e:
        logger.error(f"Stats error: {e}")


def main():
    application = Application.builder().token(BOT_TOKEN).build()
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("id", get_id))
    application.add_handler(CommandHandler("stats", stats_command))
    application.add_handler(CommandHandler("pending", pending_command))
    application.add_handler(CommandHandler("reply", reply_command))
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    application.add_handler(MessageHandler(filters.Document.ALL | filters.AUDIO, handle_document))
    application.add_handler(PreCheckoutQueryHandler(precheckout_callback))
    application.add_handler(MessageHandler(filters.SUCCESSFUL_PAYMENT, successful_payment_callback))
    logger.info("Бот запущен!")
    application.run_polling(allowed_updates=Update.ALL_TYPES)


if __name__ == "__main__":
    main()
