import telebot
from telebot.types import InlineKeyboardMarkup, InlineKeyboardButton
import smtplib
import sqlite3
import re
import logging
import time
from datetime import datetime, timedelta

# إعداد السجل
logging.basicConfig(level=logging.INFO)

# توكن البوت
bot_token = '8121910129:AAEuTVHFz_wkyZXYW2DbP3Nkgolr33UiyQY'  # استبدل بـ توكن البوت الخاص بك
bot = telebot.TeleBot(bot_token)

# معرف المطور
DEVELOPER_ID = 7783444698

# إعداد قاعدة البيانات
conn = sqlite3.connect('users.db', check_same_thread=False)
cursor = conn.cursor()
cursor.execute('''
    CREATE TABLE IF NOT EXISTS users (
        user_id INTEGER, 
        account_name TEXT, 
        email TEXT, 
        app_password TEXT, 
        PRIMARY KEY(user_id, email)
    )
''')
cursor.execute('''
    CREATE TABLE IF NOT EXISTS allowed_users (
        user_id INTEGER PRIMARY KEY,
        expiry_date TEXT
    )
''')
cursor.execute('''
    CREATE TABLE IF NOT EXISTS interacted_users (
        user_id INTEGER PRIMARY KEY
    )
''')
conn.commit()

# إرسال واجهة البداية
def send_welcome(chat_id, first_name):
    text = f"""👋 أهلاً {first_name}!
🌟 هذا البوت سيساعدك في فك الحظر عن أرقام الهواتف والقنوات والمجموعات بكل سهولة. 
يُرجى اختيار أحد الخيارات أدناه للبدء.

تم تطوير البوت من قبل 🎭:
Alihatem: @bxnxkcb

نسخة البوت 👾: 25.1.3
"""
     
    
    
    markup = InlineKeyboardMarkup(row_width=2)
    markup.add(
        InlineKeyboardButton("📧 إضافة حساب", callback_data="add_account"),
        InlineKeyboardButton("👤 حساباتي", callback_data="my_accounts"),
        InlineKeyboardButton("🗑️ حذف حساب", callback_data="delete_account"),
        InlineKeyboardButton("📞 فك رقم محظور", callback_data="unblock_number"),
        InlineKeyboardButton("📡 فك حظر قناة", callback_data="unblock_channel"),
        InlineKeyboardButton("💬 فك حظر مجموعة", callback_data="unblock_group"),
        InlineKeyboardButton("✉️ إرسال رسالة حرة", callback_data="custom_message")
    )
    if chat_id == DEVELOPER_ID:
        markup.add(InlineKeyboardButton("⚙️ إعدادات المطور", callback_data="developer_settings"))
    bot.send_message(chat_id, text, reply_markup=markup)

@bot.message_handler(commands=['start'])
def handle_start(message):
    user_id = message.from_user.id
    bot.clear_step_handler_by_chat_id(message.chat.id)  # إلغاء أي أمر سابق
    cursor.execute("INSERT OR IGNORE INTO interacted_users (user_id) VALUES (?)", (user_id,))
    conn.commit()
    if user_id == DEVELOPER_ID:  # إذا كان المستخدم هو المطور
        send_welcome(message.chat.id, message.from_user.first_name)
    else:  # إذا كان مستخدمًا عاديًا
        cursor.execute("SELECT expiry_date FROM allowed_users WHERE user_id=?", (user_id,))
        user = cursor.fetchone()
        if user and datetime.now() < datetime.fromisoformat(user[0]):
            send_welcome(message.chat.id, message.from_user.first_name)
        else:
            bot.send_message(message.chat.id, "❌ ليس لديك إذن لاستخدام هذا البوت راسل المطور لتتمكن من استخدم البوت ( @bxnxkcb )")

# إعدادات المطور
@bot.callback_query_handler(func=lambda call: call.data == "developer_settings")
def developer_settings(call):
    if call.from_user.id == DEVELOPER_ID:
        bot.delete_message(call.message.chat.id, call.message.message_id)  # حذف الرسالة القديمة
        markup = InlineKeyboardMarkup()
        markup.add(InlineKeyboardButton("📢 إرسال إعلان", callback_data="send_announcement"))
        markup.add(InlineKeyboardButton("➕ إضافة مستخدم", callback_data="add_user"))
        markup.add(InlineKeyboardButton("➖ حذف مستخدم", callback_data="remove_user"))
        markup.add(InlineKeyboardButton("👤 معلومات مستخدم", callback_data="user_info"))
        markup.add(InlineKeyboardButton("🔙 رجوع إلى القائمة الرئيسية", callback_data="main_menu"))
        bot.send_message(call.message.chat.id, "👨‍💻اعدادات المطور", reply_markup=markup)
    else:
        bot.send_message(call.message.chat.id, "❌ ليس لديك إذن للوصول إلى هذه الإعدادات.")

