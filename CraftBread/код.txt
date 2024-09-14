import logging
from telegram import Update, ReplyKeyboardMarkup, KeyboardButton, ReplyKeyboardRemove
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes, ConversationHandler
from telegram.error import BadRequest

# Настройка логирования
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)

# Определение состояний для ConversationHandler
MENU, ORDER, GET_PHONE, GET_LOCATION, VIEW_CART, CONFIRM_ORDER, DRINKS, HOME_FOOD = range(8)

# Пример меню (можно заменить на запрос к базе данных или API)
home_food_menu = {
    "🍚 Плов узбекский (говядина)": 99,
    "🍝 Макароны по-флотски (говядина)": 99,
    "🥪 Самса 3шт. (говядина)": 99,
    "🍩 Беляши 2шт.": 99,
    "🥟 Манты 3шт. (говядина)": 99,
    "🧆 Котлеты с пюре 2шт. (говядина)": 99,
    "🫓 Блинчики 3шт.": 50,
    "🫔 Блинчики с мясом 3шт. (говядина)": 99,
    "🌭 Сосиска в тесте (куриная)": 30,
    "🍞 Хлеб Тартин (на закваске)": 60,
    "🍞 Хлеб Чиабатта": 30,
    "🍞 Хлеб Белый нарезка 2шт. (на закваске)": 20,
    "🍞 Хлеб Черный солодовый нарезка 2шт. (на закваске)": 20
}

drinks_menu = {
    "🫖 Чай": 20,
    "☕️ Кофе": 30,
    "🧃 Сок": 25,
    "🥛 Вода": 10
}

# Корзина пользователя
user_cart = {}

# Хранение номеров телефонов и геолокаций пользователей
user_phone_numbers = {}
user_locations = {}

# Телеграм пользователя-администратора для уведомлений
admin_chat_id = -4138037091  # Замените на ваш числовой chat ID

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    reply_keyboard = [['📋 Ассортимент', '🛒 Заказать'], ['🛍 Корзина']]
    await update.message.reply_text(
        'Добро пожаловать в наш магазин! Чем могу помочь?',
        reply_markup=ReplyKeyboardMarkup(reply_keyboard, one_time_keyboard=True, resize_keyboard=True)
    )
    return MENU

