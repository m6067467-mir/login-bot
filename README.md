import base64
import json
import os
import re
from datetime import datetime, timedelta, timezone
from io import BytesIO

from telegram import (
    Update,
    ReplyKeyboardMarkup,
    KeyboardButton,
    InlineKeyboardMarkup,
    InlineKeyboardButton,
)
from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    MessageHandler,
    ConversationHandler,
    CallbackQueryHandler,
    ContextTypes,
    filters,
)

# =========================
# SOZLAMALAR
# =========================
TOKEN = "8698224847:AAFuivL4ywKGMwt3ofq1gd2knsPnuDTdNkQ"
OWNER_ADMIN_ID = 7410827570
DATA_FILE = "bot_data.json"
LOGO_PATH = "/mnt/data/photo_2026-03-10_16-37-11.jpg"
SHOP_LIMIT_DAYS = 5
ASSIGNMENT_BASE_COIN = 10
VIDEO_WATCH_BASE_COIN = 1

COURSES = [
    "🎨 Grafik dizayn",
    "💻 Frontend",
    "🗄 Backend",
    "🔐 Kiber xavfsizlik",
    "🤖 Sun’iy intellekt",
]

DEFAULT_PRODUCTS = {
    "noutbuk": {
        "title": "💻 Noutbuk",
        "price": 7000000,
        "stock": 5,
        "image": "https://images.unsplash.com/photo-1496181133206-80ce9b88a853?q=80&w=1200&auto=format&fit=crop",
    },
    "klaviatura": {
        "title": "⌨️ Klaviatura",
        "price": 750,
        "stock": 2,
        "image": "https://images.unsplash.com/photo-1511467687858-23d96c32e4ae?q=80&w=1200&auto=format&fit=crop",
    },
    "lampa": {
        "title": "💡 Stol lampasi",
        "price": 500,
        "stock": 1,
        "image": "https://images.unsplash.com/photo-1507473885765-e6ed057f782c?q=80&w=1200&auto=format&fit=crop",
    },
    "sichqoncha": {
        "title": "🖱 Sichqoncha",
        "price": 3580,
        "stock": 2,
        "image": "https://images.unsplash.com/photo-1527864550417-7fd91fc51a46?q=80&w=1200&auto=format&fit=crop",
    },
    "premium": {
        "title": "⭐ Premium",
        "price": 3000,
        "stock": 999,
        "image": "https://images.unsplash.com/photo-1607082350920-7c0f1d8e6b3f?q=80&w=1200&auto=format&fit=crop",
    },
}

(
    REG_FULLNAME,
    REG_PHONE,
    REG_COURSE,
    UPDATE_NAME,
    ADD_COURSE,
    REMOVE_COURSE,
    HELP_MESSAGE,
    ADMIN_ADD_ADMIN,
    ADMIN_REMOVE_ADMIN,
    ADMIN_SET_LOCATION_TEXT,
    ADMIN_SET_LOCATION_MAP,
    ADMIN_SET_TASK_COURSE,
    ADMIN_SET_TASK_TEXT,
    ADMIN_ADD_LESSON_COURSE,
    ADMIN_ADD_LESSON_TYPE,
    ADMIN_ADD_LESSON_CONTENT,
    SUBMIT_TASK,
    ADMIN_BLOCK_USER,
    ADMIN_UNBLOCK_USER,
    ADMIN_DELETE_USER,
    ADMIN_COIN_USER,
    ADMIN_COIN_AMOUNT,
    ADMIN_PREMIUM_USER,
    ADMIN_REMOVE_PREMIUM_USER,
    LEARN_SELECT_COURSE,
) = range(25)


# =========================
# DATA
# =========================
def save_data(data):
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)


def load_data():
    default_data = {
        "admins": [OWNER_ADMIN_ID],
        "users": {},
        "location_text": "Lokatsiya hali kiritilmagan.",
        "location_map": "",
        "tasks": {},
        "lessons": {},
        "submissions": [],
        "help_requests": [],
        "purchases": [],
        "purchase_counter": 1,
        "lesson_counter": 1,
        "products": DEFAULT_PRODUCTS,
        "security_logs": [],
    }

    if not os.path.exists(DATA_FILE):
        save_data(default_data)
        return default_data

    with open(DATA_FILE, "r", encoding="utf-8") as f:
        data = json.load(f)

    for key, value in default_data.items():
        if key not in data:
            data[key] = value

    if "products" not in data or not isinstance(data["products"], dict):
        data["products"] = DEFAULT_PRODUCTS

    save_data(data)
    return data


def now_utc():
    return datetime.now(timezone.utc)


def parse_dt(value: str):
    if not value:
        return None
    try:
        dt = datetime.fromisoformat(value)
        if dt.tzinfo is None:
            dt = dt.replace(tzinfo=timezone.utc)
        return dt
    except Exception:
        return None


def is_admin(user_id: int) -> bool:
    data = load_data()
    return user_id in data["admins"]


def ensure_user(user):
    data = load_data()
    uid = str(user.id)

    if uid not in data["users"]:
        data["users"][uid] = {
            "user_id": user.id,
            "telegram_name": user.full_name,
            "username": user.username or "",
            "full_name": "",
            "phone": "",
            "courses": [],
            "coins": 0,
            "premium": False,
            "registered": False,
            "total_submissions": 0,
            "blocked_until": "",
            "deleted": False,
            "watched_lessons": [],
        }
        save_data(data)


def get_user_link(user):
    if user.username:
        return f"https://t.me/{user.username}"
    return "Username yo‘q"


def is_user_blocked(user_id: int):
    data = load_data()
    uid = str(user_id)
    if uid not in data["users"]:
        return False, None

    blocked_until = data["users"][uid].get("blocked_until", "")
    dt = parse_dt(blocked_until)
    if not dt:
        return False, None
    if dt > now_utc():
        return True, dt
    return False, None


def set_user_block(user_id: int, minutes: int):
    data = load_data()
    uid = str(user_id)
    if uid in data["users"]:
        until = now_utc() + timedelta(minutes=minutes)
        data["users"][uid]["blocked_until"] = until.isoformat()
        save_data(data)


def clear_user_block(user_id: int):
    data = load_data()
    uid = str(user_id)
    if uid in data["users"]:
        data["users"][uid]["blocked_until"] = ""
        save_data(data)


def get_user_main_keyboard(user_id: int):
    data = load_data()

    if is_admin(user_id):
        return ReplyKeyboardMarkup(
            [
                ["📊 Admin statistika", "👥 Foydalanuvchilar"],
                ["👮 Adminlar"],
                ["📝 Vazifa qo‘shish", "📚 Darslik qo‘shish"],
                ["💰 Coin boshqarish", "⭐ Premium boshqarish"],
                ["🚫 Foydalanuvchini bloklash", "✅ Blokdan chiqarish"],
                ["🗑 Foydalanuvchini o‘chirish", "📍 Lokatsiya sozlash"],
                ["🏆 To‘liq reyting", "🛍 Mahsulotlar"],
                ["📖 O‘qish", "🆘 Yordamlar"],
                ["ℹ️ Biz haqimizda"],
            ],
            resize_keyboard=True,
        )

    uid = str(user_id)
    registered = False
    if uid in data["users"]:
        registered = data["users"][uid].get("registered", False)

    if not registered:
        return ReplyKeyboardMarkup(
            [
                ["📝 Ro‘yxatdan o‘tish"],
                ["📞 Aloqa", "ℹ️ Biz haqimizda"],
            ],
            resize_keyboard=True,
        )

    return ReplyKeyboardMarkup(
        [
            ["📚 Kurslar", "👤 Profilim"],
            ["🏆 Reytingim", "🛍 Mahsulotlar"],
            ["📖 O‘qish", "📤 Vazifa topshirish"],
            ["🆘 Yordam", "📞 Aloqa"],
            ["ℹ️ Biz haqimizda"],
        ],
        resize_keyboard=True,
    )


courses_keyboard = ReplyKeyboardMarkup(
    [
        ["🎨 Grafik dizayn", "💻 Frontend"],
        ["🗄 Backend", "🔐 Kiber xavfsizlik"],
        ["🤖 Sun’iy intellekt"],
        ["⬅️ Orqaga"],
    ],
    resize_keyboard=True,
)

cancel_keyboard = ReplyKeyboardMarkup(
    [["❌ Bekor qilish"]],
    resize_keyboard=True,
)