# إرسال إعلان
@bot.callback_query_handler(func=lambda call: call.data == "send_announcement")
def send_announcement(call):
    if call.from_user.id == DEVELOPER_ID:
        bot.delete_message(call.message.chat.id, call.message.message_id)  # حذف الرسالة القديمة
        msg = bot.send_message(call.message.chat.id, "✏️ أدخل نص الإعلان:")
        bot.register_next_step_handler(msg, process_announcement)
    else:
        bot.send_message(call.message.chat.id, "❌ ليس لديك إذن للوصول إلى هذه الإعدادات.")

def process_announcement(message):
    announcement_text = message.text
    cursor.execute("SELECT user_id FROM interacted_users")
    users = cursor.fetchall()
    for user in users:
        try:
            bot.send_message(user[0], f"📢 إعلان:\n\n{announcement_text}")
        except Exception as e:
            logging.error(f"فشل إرسال الإعلان إلى {user[0]}: {e}")
    bot.send_message(message.chat.id, "✅ تم إرسال الإعلان إلى جميع المستخدمين.")

# معلومات المستخدم
@bot.callback_query_handler(func=lambda call: call.data == "user_info")
def user_info(call):
    if call.from_user.id == DEVELOPER_ID:
        bot.delete_message(call.message.chat.id, call.message.message_id)  # حذف الرسالة القديمة
        msg = bot.send_message(call.message.chat.id, "✏️ أدخل معرف المستخدم (User ID):")
        bot.register_next_step_handler(msg, process_user_info)
    else:
        bot.send_message(call.message.chat.id, "❌ ليس لديك إذن للوصول إلى هذه الإعدادات.")

def process_user_info(message):
    try:
        user_id = int(message.text)
        cursor.execute("SELECT expiry_date FROM allowed_users WHERE user_id=?", (user_id,))
        user = cursor.fetchone()
        if user:
            expiry_date = datetime.fromisoformat(user[0])
            remaining_days = (expiry_date - datetime.now()).days
            text = f"👨‍💼 ID: {user_id}\n"
            text += f"⏱️ المدة: {remaining_days} يوم\n"
            text += f"⏱️ المدة المتبقية: {remaining_days} يوم\n"
            markup = InlineKeyboardMarkup()
            markup.add(InlineKeyboardButton("🔙 رجوع إلى القائمة الرئيسية", callback_data="main_menu"))
            bot.send_message(message.chat.id, text, reply_markup=markup)
        else:
            bot.send_message(message.chat.id, "❌ لم يتم العثور على المستخدم.")
    except ValueError:
        bot.send_message(message.chat.id, "❌ أدخل معرف مستخدم صحيح.")

# إضافة مستخدم
@bot.callback_query_handler(func=lambda call: call.data == "add_user")
def add_user(call):
    if call.from_user.id == DEVELOPER_ID:
        bot.delete_message(call.message.chat.id, call.message.message_id)  # حذف الرسالة القديمة
        msg = bot.send_message(call.message.chat.id, "✏️ أدخل معرف المستخدم (User ID):")
        bot.register_next_step_handler(msg, process_add_user)
    else:
        bot.send_message(call.message.chat.id, "❌ ليس لديك إذن للوصول إلى هذه الإعدادات.")

def process_add_user(message):
    try:
        user_id = int(message.text)
        msg = bot.send_message(message.chat.id, "✏️ أدخل المدة بالأيام:")
        bot.register_next_step_handler(msg, lambda m: process_add_user_duration(user_id, m.text))
    except ValueError:
        bot.send_message(message.chat.id, "❌ أدخل معرف مستخدم صحيح.")

def process_add_user_duration(user_id, duration_text):
    try:
        duration = int(duration_text)
        expiry_date = datetime.now() + timedelta(days=duration)
        cursor.execute("INSERT OR REPLACE INTO allowed_users (user_id, expiry_date) VALUES (?, ?)",
                       (user_id, expiry_date.isoformat()))
        conn.commit()
        bot.send_message(DEVELOPER_ID, f"✅ تم إضافة المستخدم {user_id} لمدة {duration} أيام.")
    except ValueError:
        bot.send_message(DEVELOPER_ID, "❌ أدخل مدة صحيحة بالأيام.")

# حذف مستخدم
@bot.callback_query_handler(func=lambda call: call.data == "remove_user")
def remove_user(call):
    if call.from_user.id == DEVELOPER_ID:
        bot.delete_message(call.message.chat.id, call.message.message_id)  # حذف الرسالة القديمة
        msg = bot.send_message(call.message.chat.id, "✏️ أدخل معرف المستخدم (User ID):")
        bot.register_next_step_handler(msg, process_remove_user)
    else:
        bot.send_message(call.message.chat.id, "❌ ليس لديك إذن للوصول إلى هذه الإعدادات.")

