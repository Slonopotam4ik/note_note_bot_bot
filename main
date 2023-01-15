import os

from aiogram import Bot, types
from aiogram.utils import executor
from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton
from aiogram.dispatcher import Dispatcher, FSMContext
from aiogram.dispatcher.filters.state import State, StatesGroup
from aiogram.contrib.fsm_storage.memory import MemoryStorage
from aiogram.dispatcher.filters import Text

storage = MemoryStorage()

bot = Bot("token")
dp = Dispatcher(bot, storage=storage)


def update_main_kb():
    main_kb = InlineKeyboardMarkup()
    for i in os.listdir(os.getcwd()):
        main_kb.insert(InlineKeyboardButton(str(i).split(".")[0], callback_data=str(i)))
    main_kb.add(InlineKeyboardButton("Выход", callback_data="exit"))
    main_kb.insert(InlineKeyboardButton("Добавить", callback_data="New"))
    return main_kb


def get_message_data(message: types.Message):
    return message.chat.id, message.message_id


class AddNote(StatesGroup):
    text = State()
    name = State()


@dp.message_handler(commands=["start"])
async def start_bot(message: types.Message):
    await message.answer("Мои заметки, введите пароль")
    await bot.delete_message(*get_message_data(message))


async def create_note_file(name, text, message):
    with open(f"{name}.txt", encoding="utf-8", mode="w") as note:
        note.write(text)
    await bot.edit_message_text("Заметки",
                                message.chat.id,
                                message.message_id - 1,
                                reply_markup=update_main_kb())
    await bot.delete_message(*get_message_data(message))


@dp.callback_query_handler(Text(startswith="back"))
async def back(callback: types.CallbackQuery, state: FSMContext):
    await state.finish()

    await bot.edit_message_text("Заметки",
                                *get_message_data(callback.message),
                                reply_markup=update_main_kb())


@dp.callback_query_handler(Text(startswith="New"))
async def create_note(callback: types.CallbackQuery):
    await AddNote.text.set()
    await bot.edit_message_text("Введите текст заметки. Первая строка будет использоваться в качестве названия",
                                *get_message_data(callback.message)
                                )


note_texts = ""
note_name = ""


@dp.callback_query_handler(Text(startswith="remove_"))
async def remove_note(callback: types.CallbackQuery):
    global note_texts, note_name
    if callback.data.split("_")[1] == "yes":
        os.remove(note_name)
        await bot.edit_message_text("Заметки",
                                    *get_message_data(callback.message),
                                    reply_markup=update_main_kb())

    elif callback.data.split("_")[1] == "no":
        note_setting_kb = InlineKeyboardMarkup()
        note_setting_kb.insert(InlineKeyboardButton("Удалить", callback_data=f"remove_{note_name}"))
        note_setting_kb.add(InlineKeyboardButton("Назад", callback_data=f"back"))
        await bot.edit_message_text(note_texts,
                                    *get_message_data(callback.message),
                                    reply_markup=note_setting_kb)
    else:
        note_texts = callback.message.text
        note_name = callback.data.split("_")[1]
        note_kb = InlineKeyboardMarkup()
        note_kb.insert(InlineKeyboardButton("Да", callback_data="remove_yes"))
        note_kb.insert(InlineKeyboardButton("Нет", callback_data="remove_no"))
        await bot.edit_message_text("Вы точно хотите удалить?",
                                    *get_message_data(callback.message),
                                    reply_markup=note_kb)


@dp.callback_query_handler(Text(startswith="exit"))
async def exit_from_note(callback: types.CallbackQuery):
    await bot.delete_message(*get_message_data(callback.message))


@dp.message_handler(state=AddNote.text)
async def create_note_text(message: types.Message, state: FSMContext):
    async with state.proxy() as note_data:
        note_data["note_name"] = message.text.split("\n")[0]
        note_data["note_text"] = "\n".join(message.text.split("\n")[1:])
    for i in os.listdir(os.getcwd()):
        if i.split(".")[0] == note_data["note_name"]:
            await bot.edit_message_text("Название занято, введите другое",
                                        message.chat.id,
                                        message.message_id - 1,
                                        )
            await AddNote.next()
            return
    await state.finish()
    await create_note_file(note_data["note_name"], note_data["note_text"], message)


@dp.message_handler(state=AddNote.name)
async def create_note_name(message: types.Message, state: FSMContext):
    async with state.proxy() as note_data:
        if note_data["note_name"] == message.text.split("\n")[0]:
            return
        note_data["note_name"] = message.text.split("\n")[0]
    await state.finish()
    await create_note_file(note_data["note_name"], note_data["note_text"], message)


@dp.callback_query_handler()
async def open_note(callback: types.CallbackQuery):
    get_note_name = callback.data
    note_setting_kb = InlineKeyboardMarkup()
    note_setting_kb.insert(InlineKeyboardButton("Удалить", callback_data=f"remove_{get_note_name}"))
    note_setting_kb.add(InlineKeyboardButton("Назад", callback_data=f"back"))
    await bot.edit_message_text(get_note_name.split(".")[0],
                                callback.message.chat.id,
                                callback.message.message_id)
    with open(f"{get_note_name}", encoding="utf-8") as note:
        await bot.edit_message_text(note.read(),
                                    *get_message_data(callback.message),
                                    reply_markup=note_setting_kb)


@dp.message_handler()
async def start(message: types.Message):
    await bot.delete_message(*get_message_data(message))

    if message.text == "********":
        await message.answer("Заметки", reply_markup=update_main_kb())


if __name__ == '__main__':
    os.chdir("Notes")
    executor.start_polling(dp, skip_updates=False)