profile_keyboard = ReplyKeyboardMarkup(
    [
        ["✏️ Ismni yangilash"],
        ["➕ Kurs qo‘shish", "➖ Kurs o‘chirish"],
        ["⬅️ Orqaga"],
    ],
    resize_keyboard=True,
)

admins_keyboard = ReplyKeyboardMarkup(
    [
        ["➕ Admin qo‘shish", "➖ Adminlikdan olish"],
        ["⬅️ Orqaga"],
    ],
    resize_keyboard=True,
)

location_keyboard = ReplyKeyboardMarkup(
    [
        ["📝 Lokatsiya matni", "🗺 Xarita linki"],
        ["⬅️ Orqaga"],
    ],
    resize_keyboard=True,
)

admin_lesson_type_keyboard = ReplyKeyboardMarkup(
    [
        ["video", "topshiriq"],
        ["darslik", "test"],
        ["❌ Bekor qilish"],
    ],
    resize_keyboard=True,
)

coin_keyboard = ReplyKeyboardMarkup(
    [
        ["➕ Coin qo‘shish", "➖ Coin ayirish"],
        ["⬅️ Orqaga"],
    ],
    resize_keyboard=True,
)

premium_keyboard = ReplyKeyboardMarkup(
    [
        ["⭐ Premium berish", "❌ Premiumni olish"],
        ["⬅️ Orqaga"],
    ],
    resize_keyboard=True,
)


# =========================
# YORDAMCHI FUNKSIYALAR
# =========================
def suspicious_message(text: str) -> bool:
    if not text:
        return False

    patterns = [
        r"token",
        r"admin id",
        r"sqlmap",
        r"drop table",
        r"<script",
        r"ddos",
        r"bypass",
        r"hack",
        r"exploit",
        r"inject",
    ]
    low = text.lower()
    return any(re.search(p, low) for p in patterns)