def process_remove_user(message):
    try:
        user_id = int(message.text)
        cursor.execute("DELETE FROM allowed_users WHERE user_id=?", (user_id,))
        conn.commit()
        bot.send_message(message.chat.id, "✅ تم حذف المستخدم بنجاح.")
    except ValueError:
        bot.send_message(message.chat.id, "❌ أدخل معرف مستخدم صحيح.")

# باقي الكود (بدون تغييرات)
@bot.callback_query_handler(func=lambda call: True)
def callback_handler(call):
    bot.delete_message(call.message.chat.id, call.message.message_id)
    data = call.data.split('|')
    action = data[0]
    email = data[1] if len(data) > 1 else None
    extra_data = data[2] if len(data) > 2 else None

    if action == "add_account":
        msg = bot.send_message(call.message.chat.id, "✨ أدخل اسم الحساب:")
        bot.register_next_step_handler(msg, save_account_name)
    elif action == "my_accounts":
        show_accounts(call.message.chat.id)
    elif action == "delete_account":
        msg = bot.send_message(call.message.chat.id, "🗑️ أدخل اسم الحساب أو البريد الإلكتروني لحذفه:")
        bot.register_next_step_handler(msg, delete_account)
    elif action in ["unblock_number", "unblock_channel", "unblock_group", "custom_message"]:
        process_step(call.message.chat.id, action)
    elif action == "main_menu":
        send_welcome(call.message.chat.id, "المستخدم")
    elif action.startswith("send_"):
        send_email_action(call.message.chat.id, email, action, extra_data)

# باقي الوظائف (بدون تغييرات)
def save_account_name(message):
    account_name = message.text
    msg = bot.send_message(message.chat.id, "📧 أدخل بريد Gmail:")
    bot.register_next_step_handler(msg, lambda m: save_email(m, account_name))

def save_email(message, account_name):
    email = message.text
    if not re.match(r"^[^@]+@gmail\.com$", email):
        msg = bot.send_message(message.chat.id, "❌ أدخل بريد Gmail صالح:")
        bot.register_next_step_handler(msg, lambda m: save_email(m, account_name))
        return
    msg = bot.send_message(message.chat.id, "🔒 أدخل كلمة مرور التطبيقات:")
    bot.register_next_step_handler(msg, lambda m: save_app_password(m, account_name, email))

def save_app_password(message, account_name, email):
    app_password = message.text
    user_id = message.chat.id
    try:
        with smtplib.SMTP("smtp.gmail.com", 587) as server:
            server.starttls()
            server.login(email, app_password)
        cursor.execute("INSERT OR REPLACE INTO users (user_id, account_name, email, app_password) VALUES (?, ?, ?, ?)",
                       (user_id, account_name, email, app_password))
        conn.commit()
        markup = InlineKeyboardMarkup()
        markup.add(InlineKeyboardButton("🔙 رجوع إلى القائمة الرئيسية", callback_data="main_menu"))
        bot.send_message(message.chat.id, "✅ تم حفظ الحساب بنجاح.", reply_markup=markup)
    except smtplib.SMTPAuthenticationError:
        bot.send_message(message.chat.id, "❌ كلمة مرور غير صحيحة. حاول مجددًا.")

def delete_account(message):
    account_identifier = message.text
    cursor.execute("SELECT * FROM users WHERE user_id=? AND (account_name=? OR email=?)",
                   (message.chat.id, account_identifier, account_identifier))
    account = cursor.fetchone()
    if account:
        cursor.execute("DELETE FROM users WHERE user_id=? AND (account_name=? OR email=?)",
                       (message.chat.id, account_identifier, account_identifier))
        conn.commit()
        bot.send_message(message.chat.id, "✅ تم حذف الحساب بنجاح.")
    else:
        bot.send_message(message.chat.id, "❌ لم يتم العثور على الحساب.")

def show_accounts(chat_id):
    cursor.execute("SELECT account_name, email FROM users WHERE user_id=?", (chat_id,))
    accounts = cursor.fetchall()
    if accounts:
        text = "👤 حساباتك:\n" + "\n".join([f"• {a[0]} ({a[1]})" for a in accounts])
        bot.send_message(chat_id, text)
    else:
        bot.send_message(chat_id, "❌ لا توجد حسابات.")