async def show_menu(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    reply_keyboard = [['🥤 Напитки', '🍲 Домашняя еда'], ['🔙 Назад']]
    await update.message.reply_text(
        'Выберите категорию:',
        reply_markup=ReplyKeyboardMarkup(reply_keyboard, one_time_keyboard=True, resize_keyboard=True)
    )
    return MENU

async def show_drinks(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    menu_text = "Вот наши напитки:\n"
    for item, price in drinks_menu.items():
        menu_text += f"{item}: {price} бат\n"
    await update.message.reply_text(menu_text)
    reply_keyboard = [[item] for item in drinks_menu.keys()]
    reply_keyboard.append(['🛍 Корзина', '🔙 Назад'])
    await update.message.reply_text(
        'Что вы хотите заказать?',
        reply_markup=ReplyKeyboardMarkup(reply_keyboard, one_time_keyboard=True, resize_keyboard=True)
    )
    return ORDER

async def show_home_food(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    menu_text = "Вот наша домашняя еда:\n"
    for item, price in home_food_menu.items():
        menu_text += f"{item}: {price} бат\n"
    await update.message.reply_text(menu_text)
    reply_keyboard = [[item] for item in home_food_menu.keys()]
    reply_keyboard.append(['🛍 Корзина', '🔙 Назад'])
    await update.message.reply_text(
        'Что вы хотите заказать?',
        reply_markup=ReplyKeyboardMarkup(reply_keyboard, one_time_keyboard=True, resize_keyboard=True)
    )
    return ORDER

async def order(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    item = update.message.text
    if item in home_food_menu:
        user_cart[item] = user_cart.get(item, 0) + 1
        await update.message.reply_text(f"{item} добавлен в вашу корзину.\nХотите что-нибудь еще?", 
                                        reply_markup=ReplyKeyboardMarkup([[item] for item in home_food_menu.keys()] + [['🛍 Корзина', '🔙 Назад']], one_time_keyboard=True, resize_keyboard=True))
        return ORDER
    elif item in drinks_menu:
        user_cart[item] = user_cart.get(item, 0) + 1
        await update.message.reply_text(f"{item} добавлен в вашу корзину.\nХотите что-нибудь еще?", 
                                        reply_markup=ReplyKeyboardMarkup([[item] for item in drinks_menu.keys()] + [['🛍 Корзина', '🔙 Назад']], one_time_keyboard=True, resize_keyboard=True))
        return ORDER
    elif item == '🔙 Назад':
        return await start(update, context)
    elif item == '🛍 Корзина':
        return await view_cart(update, context)
    else:
        await update.message.reply_text("Пожалуйста, выберите из Ассортимент или нажмите '🛍 Корзина' или '🔙 Назад'.")
        return ORDER

async def view_cart(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    user_id = update.message.from_user.id

    cart_text = "Ваша корзина:\n"
    for item, quantity in user_cart.items():
        cart_text += f"{item} x {quantity}\n"
    if not user_cart:
        cart_text = "Ваша корзина пуста."
    await update.message.reply_text(cart_text)

    # Если у пользователя еще нет номера телефона, запрашиваем его
    if user_id not in user_phone_numbers:
        await update.message.reply_text('Пожалуйста, отправьте ваш номер телефона.', 
                                        reply_markup=ReplyKeyboardMarkup([[KeyboardButton("📞 Отправить номер телефона", request_contact=True)], ['🔙 Назад']], one_time_keyboard=True, resize_keyboard=True))
        return GET_PHONE
    
    # Если у пользователя еще нет геопозиции, запрашиваем ее
    if user_id not in user_locations:
        await update.message.reply_text('Пожалуйста, отправьте вашу геолокацию.', 
                                        reply_markup=ReplyKeyboardMarkup([[KeyboardButton("📍 Отправить геолокацию", request_location=True)], ['🔙 Назад']], one_time_keyboard=True, resize_keyboard=True))
        return GET_LOCATION
    
    # Если оба параметра уже есть, переходим к подтверждению заказа
    await update.message.reply_text("Введите '✅ Подтвердить заказ' для завершения или '🔙 Назад' для возврата в Ассортимент.", 
                                    reply_markup=ReplyKeyboardMarkup([['✅ Подтвердить заказ', '🔙 Назад']], one_time_keyboard=True, resize_keyboard=True))
    return CONFIRM_ORDER

async def confirm_order(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    user_id = update.message.from_user.id
    text = update.message.text

    if text == '✅ Подтвердить заказ':
        cart_text = "Ваш заказ:\n"
        for item, quantity in user_cart.items():
            cart_text += f"{item} x {quantity}\n"
        
        user_phone = user_phone_numbers.get(user_id, 'Не предоставлен')
        user_location = user_locations.get(user_id)
        location_text = f"{user_location.latitude}, {user_location.longitude}" if user_location else 'Не предоставлена'
        
        order_text = f"{cart_text}\nТелефон: {user_phone}\nГеолокация: {location_text}"

        try:
            await context.bot.send_message(chat_id=admin_chat_id, text=order_text)
            logging.info(f"Sent order to group: {order_text}")
            if user_location:
                await context.bot.send_location(chat_id=admin_chat_id, latitude=user_location.latitude, longitude=user_location.longitude)
                logging.info(f"Sent location to group: {user_location.latitude}, {user_location.longitude}")
        except BadRequest as e:
            logging.error(f"Failed to send message to group: {e}")
            await update.message.reply_text("Произошла ошибка при отправке вашего заказа. Пожалуйста, попробуйте снова.")
            return CONFIRM_ORDER
        
        await update.message.reply_text("Спасибо за ваш заказ! Мы свяжемся с вами для подтверждения.")
        user_cart.clear()
        return await start(update, context)
    elif text == '🔙 Назад':
        return await start(update, context)
    else:
        await update.message.reply_text("Пожалуйста, выберите '✅ Подтвердить заказ' или '🔙 Назад'.")
        return CONFIRM_ORDER

async def get_phone(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    if update.message.contact:
        user_id = update.message.from_user.id
        user_phone_numbers[user_id] = update.message.contact.phone_number
        await update.message.reply_text(f'Спасибо! Ваш номер телефона: {update.message.contact.phone_number}.')
        return await view_cart(update, context)
    elif update.message.text == '🔙 Назад':
        return await start(update, context)
    else:
        await update.message.reply_text("Пожалуйста, отправьте ваш номер телефона, используя кнопку ниже.")
        return GET_PHONE

async def get_location(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    if update.message.location:
        user_id = update.message.from_user.id
        user_locations[user_id] = update.message.location
        await update.message.reply_text(f'Спасибо! Мы доставим ваш заказ по адресу: {update.message.location.latitude}, {update.message.location.longitude}.')
        return await view_cart(update, context)
    elif update.message.text == '🔙 Назад':
        return await start(update, context)
    else:
        await update.message.reply_text("Пожалуйста, отправьте вашу геолокацию, используя кнопку ниже.")
        return GET_LOCATION

async def cancel(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    await update.message.reply_text('До свидания! Надеемся увидеть вас снова.', reply_markup=ReplyKeyboardRemove())
    return ConversationHandler.END

def main() -> None:
    application = Application.builder().token("7206388222:AAHdtplOSWT8G5b88-iIMeXIr9Xn85GWPAk").build()

    conv_handler = ConversationHandler(
        entry_points=[CommandHandler('start', start)],
        states={
            MENU: [
                MessageHandler(filters.Regex('^(📋 Ассортимент)$'), show_menu),
                MessageHandler(filters.Regex('^(🛒 Заказать)$'), show_menu),
                MessageHandler(filters.Regex('^(🥤 Напитки)$'), show_drinks),
                MessageHandler(filters.Regex('^(🍲 Домашняя еда)$'), show_home_food),
                MessageHandler(filters.Regex('^(🛍 Корзина)$'), view_cart),
                MessageHandler(filters.Regex('^(🔙 Назад)$'), start),  # Добавлено для обработки кнопки "Назад"
            ],
            ORDER: [
                MessageHandler(filters.TEXT & ~filters.COMMAND, order),
                MessageHandler(filters.Regex('^(🔙 Назад)$'), start),  # Добавлено для обработки кнопки "Назад"
            ],
            CONFIRM_ORDER: [
                MessageHandler(filters.Regex('^(✅ Подтвердить заказ)$'), confirm_order),
                MessageHandler(filters.Regex('^(🔙 Назад)$'), start),  # Добавлено для обработки кнопки "Назад"
            ],
            GET_PHONE: [
                MessageHandler(filters.CONTACT, get_phone),
                MessageHandler(filters.Regex('^(🔙 Назад)$'), start),  # Добавлено для обработки кнопки "Назад"
            ],
            GET_LOCATION: [
                MessageHandler(filters.LOCATION, get_location),
                MessageHandler(filters.Regex('^(🔙 Назад)$'), start),  # Добавлено для обработки кнопки "Назад"
            ],
        },
        fallbacks=[CommandHandler('cancel', cancel)],
    )

    application.add_handler(conv_handler)

    application.run_polling()

if __name__ == '__main__':
    main()