async def auto_security_check(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not update.effective_user or not update.message:
        return False

    user = update.effective_user
    if is_admin(user.id):
        return False

    ensure_user(user)

    blocked, until = is_user_blocked(user.id)
    if blocked:
        await update.message.reply_text(
            f"⛔ Siz vaqtincha bloklangansiz.\nBlok tugashi: {until.strftime('%Y-%m-%d %H:%M:%S UTC')}"
        )
        return True

    text = update.message.text or ""
    if suspicious_message(text):
        set_user_block(user.id, 2)
        data = load_data()
        data["security_logs"].append(
            {
                "user_id": user.id,
                "full_name": user.full_name,
                "text": text,
                "created_at": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
            }
        )
        save_data(data)

        for admin_id in data["admins"]:
            try:
                await context.bot.send_message(
                    chat_id=admin_id,
                    text=(
                        "🚨 Shubhali urinish aniqlandi\n\n"
                        f"👤 {user.full_name}\n"
                        f"🆔 {user.id}\n"
                        f"🔗 {get_user_link(user)}\n"
                        f"Xabar: {text}\n\n"
                        "Bot foydalanuvchini 2 daqiqaga blokladi."
                    ),
                )
            except Exception:
                pass

        await update.message.reply_text(
            "🚫 Shubhali faoliyat aniqlandi. Siz 2 daqiqaga bloklandingiz."
        )
        return True

    return False


def rating_sorted_users():
    data = load_data()
    users = []
    for uid, u in data["users"].items():
        if int(uid) not in data["admins"] and not u.get("deleted", False):
            users.append(u)
    users.sort(key=lambda x: x.get("coins", 0), reverse=True)
    return users


def get_user_rank(user_id: int):
    users = rating_sorted_users()
    for i, u in enumerate(users, start=1):
        if u["user_id"] == user_id:
            return i, len(users)
    return None, len(users)


def lesson_reward_for_user(user_record: dict) -> int:
    return VIDEO_WATCH_BASE_COIN * (2 if user_record.get("premium") else 1)


def assignment_reward_for_user(user_record: dict) -> int:
    return ASSIGNMENT_BASE_COIN * (2 if user_record.get("premium") else 1)


def find_lesson_by_id(data, lesson_id: int):
    for course, lessons in data["lessons"].items():
        for lesson in lessons:
            if lesson.get("id") == lesson_id:
                return course, lesson
    return None, None


def can_buy_product_now(data, user_id: int, item_key: str):
    limit_since = now_utc() - timedelta(days=SHOP_LIMIT_DAYS)
    for purchase in reversed(data["purchases"]):
        if purchase["user_id"] != user_id:
            continue
        if purchase["item_key"] != item_key:
            continue
        dt = parse_dt(purchase.get("created_at_iso", ""))
        if dt and dt >= limit_since and purchase.get("status") in ["pending", "approved"]:
            return False, dt + timedelta(days=SHOP_LIMIT_DAYS)
    return True, None


def decode_data_url_to_bytes(data_url: str):
    if not data_url.startswith("data:"):
        return None
    try:
        header, encoded = data_url.split(",", 1)
        mime_match = re.search(r"data:([^;]+)", header)
        mime = mime_match.group(1) if mime_match else "image/jpeg"
        ext = mime.split("/")[-1].replace("jpeg", "jpg")
        raw = base64.b64decode(encoded)
        bio = BytesIO(raw)
        bio.name = f"image.{ext}"
        bio.seek(0)
        return bio
    except Exception:
        return None


async def send_logo(chat_id: int, context: ContextTypes.DEFAULT_TYPE, caption: str, reply_markup=None):
    try:
        if os.path.exists(LOGO_PATH):
            with open(LOGO_PATH, "rb") as f:
                await context.bot.send_photo(chat_id=chat_id, photo=f, caption=caption, reply_markup=reply_markup)
        else:
            await context.bot.send_message(chat_id=chat_id, text=caption, reply_markup=reply_markup)
    except Exception:
        await context.bot.send_message(chat_id=chat_id, text=caption, reply_markup=reply_markup)


async def send_product_card(chat_id: int, context: ContextTypes.DEFAULT_TYPE, photo_value: str, caption: str, reply_markup):
    try:
        if isinstance(photo_value, str) and photo_value.startswith("data:"):
            bio = decode_data_url_to_bytes(photo_value)
            if bio:
                await context.bot.send_photo(chat_id=chat_id, photo=bio, caption=caption, reply_markup=reply_markup)
                return
        await context.bot.send_photo(chat_id=chat_id, photo=photo_value, caption=caption, reply_markup=reply_markup)
    except Exception:
        await context.bot.send_message(chat_id=chat_id, text=caption, reply_markup=reply_markup)


# =========================
# START
# =========================
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    ensure_user(update.effective_user)

    text = (
        "👋 Assalomu alaykum!\n\n"
        "IT Creative botiga xush kelibsiz.\n\n"
        "Bu bot orqali siz:\n"
        "📚 Kurslarni ko‘rasiz\n"
        "📖 O‘qish bo‘limiga kirasiz\n"
        "🎥 Video darslarni ko‘rasiz\n"
        "📝 Ro‘yxatdan o‘tasiz\n"
        "📤 .html topshiriq yuborasiz\n"
        "🛍 Mahsulot xarid qilasiz\n"
        "🏆 Reytingni ko‘rasiz"
    )

    await send_logo(
        chat_id=update.effective_chat.id,
        context=context,
        caption=text,
        reply_markup=get_user_main_keyboard(update.effective_user.id),
    )


# =========================
# MENU
# =========================
async def menu_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not update.message:
        return ConversationHandler.END

    if await auto_security_check(update, context):
        return ConversationHandler.END

    text = update.message.text
    user = update.effective_user
    ensure_user(user)
    data = load_data()

    if is_admin(user.id):
        if text == "📊 Admin statistika":
            real_users = [
                u for uid, u in data["users"].items()
                if int(uid) not in data["admins"] and not u.get("deleted", False)
            ]
            await update.message.reply_text(
                f"📊 Statistika\n\n"
                f"👥 Foydalanuvchilar: {len(real_users)}\n"
                f"📤 Topshiriqlar: {len(data['submissions'])}\n"
                f"📚 Darslar: {sum(len(v) for v in data['lessons'].values())}\n"
                f"🆘 Yordamlar: {len(data['help_requests'])}\n"
                f"🛍 Pending xaridlar: {len([p for p in data['purchases'] if p['status'] == 'pending'])}",
                reply_markup=get_user_main_keyboard(user.id),
            )
            return ConversationHandler.END

        elif text == "👥 Foydalanuvchilar":
            users = [
                u for uid, u in data["users"].items()
                if int(uid) not in data["admins"] and not u.get("deleted", False)
            ]
            if not users:
                await update.message.reply_text("Foydalanuvchi yo‘q.")
                return ConversationHandler.END

            msg = "👥 Foydalanuvchilar:\n\n"
            for u in users[:50]:
                tg_link = f"https://t.me/{u['username']}" if u.get("username") else "Username yo‘q"
                blocked = "Ha" if u.get("blocked_until") else "Yo‘q"
                msg += (
                    f"ID: {u['user_id']}\n"
                    f"Ism: {u.get('full_name', 'yo‘q')}\n"
                    f"Telegram: {tg_link}\n"
                    f"Kurslar: {', '.join(u.get('courses', [])) or 'yo‘q'}\n"
                    f"Coin: {u.get('coins', 0)}\n"
                    f"Premium: {'Ha' if u.get('premium') else 'Yo‘q'}\n"
                    f"Blok: {blocked}\n"
                    f"------------------\n"
                )
            await update.message.reply_text(msg)
            return ConversationHandler.END

        elif text == "👮 Adminlar":
            msg = "👮 Adminlar:\n\n"
            for admin_id in data["admins"]:
                msg += f"{admin_id}\n"
            await update.message.reply_text(msg, reply_markup=admins_keyboard)
            return ConversationHandler.END

        elif text == "➕ Admin qo‘shish":
            await update.message.reply_text("Admin qilinadigan user ID ni yuboring:", reply_markup=cancel_keyboard)
            return ADMIN_ADD_ADMIN

        elif text == "➖ Adminlikdan olish":
            await update.message.reply_text("Adminlikdan olinadigan user ID ni yuboring:", reply_markup=cancel_keyboard)
            return ADMIN_REMOVE_ADMIN

        elif text == "📍 Lokatsiya sozlash":
            await update.message.reply_text("Lokatsiya sozlamasini tanlang:", reply_markup=location_keyboard)
            return ConversationHandler.END

        elif text == "📝 Lokatsiya matni":
            await update.message.reply_text("Lokatsiya matnini yuboring:", reply_markup=cancel_keyboard)
            return ADMIN_SET_LOCATION_TEXT

        elif text == "🗺 Xarita linki":
            await update.message.reply_text("Google Maps yoki Yandex Maps linkini yuboring:", reply_markup=cancel_keyboard)
            return ADMIN_SET_LOCATION_MAP

        elif text == "📝 Vazifa qo‘shish":
            await update.message.reply_text(
                "Qaysi kurs uchun vazifa qo‘shiladi?\n" + "\n".join(COURSES),
                reply_markup=courses_keyboard,
            )
            return ADMIN_SET_TASK_COURSE

        elif text == "📚 Darslik qo‘shish":
            await update.message.reply_text(
                "Qaysi kurs uchun darslik qo‘shiladi?\n" + "\n".join(COURSES),
                reply_markup=courses_keyboard,
            )
            return ADMIN_ADD_LESSON_COURSE

        elif text == "💰 Coin boshqarish":
            await update.message.reply_text("Coin boshqarish turini tanlang:", reply_markup=coin_keyboard)
            return ConversationHandler.END

        elif text == "➕ Coin qo‘shish":
            context.user_data["coin_mode"] = "add"
            await update.message.reply_text("Foydalanuvchi ID sini yuboring:", reply_markup=cancel_keyboard)
            return ADMIN_COIN_USER

        elif text == "➖ Coin ayirish":
            context.user_data["coin_mode"] = "remove"
            await update.message.reply_text("Foydalanuvchi ID sini yuboring:", reply_markup=cancel_keyboard)
            return ADMIN_COIN_USER

        elif text == "⭐ Premium boshqarish":
            await update.message.reply_text("Premium boshqarish turini tanlang:", reply_markup=premium_keyboard)
            return ConversationHandler.END

        elif text == "⭐ Premium berish":
            await update.message.reply_text("Premium beriladigan user ID ni yuboring:", reply_markup=cancel_keyboard)
            return ADMIN_PREMIUM_USER

        elif text == "❌ Premiumni olish":
            await update.message.reply_text("Premiumi olinadigan user ID ni yuboring:", reply_markup=cancel_keyboard)
            return ADMIN_REMOVE_PREMIUM_USER

        elif text == "🚫 Foydalanuvchini bloklash":
            await update.message.reply_text("Blok qilinadigan user ID ni yuboring:", reply_markup=cancel_keyboard)
            return ADMIN_BLOCK_USER

        elif text == "✅ Blokdan chiqarish":
            await update.message.reply_text("Blokdan chiqariladigan user ID ni yuboring:", reply_markup=cancel_keyboard)
            return ADMIN_UNBLOCK_USER

        elif text == "🗑 Foydalanuvchini o‘chirish":
            await update.message.reply_text("O‘chiriladigan user ID ni yuboring:", reply_markup=cancel_keyboard)
            return ADMIN_DELETE_USER

        elif text == "🏆 To‘liq reyting":
            users = rating_sorted_users()
            if not users:
                await update.message.reply_text("Reyting bo‘sh.")
                return ConversationHandler.END

            msg = "🏆 To‘liq reyting:\n\n"
            for i, u in enumerate(users, start=1):
                msg += f"{i}. {u.get('full_name', 'No name')} — {u.get('coins', 0)} coin\n"
            await update.message.reply_text(msg)
            return ConversationHandler.END

        elif text == "🆘 Yordamlar":
            if not data["help_requests"]:
                await update.message.reply_text("Yordam xabarlari yo‘q.")
                return ConversationHandler.END

            msg = "🆘 Yordam xabarlari:\n\n"
            for h in data["help_requests"][-10:]:
                msg += (
                    f"ID: {h['user_id']}\n"
                    f"Ism: {h['full_name']}\n"
                    f"Xabar: {h['text']}\n"
                    f"Vaqt: {h['created_at']}\n"
                    f"---\n"
                )
            await update.message.reply_text(msg)
            return ConversationHandler.END

    if text == "📚 Kurslar":
        msg = "📚 Mavjud kurslar:\n\n" + "\n".join(COURSES)
        uid = str(user.id)
        if uid in data["users"] and data["users"][uid].get("courses"):
            msg += "\n\n📌 Sizning kurslaringiz:\n" + "\n".join(data["users"][uid]["courses"])
        await update.message.reply_text(msg, reply_markup=courses_keyboard)
        return ConversationHandler.END

    elif text == "📖 O‘qish":
        uid = str(user.id)
        if uid not in data["users"] or not data["users"][uid].get("registered"):
            await update.message.reply_text("Avval ro‘yxatdan o‘ting.")
            return ConversationHandler.END
        my_courses = data["users"][uid].get("courses", [])
        if not my_courses:
            await update.message.reply_text("Sizda kurs yo‘q.")
            return ConversationHandler.END
        await update.message.reply_text(
            "Qaysi kursni ochmoqchisiz?",
            reply_markup=ReplyKeyboardMarkup([[c] for c in my_courses] + [["❌ Bekor qilish"]], resize_keyboard=True),
        )
        return LEARN_SELECT_COURSE

    elif text == "📝 Ro‘yxatdan o‘tish":
        uid = str(user.id)
        if uid in data["users"] and data["users"][uid].get("registered"):
            await update.message.reply_text("Siz allaqachon ro‘yxatdan o‘tgansiz.", reply_markup=get_user_main_keyboard(user.id))
            return ConversationHandler.END

        await update.message.reply_text("Ism va familiyangizni yuboring:", reply_markup=cancel_keyboard)
        return REG_FULLNAME

    elif text == "👤 Profilim":
        uid = str(user.id)
        u = data["users"].get(uid)
        if not u:
            await update.message.reply_text("Avval ro‘yxatdan o‘ting.")
            return ConversationHandler.END

        rank, total = get_user_rank(user.id)
        await update.message.reply_text(
            f"👤 Profil\n\n"
            f"Ism: {u.get('full_name', 'yo‘q')}\n"
            f"Telefon: {u.get('phone', 'yo‘q')}\n"
            f"Telegram: {get_user_link(user)}\n"
            f"Kurslar: {', '.join(u.get('courses', [])) or 'yo‘q'}\n"
            f"Coin: {u.get('coins', 0)}\n"
            f"Premium: {'Ha' if u.get('premium') else 'Yo‘q'}\n"
            f"Reyting: {rank if rank else '-'} / {total}",
            reply_markup=profile_keyboard,
        )
        return ConversationHandler.END

    elif text == "✏️ Ismni yangilash":
        await update.message.reply_text("Yangi ism-familiyangizni yuboring:", reply_markup=cancel_keyboard)
        return UPDATE_NAME

    elif text == "➕ Kurs qo‘shish":
        await update.message.reply_text("Qo‘shmoqchi bo‘lgan kursni tanlang:", reply_markup=courses_keyboard)
        return ADD_COURSE

    elif text == "➖ Kurs o‘chirish":
        await update.message.reply_text("O‘chirmoqchi bo‘lgan kursni tanlang:", reply_markup=courses_keyboard)
        return REMOVE_COURSE

    elif text == "🏆 Reytingim":
        rank, total = get_user_rank(user.id)
        uid = str(user.id)
        my_coin = data["users"].get(uid, {}).get("coins", 0)
        await update.message.reply_text(
            f"🏆 Sizning o‘rningiz: {rank if rank else '-'} / {total}\n💰 Coin: {my_coin}",
            reply_markup=get_user_main_keyboard(user.id),
        )
        return ConversationHandler.END

    elif text == "🛍 Mahsulotlar":
        products = data["products"]
        for key, item in products.items():
            price = item["price"]
            uid = str(user.id)
            discount_text = ""
            if uid in data["users"] and data["users"][uid].get("premium") and key != "premium":
                discounted = int(price * 0.95)
                discount_text = f"\n⭐ Premium chegirma bilan: {discounted} coin"

            caption = (
                f"{item['title']}\n"
                f"Narxi: {price} coin\n"
                f"Qoldiq: {item['stock']} ta\n"
                f"⏱ Bir xil mahsulotni {SHOP_LIMIT_DAYS} kunda 1 marta olish mumkin"
                f"{discount_text}"
            )
            keyboard = InlineKeyboardMarkup(
                [[InlineKeyboardButton("🛒 Sotib olish", callback_data=f"buy:{key}")]]
            )
            await send_product_card(
                chat_id=update.effective_chat.id,
                context=context,
                photo_value=item["image"],
                caption=caption,
                reply_markup=keyboard,
            )
        return ConversationHandler.END

    elif text == "📤 Vazifa topshirish":
        uid = str(user.id)
        if uid not in data["users"] or not data["users"][uid].get("registered"):
            await update.message.reply_text("Avval ro‘yxatdan o‘ting.")
            return ConversationHandler.END

        courses = data["users"][uid].get("courses", [])
        if not courses:
            await update.message.reply_text("Sizda kurs yo‘q.")
            return ConversationHandler.END

        msg = "Sizning kurslaringiz bo‘yicha vazifalar:\n\n"
        for c in courses:
            task = data["tasks"].get(c, "Vazifa yo‘q")
            msg += f"{c}\n📝 {task}\n\n"

        msg += "Faqat .html fayl yuboring. Admin tasdiqlasa oddiy foydalanuvchiga 10 coin, premium foydalanuvchiga 20 coin beriladi."
        await update.message.reply_text(msg, reply_markup=cancel_keyboard)
        return SUBMIT_TASK

    elif text == "🆘 Yordam":
        await update.message.reply_text("Muammo yoki xatoni yozing.\nRasm ham yuborishingiz mumkin.", reply_markup=cancel_keyboard)
        return HELP_MESSAGE

    elif text == "📞 Aloqa":
        keyboard = InlineKeyboardMarkup(
            [[InlineKeyboardButton("📩 Telegramga yozish", url="https://t.me/husanbek_coder")]]
        )
        await update.message.reply_text("📞 Aloqa\nTelefon: +998 97 521 66 86", reply_markup=keyboard)
        return ConversationHandler.END

    elif text == "ℹ️ Biz haqimizda":
        loc_text = data.get("location_text", "Kiritilmagan")
        loc_map = data.get("location_map", "")
        keyboard = None
        if loc_map:
            keyboard = InlineKeyboardMarkup(
                [[InlineKeyboardButton("🗺 Xaritada ochish", url=loc_map)]]
            )
        await send_logo(
            chat_id=update.effective_chat.id,
            context=context,
            caption=f"IT Creative — zamonaviy kasblarni o‘rgatuvchi markaz.\n\n📍 {loc_text}",
            reply_markup=keyboard or get_user_main_keyboard(user.id),
        )
        return ConversationHandler.END

    elif text == "⬅️ Orqaga":
        await update.message.reply_text("Asosiy menyu", reply_markup=get_user_main_keyboard(user.id))
        return ConversationHandler.END

    elif text in COURSES:
        course_info = {
            "🎨 Grafik dizayn": "Photoshop, Illustrator, banner, logo, poster.",
            "💻 Frontend": "HTML, CSS, JavaScript, sayt yasash.",
            "🗄 Backend": "Python, API, database, server.",
            "🔐 Kiber xavfsizlik": "Tarmoq xavfsizligi va himoya.",
            "🤖 Sun’iy intellekt": "AI, ML, neyron tarmoqlar.",
        }
        lessons = data["lessons"].get(text, [])
        lesson_text = ""
        if lessons:
            lesson_text = "\n\n📚 So‘nggi darslar:\n"
            for l in lessons[-3:]:
                lesson_text += f"- [{l['type']}] {l['content_label']}\n"

        await update.message.reply_text(
            f"{text}\n\n{course_info.get(text, '')}{lesson_text}",
            reply_markup=courses_keyboard,
        )
        return ConversationHandler.END

    else:
        await update.message.reply_text("Iltimos, tugmalardan foydalaning.", reply_markup=get_user_main_keyboard(user.id))
        return ConversationHandler.END


# =========================
# USER FLOWS
# =========================
async def reg_fullname(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    if text == "❌ Bekor qilish":
        await update.message.reply_text("Bekor qilindi.", reply_markup=get_user_main_keyboard(update.effective_user.id))
        return ConversationHandler.END

    context.user_data["reg_full_name"] = text

    phone_keyboard = ReplyKeyboardMarkup(
        [
            [KeyboardButton("📱 Telefon raqamni yuborish", request_contact=True)],
            ["❌ Bekor qilish"],
        ],
        resize_keyboard=True,
        one_time_keyboard=True,
    )

    await update.message.reply_text("Telefon raqamingizni yuboring:", reply_markup=phone_keyboard)
    return REG_PHONE


async def reg_phone(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.message.text == "❌ Bekor qilish":
        await update.message.reply_text("Bekor qilindi.", reply_markup=get_user_main_keyboard(update.effective_user.id))
        return ConversationHandler.END

    phone = update.message.contact.phone_number if update.message.contact else update.message.text
    context.user_data["reg_phone"] = phone

    await update.message.reply_text("Birinchi kursni tanlang:", reply_markup=courses_keyboard)
    return REG_COURSE


async def reg_course(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    if text == "❌ Bekor qilish":
        await update.message.reply_text("Bekor qilindi.", reply_markup=get_user_main_keyboard(update.effective_user.id))
        return ConversationHandler.END

    if text not in COURSES:
        await update.message.reply_text("Kurslardan birini tanlang.")
        return REG_COURSE

    data = load_data()
    uid = str(update.effective_user.id)

    data["users"][uid]["full_name"] = context.user_data.get("reg_full_name", "")
    data["users"][uid]["phone"] = context.user_data.get("reg_phone", "")
    data["users"][uid]["courses"] = [text]
    data["users"][uid]["registered"] = True
    data["users"][uid]["username"] = update.effective_user.username or ""
    save_data(data)

    tg_link = get_user_link(update.effective_user)
    for admin_id in data["admins"]:
        try:
            await context.bot.send_message(
                chat_id=admin_id,
                text=(
                    "📥 Yangi foydalanuvchi ro‘yxatdan o‘tdi\n\n"
                    f"👤 Ism: {data['users'][uid]['full_name']}\n"
                    f"📞 Telefon: {data['users'][uid]['phone']}\n"
                    f"🔗 Telegram: {tg_link}\n"
                    f"🆔 ID: {uid}\n"
                    f"📚 Kurs: {text}"
                ),
            )
        except Exception:
            pass

    await update.message.reply_text("✅ Muvaffaqiyatli ro‘yxatdan o‘tdingiz.", reply_markup=get_user_main_keyboard(update.effective_user.id))
    return ConversationHandler.END


async def update_name_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    if text == "❌ Bekor qilish":
        await update.message.reply_text("Bekor qilindi.", reply_markup=profile_keyboard)
        return ConversationHandler.END

    data = load_data()
    uid = str(update.effective_user.id)
    data["users"][uid]["full_name"] = text
    save_data(data)

    await update.message.reply_text("✅ Ismingiz yangilandi.", reply_markup=profile_keyboard)
    return ConversationHandler.END


async def add_course_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    if text == "❌ Bekor qilish":
        await update.message.reply_text("Bekor qilindi.", reply_markup=profile_keyboard)
        return ConversationHandler.END

    if text not in COURSES:
        await update.message.reply_text("Kurslardan birini tanlang.")
        return ADD_COURSE

    data = load_data()
    uid = str(update.effective_user.id)
    user_courses = data["users"][uid].get("courses", [])

    if text in user_courses:
        await update.message.reply_text("Bu kurs sizda allaqachon bor.", reply_markup=profile_keyboard)
        return ConversationHandler.END

    if len(user_courses) >= 3:
        await update.message.reply_text("Siz eng ko‘pi bilan 3 ta kursga qo‘shila olasiz.", reply_markup=profile_keyboard)
        return ConversationHandler.END

    user_courses.append(text)
    data["users"][uid]["courses"] = user_courses
    save_data(data)

    await update.message.reply_text("✅ Kurs qo‘shildi.", reply_markup=profile_keyboard)
    return ConversationHandler.END


async def remove_course_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    if text == "❌ Bekor qilish":
        await update.message.reply_text("Bekor qilindi.", reply_markup=profile_keyboard)
        return ConversationHandler.END

    data = load_data()
    uid = str(update.effective_user.id)
    user_courses = data["users"][uid].get("courses", [])

    if text not in user_courses:
        await update.message.reply_text("Bu kurs sizda yo‘q.", reply_markup=profile_keyboard)
        return ConversationHandler.END

    user_courses.remove(text)
    data["users"][uid]["courses"] = user_courses
    save_data(data)

    await update.message.reply_text("✅ Kurs o‘chirildi.", reply_markup=profile_keyboard)
    return ConversationHandler.END


async def help_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    data = load_data()
    user = update.effective_user

    if update.message.text == "❌ Bekor qilish":
        await update.message.reply_text("Bekor qilindi.", reply_markup=get_user_main_keyboard(user.id))
        return ConversationHandler.END

    record = {
        "user_id": user.id,
        "full_name": user.full_name,
        "created_at": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
        "text": update.message.text if update.message.text else "",
        "has_photo": bool(update.message.photo),
    }
    data["help_requests"].append(record)
    save_data(data)

    for admin_id in data["admins"]:
        try:
            if update.message.photo:
                photo = update.message.photo[-1].file_id
                caption = (
                    f"🆘 Yordam xabari\n"
                    f"👤 {user.full_name}\n"
                    f"🆔 {user.id}\n"
                    f"🔗 {get_user_link(user)}"
                )
                await context.bot.send_photo(chat_id=admin_id, photo=photo, caption=caption)
            else:
                await context.bot.send_message(
                    chat_id=admin_id,
                    text=(
                        f"🆘 Yordam xabari\n"
                        f"👤 {user.full_name}\n"
                        f"🆔 {user.id}\n"
                        f"🔗 {get_user_link(user)}\n"
                        f"Xabar: {record['text']}"
                    ),
                )
        except Exception:
            pass

    await update.message.reply_text("✅ Xabaringiz adminga yuborildi.", reply_markup=get_user_main_keyboard(user.id))
    return ConversationHandler.END


async def learn_select_course_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    user = update.effective_user
    if text == "❌ Bekor qilish":
        await update.message.reply_text("Bekor qilindi.", reply_markup=get_user_main_keyboard(user.id))
        return ConversationHandler.END

    data = load_data()
    uid = str(user.id)
    my_courses = data["users"].get(uid, {}).get("courses", [])
    if text not in my_courses:
        await update.message.reply_text("Faqat o‘zingizdagi kursni tanlang.")
        return LEARN_SELECT_COURSE

    lessons = data["lessons"].get(text, [])
    if not lessons:
        await update.message.reply_text("Bu kursda hali dars yo‘q.", reply_markup=get_user_main_keyboard(user.id))
        return ConversationHandler.END

    await update.message.reply_text(f"📖 {text} darslari:")
    for lesson in lessons:
        buttons = []
        if lesson["type"] == "video":
            buttons.append([InlineKeyboardButton("🎥 Videoni ochish", callback_data=f"lesson:{lesson['id']}")])
            buttons.append([InlineKeyboardButton("✅ Ko‘rdim", callback_data=f"watch:{lesson['id']}")])
        elif lesson["type"] == "topshiriq":
            buttons.append([InlineKeyboardButton("📝 Topshiriqni ochish", callback_data=f"lesson:{lesson['id']}")])
        else:
            buttons.append([InlineKeyboardButton("📚 Ochish", callback_data=f"lesson:{lesson['id']}")])

        kb = InlineKeyboardMarkup(buttons)
        await update.message.reply_text(
            f"ID: {lesson['id']}\nTuri: {lesson['type']}\nNomi: {lesson['content_label']}",
            reply_markup=kb,
        )

    await update.message.reply_text("Asosiy menyuga qaytish uchun ⬅️ Orqaga ni bosing.", reply_markup=get_user_main_keyboard(user.id))
    return ConversationHandler.END


# =========================
# ADMIN FLOWS
# =========================
async def admin_add_admin_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    if text == "❌ Bekor qilish":
        await update.message.reply_text("Bekor qilindi.", reply_markup=admins_keyboard)
        return ConversationHandler.END

    try:
        new_admin_id = int(text)
    except ValueError:
        await update.message.reply_text("Faqat user ID yuboring.")
        return ADMIN_ADD_ADMIN

    data = load_data()
    if new_admin_id not in data["admins"]:
        data["admins"].append(new_admin_id)
        save_data(data)

    await update.message.reply_text("✅ Yangi admin qo‘shildi.", reply_markup=admins_keyboard)
    return ConversationHandler.END


async def admin_remove_admin_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    if text == "❌ Bekor qilish":
        await update.message.reply_text("Bekor qilindi.", reply_markup=admins_keyboard)
        return ConversationHandler.END

    try:
        admin_id = int(text)
    except ValueError:
        await update.message.reply_text("Faqat user ID yuboring.")
        return ADMIN_REMOVE_ADMIN

    if admin_id == OWNER_ADMIN_ID:
        await update.message.reply_text("Asosiy adminni olib bo‘lmaydi.", reply_markup=admins_keyboard)
        return ConversationHandler.END

    data = load_data()
    if admin_id in data["admins"]:
        data["admins"].remove(admin_id)
        save_data(data)
        await update.message.reply_text("✅ Adminlik olib tashlandi.", reply_markup=admins_keyboard)
    else:
        await update.message.reply_text("Bunday admin yo‘q.", reply_markup=admins_keyboard)

    return ConversationHandler.END


async def admin_set_location_text_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    if text == "❌ Bekor qilish":
        await update.message.reply_text("Bekor qilindi.", reply_markup=location_keyboard)
        return ConversationHandler.END

    data = load_data()
    data["location_text"] = text
    save_data(data)

    await update.message.reply_text("✅ Lokatsiya matni saqlandi.", reply_markup=location_keyboard)
    return ConversationHandler.END


async def admin_set_location_map_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    if text == "❌ Bekor qilish":
        await update.message.reply_text("Bekor qilindi.", reply_markup=location_keyboard)
        return ConversationHandler.END

    data = load_data()
    data["location_map"] = text
    save_data(data)

    await update.message.reply_text("✅ Xarita linki saqlandi.", reply_markup=location_keyboard)
    return ConversationHandler.END


async def admin_task_course_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    if text == "❌ Bekor qilish":
        await update.message.reply_text("Bekor qilindi.", reply_markup=get_user_main_keyboard(update.effective_user.id))
        return ConversationHandler.END

    if text not in COURSES:
        await update.message.reply_text("Kursni tanlang.")
        return ADMIN_SET_TASK_COURSE

    context.user_data["admin_task_course"] = text
    await update.message.reply_text(f"{text} uchun vazifa matnini yuboring:", reply_markup=cancel_keyboard)
    return ADMIN_SET_TASK_TEXT


async def admin_task_text_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    if text == "❌ Bekor qilish":
        await update.message.reply_text("Bekor qilindi.", reply_markup=get_user_main_keyboard(update.effective_user.id))
        return ConversationHandler.END

    course = context.user_data.get("admin_task_course")
    data = load_data()
    data["tasks"][course] = text
    save_data(data)

    for uid, u in data["users"].items():
        if course in u.get("courses", []):
            try:
                await context.bot.send_message(chat_id=int(uid), text=f"📝 {course} uchun yangi vazifa:\n\n{text}")
            except Exception:
                pass

    await update.message.reply_text("✅ Vazifa qo‘shildi.", reply_markup=get_user_main_keyboard(update.effective_user.id))
    return ConversationHandler.END


async def admin_lesson_course_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    if text == "❌ Bekor qilish":
        await update.message.reply_text("Bekor qilindi.", reply_markup=get_user_main_keyboard(update.effective_user.id))
        return ConversationHandler.END

    if text not in COURSES:
        await update.message.reply_text("Kursni tanlang.")
        return ADMIN_ADD_LESSON_COURSE

    context.user_data["lesson_course"] = text
    await update.message.reply_text("Turini tanlang:", reply_markup=admin_lesson_type_keyboard)
    return ADMIN_ADD_LESSON_TYPE


async def admin_lesson_type_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    if text == "❌ Bekor qilish":
        await update.message.reply_text("Bekor qilindi.", reply_markup=get_user_main_keyboard(update.effective_user.id))
        return ConversationHandler.END

    if text not in ["video", "test", "topshiriq", "darslik"]:
        await update.message.reply_text("To‘g‘ri tur tanlang.")
        return ADMIN_ADD_LESSON_TYPE

    context.user_data["lesson_type"] = text

    if text == "video":
        await update.message.reply_text(
            "Video uchun .mp4 fayl yuboring. Caption yoki keyingi xabarda video nomini yuborishingiz mumkin.",
            reply_markup=cancel_keyboard,
        )
    else:
        await update.message.reply_text(
            "Kontentni matn ko‘rinishida yuboring:",
            reply_markup=cancel_keyboard,
        )
    return ADMIN_ADD_LESSON_CONTENT


async def admin_lesson_content_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.message.text == "❌ Bekor qilish":
        await update.message.reply_text("Bekor qilindi.", reply_markup=get_user_main_keyboard(update.effective_user.id))
        return ConversationHandler.END

    course = context.user_data.get("lesson_course")
    lesson_type = context.user_data.get("lesson_type")
    data = load_data()

    content_value = ""
    content_label = ""

    if lesson_type == "video":
        if not update.message.document:
            await update.message.reply_text("Faqat .mp4 video fayl yuboring.")
            return ADMIN_ADD_LESSON_CONTENT

        filename = (update.message.document.file_name or "").lower()
        if not filename.endswith(".mp4"):
            await update.message.reply_text("Faqat .mp4 video qabul qilinadi.")
            return ADMIN_ADD_LESSON_CONTENT

        content_value = f"file_id:{update.message.document.file_id}"
        content_label = update.message.document.file_name or "Video dars"
    else:
        if not update.message.text:
            await update.message.reply_text("Matn yuboring.")
            return ADMIN_ADD_LESSON_CONTENT
        content_value = update.message.text
        content_label = update.message.text[:60]

    if course not in data["lessons"]:
        data["lessons"][course] = []

    lesson_id = data.get("lesson_counter", 1)
    data["lesson_counter"] = lesson_id + 1

    data["lessons"][course].append(
        {
            "id": lesson_id,
            "type": lesson_type,
            "content": content_value,
            "content_label": content_label,
            "created_at": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
            "watched_by": [],
        }
    )
    save_data(data)

    await update.message.reply_text("✅ Darslik qo‘shildi.", reply_markup=get_user_main_keyboard(update.effective_user.id))
    return ConversationHandler.END


async def admin_block_user_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    if text == "❌ Bekor qilish":
        await update.message.reply_text("Bekor qilindi.", reply_markup=get_user_main_keyboard(update.effective_user.id))
        return ConversationHandler.END

    try:
        user_id = int(text)
    except ValueError:
        await update.message.reply_text("Faqat user ID yuboring.")
        return ADMIN_BLOCK_USER

    set_user_block(user_id, 120)
    await update.message.reply_text("✅ Foydalanuvchi 120 daqiqaga bloklandi.", reply_markup=get_user_main_keyboard(update.effective_user.id))
    try:
        await context.bot.send_message(chat_id=user_id, text="⛔ Siz admin tomonidan vaqtincha bloklandingiz.")
    except Exception:
        pass
    return ConversationHandler.END


async def admin_unblock_user_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    if text == "❌ Bekor qilish":
        await update.message.reply_text("Bekor qilindi.", reply_markup=get_user_main_keyboard(update.effective_user.id))
        return ConversationHandler.END

    try:
        user_id = int(text)
    except ValueError:
        await update.message.reply_text("Faqat user ID yuboring.")
        return ADMIN_UNBLOCK_USER

    clear_user_block(user_id)
    await update.message.reply_text("✅ Foydalanuvchi blokdan chiqarildi.", reply_markup=get_user_main_keyboard(update.effective_user.id))
    try:
        await context.bot.send_message(chat_id=user_id, text="✅ Siz blokdan chiqarildingiz.")
    except Exception:
        pass
    return ConversationHandler.END


async def admin_delete_user_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    if text == "❌ Bekor qilish":
        await update.message.reply_text("Bekor qilindi.", reply_markup=get_user_main_keyboard(update.effective_user.id))
        return ConversationHandler.END

    data = load_data()
    if text not in data["users"]:
        await update.message.reply_text("Bunday foydalanuvchi topilmadi.")
        return ADMIN_DELETE_USER

    data["users"][text]["deleted"] = True
    save_data(data)

    await update.message.reply_text("✅ Foydalanuvchi o‘chirildi.", reply_markup=get_user_main_keyboard(update.effective_user.id))
    return ConversationHandler.END


async def admin_coin_user_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    if text == "❌ Bekor qilish":
        await update.message.reply_text("Bekor qilindi.", reply_markup=coin_keyboard)
        return ConversationHandler.END

    context.user_data["coin_target_user"] = text
    await update.message.reply_text("Coin miqdorini yuboring. Masalan: 100", reply_markup=cancel_keyboard)
    return ADMIN_COIN_AMOUNT


async def admin_coin_amount_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    if text == "❌ Bekor qilish":
        await update.message.reply_text("Bekor qilindi.", reply_markup=coin_keyboard)
        return ConversationHandler.END

    try:
        amount = int(text)
    except ValueError:
        await update.message.reply_text("Faqat son yuboring.")
        return ADMIN_COIN_AMOUNT

    data = load_data()
    target_uid = context.user_data.get("coin_target_user")
    if target_uid not in data["users"]:
        await update.message.reply_text("Foydalanuvchi topilmadi.", reply_markup=coin_keyboard)
        return ConversationHandler.END

    mode = context.user_data.get("coin_mode", "add")
    if mode == "remove":
        data["users"][target_uid]["coins"] -= amount
        if data["users"][target_uid]["coins"] < 0:
            data["users"][target_uid]["coins"] = 0
    else:
        data["users"][target_uid]["coins"] += amount

    save_data(data)

    await update.message.reply_text("✅ Coin yangilandi.", reply_markup=coin_keyboard)
    try:
        await context.bot.send_message(
            chat_id=int(target_uid),
            text=f"💰 Coiningiz yangilandi. Hozirgi coin: {data['users'][target_uid]['coins']}",
        )
    except Exception:
        pass
    return ConversationHandler.END


async def admin_premium_user_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    if text == "❌ Bekor qilish":
        await update.message.reply_text("Bekor qilindi.", reply_markup=premium_keyboard)
        return ConversationHandler.END

    data = load_data()
    if text not in data["users"]:
        await update.message.reply_text("Foydalanuvchi topilmadi.")
        return ADMIN_PREMIUM_USER

    data["users"][text]["premium"] = True
    save_data(data)

    await update.message.reply_text("✅ Premium berildi.", reply_markup=premium_keyboard)
    return ConversationHandler.END


async def admin_remove_premium_user_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    if text == "❌ Bekor qilish":
        await update.message.reply_text("Bekor qilindi.", reply_markup=premium_keyboard)
        return ConversationHandler.END

    data = load_data()
    if text not in data["users"]:
        await update.message.reply_text("Foydalanuvchi topilmadi.")
        return ADMIN_REMOVE_PREMIUM_USER

    data["users"][text]["premium"] = False
    save_data(data)

    await update.message.reply_text("✅ Premium olib tashlandi.", reply_markup=premium_keyboard)
    return ConversationHandler.END


# =========================
# SUBMISSION
# =========================
def allowed_document(filename: str):
    if not filename:
        return False
    return filename.lower().endswith(".html")


async def submit_task_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    data = load_data()
    uid = str(user.id)

    if update.message.text == "❌ Bekor qilish":
        await update.message.reply_text("Bekor qilindi.", reply_markup=get_user_main_keyboard(user.id))
        return ConversationHandler.END

    if not update.message.document:
        await update.message.reply_text("Faqat .html fayl yuboring.")
        return SUBMIT_TASK

    filename = update.message.document.file_name
    if not allowed_document(filename):
        await update.message.reply_text("Faqat .html fayl yuboriladi.")
        return SUBMIT_TASK

    file_info = {
        "type": "document",
        "file_id": update.message.document.file_id,
        "file_name": filename,
    }

    record = {
        "user_id": user.id,
        "full_name": data["users"][uid].get("full_name", user.full_name),
        "courses": data["users"][uid].get("courses", []),
        "text": "",
        "file": file_info,
        "created_at": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
        "status": "pending",
    }

    data["submissions"].append(record)
    data["users"][uid]["total_submissions"] += 1
    save_data(data)

    reward_preview = assignment_reward_for_user(data["users"][uid])
    for admin_id in data["admins"]:
        try:
            admin_text = (
                f"📤 Yangi .html topshiriq topshirildi\n\n"
                f"👤 {record['full_name']}\n"
                f"🆔 {user.id}\n"
                f"🔗 {get_user_link(user)}\n"
                f"📚 Kurslar: {', '.join(record['courses'])}\n"
                f"🪙 Tasdiqlansa beriladigan coin: {reward_preview}\n\n"
                f"Tasdiqlash: /grade {user.id}"
            )
            await context.bot.send_document(chat_id=admin_id, document=file_info["file_id"], caption=admin_text)
        except Exception:
            pass

    await update.message.reply_text(
        "✅ .html topshiriq yuborildi. Admin tasdiqlagach coin beriladi.",
        reply_markup=get_user_main_keyboard(user.id),
    )
    return ConversationHandler.END


# =========================
# SHOP
# =========================
async def buy_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    user = query.from_user
    ensure_user(user)

    data = load_data()
    uid = str(user.id)

    if is_admin(user.id):
        await query.message.reply_text("Admin xarid qilmaydi.")
        return

    item_key = query.data.split(":")[1]
    item = data["products"].get(item_key)
    if not item:
        await query.message.reply_text("Mahsulot topilmadi.")
        return

    if item["stock"] <= 0:
        await query.message.reply_text("❌ Mahsulot tugagan.")
        return

    allowed, next_time = can_buy_product_now(data, user.id, item_key)
    if not allowed:
        await query.message.reply_text(
            f"❌ Bu mahsulotni {SHOP_LIMIT_DAYS} kunda faqat 1 marta olishingiz mumkin.\n"
            f"Keyinroq urinib ko‘ring: {next_time.strftime('%Y-%m-%d %H:%M UTC')}"
        )
        return

    user_data = data["users"].get(uid)
    if not user_data or not user_data.get("registered"):
        await query.message.reply_text("Avval ro‘yxatdan o‘ting.")
        return

    final_price = item["price"]
    if user_data.get("premium") and item_key != "premium":
        final_price = int(final_price * 0.95)

    if user_data["coins"] < final_price:
        await query.message.reply_text(
            f"❌ Coin yetarli emas.\nKerak: {final_price}\nSizda: {user_data['coins']}"
        )
        return

    purchase_id = data["purchase_counter"]
    data["purchase_counter"] += 1

    purchase = {
        "purchase_id": purchase_id,
        "user_id": user.id,
        "full_name": user_data.get("full_name", user.full_name),
        "item_key": item_key,
        "item_title": item["title"],
        "price": final_price,
        "status": "pending",
        "created_at": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
        "created_at_iso": now_utc().isoformat(),
    }

    data["purchases"].append(purchase)
    save_data(data)

    for admin_id in data["admins"]:
        try:
            await context.bot.send_message(
                chat_id=admin_id,
                text=(
                    f"🛍 Yangi xarid so‘rovi\n\n"
                    f"ID: {purchase_id}\n"
                    f"👤 {purchase['full_name']}\n"
                    f"🆔 {user.id}\n"
                    f"📦 {purchase['item_title']}\n"
                    f"💰 Narx: {final_price}\n\n"
                    f"Tasdiqlash: /approvebuy {purchase_id}\n"
                    f"Bekor qilish: /rejectbuy {purchase_id}"
                ),
            )
        except Exception:
            pass

    await query.message.reply_text("✅ So‘rov adminga yuborildi. Tasdiq kutilmoqda.")


async def approve_buy_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_admin(update.effective_user.id):
        return

    if len(context.args) != 1:
        await update.message.reply_text("Foydalanish: /approvebuy PURCHASE_ID")
        return

    purchase_id = context.args[0]
    data = load_data()

    purchase = None
    for p in data["purchases"]:
        if str(p["purchase_id"]) == purchase_id:
            purchase = p
            break

    if not purchase:
        await update.message.reply_text("Buyurtma topilmadi.")
        return
    if purchase["status"] != "pending":
        await update.message.reply_text("Bu buyurtma allaqachon ko‘rib chiqilgan.")
        return

    user_id = str(purchase["user_id"])
    item_key = purchase["item_key"]
    price = purchase["price"]

    if data["products"][item_key]["stock"] <= 0:
        await update.message.reply_text("Mahsulot stockda qolmagan.")
        return
    if user_id not in data["users"]:
        await update.message.reply_text("Foydalanuvchi topilmadi.")
        return
    if data["users"][user_id]["coins"] < price:
        await update.message.reply_text("Foydalanuvchida coin yetarli emas.")
        return

    data["users"][user_id]["coins"] -= price
    data["products"][item_key]["stock"] -= 1
    purchase["status"] = "approved"

    if item_key == "premium":
        data["users"][user_id]["premium"] = True

    save_data(data)

    await update.message.reply_text("✅ Xarid tasdiqlandi.")
    try:
        await context.bot.send_message(
            chat_id=int(user_id),
            text=(
                f"✅ Xaridingiz tasdiqlandi.\n"
                f"Mahsulot: {purchase['item_title']}\n"
                f"Yechilgan coin: {price}"
            ),
        )
    except Exception:
        pass


async def reject_buy_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_admin(update.effective_user.id):
        return

    if len(context.args) != 1:
        await update.message.reply_text("Foydalanish: /rejectbuy PURCHASE_ID")
        return

    purchase_id = context.args[0]
    data = load_data()

    for p in data["purchases"]:
        if str(p["purchase_id"]) == purchase_id:
            p["status"] = "rejected"
            save_data(data)
            await update.message.reply_text("❌ Xarid rad etildi.")
            try:
                await context.bot.send_message(chat_id=int(p["user_id"]), text=f"❌ Xaridingiz rad etildi: {p['item_title']}")
            except Exception:
                pass
            return

    await update.message.reply_text("Buyurtma topilmadi.")


# =========================
# LESSON CALLBACKS
# =========================
async def lesson_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    user = query.from_user
    ensure_user(user)

    data = load_data()
    lesson_id = int(query.data.split(":")[1])
    course, lesson = find_lesson_by_id(data, lesson_id)
    if not lesson:
        await query.message.reply_text("Dars topilmadi.")
        return

    uid = str(user.id)
    user_courses = data["users"].get(uid, {}).get("courses", [])
    if course not in user_courses and not is_admin(user.id):
        await query.message.reply_text("Siz bu kursga yozilmagansiz.")
        return

    if lesson["type"] == "video" and lesson["content"].startswith("file_id:"):
        file_id = lesson["content"].split(":", 1)[1]
        await context.bot.send_document(
            chat_id=query.message.chat_id,
            document=file_id,
            caption=f"🎥 {lesson['content_label']}\nKurs: {course}",
        )
    else:
        await query.message.reply_text(
            f"📚 {lesson['content_label']}\n\n{lesson['content']}"
        )


async def watch_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    user = query.from_user
    ensure_user(user)

    data = load_data()
    lesson_id = int(query.data.split(":")[1])
    course, lesson = find_lesson_by_id(data, lesson_id)
    if not lesson:
        await query.message.reply_text("Dars topilmadi.")
        return

    uid = str(user.id)
    user_record = data["users"].get(uid)
    if not user_record:
        await query.message.reply_text("Avval ro‘yxatdan o‘ting.")
        return
    if course not in user_record.get("courses", []):
        await query.message.reply_text("Siz bu kursga yozilmagansiz.")
        return
    if lesson["type"] != "video":
        await query.message.reply_text("Faqat video uchun ishlaydi.")
        return

    watched_by = lesson.get("watched_by", [])
    if uid in watched_by:
        await query.message.reply_text("Siz bu video uchun coin olib bo‘lgansiz.")
        return

    reward = lesson_reward_for_user(user_record)
    watched_by.append(uid)
    lesson["watched_by"] = watched_by
    user_record["coins"] += reward
    save_data(data)

    await query.message.reply_text(
        f"✅ Video ko‘rildi deb belgilandi. Sizga {reward} coin berildi."
    )


# =========================
# EXTRA COMMANDS
# =========================
async def grade_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_admin(update.effective_user.id):
        return

    if len(context.args) != 1:
        await update.message.reply_text("Foydalanish: /grade USER_ID")
        return

    target_uid = context.args[0]
    data = load_data()
    if target_uid not in data["users"]:
        await update.message.reply_text("Foydalanuvchi topilmadi.")
        return

    reward = assignment_reward_for_user(data["users"][target_uid])
    data["users"][target_uid]["coins"] += reward

    graded = False
    for s in reversed(data["submissions"]):
        if str(s["user_id"]) == target_uid and s["status"] == "pending":
            s["status"] = "graded"
            s["coin"] = reward
            graded = True
            break

    save_data(data)

    if not graded:
        await update.message.reply_text("Pending topshiriq topilmadi, lekin coin qo‘shildi.")
    else:
        await update.message.reply_text(f"✅ Topshiriq baholandi. Berilgan coin: {reward}")
    try:
        await context.bot.send_message(
            chat_id=int(target_uid),
            text=f"✅ Vazifangiz baholandi.\nBerilgan coin: {reward}",
        )
    except Exception:
        pass


async def cancel(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Bekor qilindi.", reply_markup=get_user_main_keyboard(update.effective_user.id))
    return ConversationHandler.END


# =========================
# MAIN
# =========================
def main():
    if not TOKEN:
        raise ValueError("BOT_TOKEN environment variable bo‘sh. Tokenni muhit o‘zgaruvchisiga yozing.")

    app = ApplicationBuilder().token(TOKEN).build()

    conv_handler = ConversationHandler(
        entry_points=[
            MessageHandler(filters.Regex("^📝 Ro‘yxatdan o‘tish$"), menu_handler),
            MessageHandler(filters.Regex("^✏️ Ismni yangilash$"), menu_handler),
            MessageHandler(filters.Regex("^➕ Kurs qo‘shish$"), menu_handler),
            MessageHandler(filters.Regex("^➖ Kurs o‘chirish$"), menu_handler),
            MessageHandler(filters.Regex("^🆘 Yordam$"), menu_handler),
            MessageHandler(filters.Regex("^➕ Admin qo‘shish$"), menu_handler),
            MessageHandler(filters.Regex("^➖ Adminlikdan olish$"), menu_handler),
            MessageHandler(filters.Regex("^📝 Lokatsiya matni$"), menu_handler),
            MessageHandler(filters.Regex("^🗺 Xarita linki$"), menu_handler),
            MessageHandler(filters.Regex("^📝 Vazifa qo‘shish$"), menu_handler),
            MessageHandler(filters.Regex("^📚 Darslik qo‘shish$"), menu_handler),
            MessageHandler(filters.Regex("^📤 Vazifa topshirish$"), menu_handler),
            MessageHandler(filters.Regex("^📖 O‘qish$"), menu_handler),
            MessageHandler(filters.Regex("^🚫 Foydalanuvchini bloklash$"), menu_handler),
            MessageHandler(filters.Regex("^✅ Blokdan chiqarish$"), menu_handler),
            MessageHandler(filters.Regex("^🗑 Foydalanuvchini o‘chirish$"), menu_handler),
            MessageHandler(filters.Regex("^➕ Coin qo‘shish$"), menu_handler),
            MessageHandler(filters.Regex("^➖ Coin ayirish$"), menu_handler),
            MessageHandler(filters.Regex("^⭐ Premium berish$"), menu_handler),
            MessageHandler(filters.Regex("^❌ Premiumni olish$"), menu_handler),
        ],
        states={
            REG_FULLNAME: [MessageHandler(filters.TEXT & ~filters.COMMAND, reg_fullname)],
            REG_PHONE: [
                MessageHandler(filters.CONTACT, reg_phone),
                MessageHandler(filters.TEXT & ~filters.COMMAND, reg_phone),
            ],
            REG_COURSE: [MessageHandler(filters.TEXT & ~filters.COMMAND, reg_course)],
            UPDATE_NAME: [MessageHandler(filters.TEXT & ~filters.COMMAND, update_name_handler)],
            ADD_COURSE: [MessageHandler(filters.TEXT & ~filters.COMMAND, add_course_handler)],
            REMOVE_COURSE: [MessageHandler(filters.TEXT & ~filters.COMMAND, remove_course_handler)],
            HELP_MESSAGE: [MessageHandler((filters.TEXT | filters.PHOTO) & ~filters.COMMAND, help_handler)],
            ADMIN_ADD_ADMIN: [MessageHandler(filters.TEXT & ~filters.COMMAND, admin_add_admin_handler)],
            ADMIN_REMOVE_ADMIN: [MessageHandler(filters.TEXT & ~filters.COMMAND, admin_remove_admin_handler)],
            ADMIN_SET_LOCATION_TEXT: [MessageHandler(filters.TEXT & ~filters.COMMAND, admin_set_location_text_handler)],
            ADMIN_SET_LOCATION_MAP: [MessageHandler(filters.TEXT & ~filters.COMMAND, admin_set_location_map_handler)],
            ADMIN_SET_TASK_COURSE: [MessageHandler(filters.TEXT & ~filters.COMMAND, admin_task_course_handler)],
            ADMIN_SET_TASK_TEXT: [MessageHandler(filters.TEXT & ~filters.COMMAND, admin_task_text_handler)],
            ADMIN_ADD_LESSON_COURSE: [MessageHandler(filters.TEXT & ~filters.COMMAND, admin_lesson_course_handler)],
            ADMIN_ADD_LESSON_TYPE: [MessageHandler(filters.TEXT & ~filters.COMMAND, admin_lesson_type_handler)],
            ADMIN_ADD_LESSON_CONTENT: [MessageHandler((filters.TEXT | filters.Document.ALL) & ~filters.COMMAND, admin_lesson_content_handler)],
            SUBMIT_TASK: [MessageHandler(filters.Document.ALL & ~filters.COMMAND, submit_task_handler)],
            ADMIN_BLOCK_USER: [MessageHandler(filters.TEXT & ~filters.COMMAND, admin_block_user_handler)],
            ADMIN_UNBLOCK_USER: [MessageHandler(filters.TEXT & ~filters.COMMAND, admin_unblock_user_handler)],
            ADMIN_DELETE_USER: [MessageHandler(filters.TEXT & ~filters.COMMAND, admin_delete_user_handler)],
            ADMIN_COIN_USER: [MessageHandler(filters.TEXT & ~filters.COMMAND, admin_coin_user_handler)],
            ADMIN_COIN_AMOUNT: [MessageHandler(filters.TEXT & ~filters.COMMAND, admin_coin_amount_handler)],
            ADMIN_PREMIUM_USER: [MessageHandler(filters.TEXT & ~filters.COMMAND, admin_premium_user_handler)],
            ADMIN_REMOVE_PREMIUM_USER: [MessageHandler(filters.TEXT & ~filters.COMMAND, admin_remove_premium_user_handler)],
            LEARN_SELECT_COURSE: [MessageHandler(filters.TEXT & ~filters.COMMAND, learn_select_course_handler)],
        },
        fallbacks=[CommandHandler("cancel", cancel)],
    )

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("grade", grade_command))
    app.add_handler(CommandHandler("approvebuy", approve_buy_command))
    app.add_handler(CommandHandler("rejectbuy", reject_buy_command))
    app.add_handler(CallbackQueryHandler(buy_callback, pattern=r"^buy:"))
    app.add_handler(CallbackQueryHandler(lesson_callback, pattern=r"^lesson:"))
    app.add_handler(CallbackQueryHandler(watch_callback, pattern=r"^watch:"))
    app.add_handler(conv_handler)
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, menu_handler))

    print("Bot ishga tushdi...")
    app.run_polling()


if __name__ == "__main__":
    main()