def process_step(chat_id, action):
    if action == "unblock_number":
        msg = bot.send_message(chat_id, "📞 أدخل الرقم (مع رمز الدولة):")
        bot.register_next_step_handler(msg, lambda m: select_account(chat_id, "send_unblock_number", m.text))
    elif action == "unblock_channel":
        msg = bot.send_message(chat_id, "📡 أدخل رابط القناة:")
        bot.register_next_step_handler(msg, lambda m: select_account(chat_id, "send_unblock_channel", m.text))
    elif action == "unblock_group":
        msg = bot.send_message(chat_id, "💬 أدخل رابط المجموعة:")
        bot.register_next_step_handler(msg, lambda m: select_account(chat_id, "send_unblock_group", m.text))
    elif action == "custom_message":
        msg = bot.send_message(chat_id, "✏️ أدخل العنوان:")
        bot.register_next_step_handler(msg, lambda m: select_account(chat_id, "send_custom_message", m.text))

def select_account(chat_id, next_step, extra_data=None):
    cursor.execute("SELECT account_name, email FROM users WHERE user_id=?", (chat_id,))
    accounts = cursor.fetchall()
    if accounts:
        markup = InlineKeyboardMarkup()
        for account in accounts:
            markup.add(InlineKeyboardButton(account[0], callback_data=f"{next_step}|{account[1]}|{extra_data}"))
        markup.add(InlineKeyboardButton("🔙 رجوع إلى القائمة الرئيسية", callback_data="main_menu"))
        bot.send_message(chat_id, "👤 اختر الحساب الذي تريد استخدامه:", reply_markup=markup)
    else:
        bot.send_message(chat_id, "❌ لا توجد حسابات. استخدم '📧 إضافة حساب'.")
        send_welcome(chat_id, "المستخدم")

def send_email_action(chat_id, email, action, extra_data):
    if action in ["send_unblock_number", "send_unblock_channel", "send_unblock_group", "send_custom_message"]:
        msg = bot.send_message(chat_id, "📤 أدخل عدد الرسائل:")
        bot.register_next_step_handler(msg, lambda m: ask_for_delay(chat_id, email, action, extra_data, m.text))

def ask_for_delay(chat_id, email, action, extra_data, msg_text):
    try:
        num_messages = int(msg_text)
        msg = bot.send_message(chat_id, "🕒 أدخل الوقت (بالثواني) بين كل رسالة:")
        bot.register_next_step_handler(msg, lambda m: send_multiple_emails(chat_id, email, action, extra_data, num_messages, m.text))
    except ValueError:
        msg = bot.send_message(chat_id, "❌ يجب إدخال رقم صحيح لعدد الرسائل:")
        bot.register_next_step_handler(msg, lambda m: ask_for_delay(chat_id, email, action, extra_data, m.text))

def send_multiple_emails(chat_id, email, action, extra_data, num_messages, delay_text):
    try:
        delay = int(delay_text)
        subject = ""
        if action == "send_unblock_number":
            subject = f"Lifting restrictions on the number: {extra_data}"
        elif action == "send_unblock_channel":
            subject = f"Lifting restrictions on my channel: {extra_data}"
        elif action == "send_unblock_group":
            subject = f"Lifting restrictions on the group: {extra_data}"
        elif action == "send_custom_message":
            subject = extra_data

        msg = bot.send_message(chat_id, "✏️ أدخل نص الرسالة:")
        bot.register_next_step_handler(msg, lambda m: send_email_loop(chat_id, email, subject, m.text, num_messages, delay))

    except ValueError:
        msg = bot.send_message(chat_id, "❌ يجب إدخال رقم صحيح لوقت الانتظار:")
        bot.register_next_step_handler(msg, lambda m: send_multiple_emails(chat_id, email, action, extra_data, num_messages, m.text))

def send_email_loop(chat_id, email, subject, body, num_messages, delay):
    cursor.execute("SELECT app_password FROM users WHERE user_id=? AND email=?", (chat_id, email))
    app_password = cursor.fetchone()
    if not app_password:
        bot.send_message(chat_id, "❌ لم يتم العثور على الحساب.")
        return
    app_password = app_password[0]

    for i in range(num_messages):
        try:
            with smtplib.SMTP("smtp.gmail.com", 587) as server:
                server.starttls()
                server.login(email, app_password)
                message = f"Subject: {subject}\n\n{body}"
                server.sendmail(email, "support@telegram.org", message.encode('utf-8'))  # استبدل البريد بالعنوان الصحيح
            bot.send_message(chat_id, f"📤 تم إرسال الرسالة رقم {i + 1} بنجاح ✅")
            time.sleep(delay)  # الانتظار قبل إرسال الرسالة التالية
        except Exception as e:
            bot.send_message(chat_id, f"❌ حدث خطأ أثناء إرسال الرسالة رقم {i + 1}: {e}")
            break

# بدء البوت
bot.polling(none_stop=True)
