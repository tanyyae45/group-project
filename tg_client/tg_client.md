```python
import logging
import json
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ApplicationBuilder, CommandHandler, CallbackQueryHandler, ContextTypes, MessageHandler, filters
from uuid import uuid4  

# Настройка логирования
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)
logger = logging.getLogger(__name__)


# --- Структуры данных ---
class Test:
    def __init__(self, title, questions):
        self.id = str(uuid4())
        self.title = title
        self.questions = questions

class Question:
    def __init__(self, text, options, correct_option_index):
        self.text = text
        self.options = options
        self.correct_option_index = correct_option_index
class UserResult:
    def __init__(self, user_id, test_id):
        self.user_id = user_id
        self.test_id = test_id
        self.answers = {}
        self.score = 0

    def calculate_score(self, test):
        self.score = 0
        for question_index, answer_index in self.answers.items():
          if question_index < len(test.questions) and answer_index == test.questions[question_index].correct_option_index:
             self.score += 1
        return self.score

# --- Хранилище данных ---
tests = {}
user_results = {}  

# Пример теста по программированию
default_test = Test(
    title="Тест по основам программирования",
    questions=[
        Question("Какой оператор используется для вывода в Python?", ["print", "input", "echo", "display"], 0),
        Question("Что такое переменная?", ["Константа", "Именованное место в памяти", "Математическая функция", "Объект"], 1),
         Question("Какой тип данных является неизменяемым?", ["Список", "Словарь", "Кортеж", "Множество"], 2),
          Question("Что делает цикл 'for'?", ["Итерация по элементам", "Проверка условия", "Определение функции", "Создание нового объекта"], 0),
    ]
)
tests[default_test.id] = default_test # Сохранение теста

# --- Функции обработки ---

async def start_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Обработчик команды /start"""
    user = update.effective_user
    await update.message.reply_text(f"Привет, {user.first_name}! Я бот для тестирования. \nИспользуй /tests чтобы увидеть список тестов, /results - для просмотра результатов.\nДля создания собственного теста используй /create_test.")

async def tests_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
  """Обработчик команды /tests для просмотра доступных тестов."""
  keyboard = []
  for test_id, test in tests.items():
    keyboard.append([InlineKeyboardButton(text=test.title, callback_data=f"test_{test_id}")])
  if keyboard:
      reply_markup = InlineKeyboardMarkup(keyboard)
      await update.message.reply_text("Выберите тест:", reply_markup=reply_markup)
  else:
      await update.message.reply_text("Нет доступных тестов.")
async def create_test_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Обработчик команды /create_test"""
    await update.message.reply_text("Начнем создание теста. Пожалуйста, введите название теста:")
    context.user_data['create_test_stage'] = 'title'
    context.user_data['new_test'] = {}

async def process_test_title(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Обработчик названия теста"""
    title = update.message.text
    context.user_data['new_test']['title'] = title
    await update.message.reply_text("Отлично. Теперь введите первый вопрос:")
    context.user_data['create_test_stage'] = 'question_text'
    context.user_data['new_test']['questions'] = []
    context.user_data['question_counter'] = 0

async def process_test_question_text(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Обработчик текста вопроса"""
    question_text = update.message.text
    context.user_data['new_test']['current_question'] = {'text':question_text, 'options':[]}
    await update.message.reply_text("Введите первый вариант ответа:")
    context.user_data['create_test_stage'] = 'option'
    context.user_data['option_counter'] = 0
async def process_test_option(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Обработчик варианта ответа"""
    option = update.message.text
    context.user_data['new_test']['current_question']['options'].append(option)
    context.user_data['option_counter'] += 1

    if context.user_data['option_counter'] < 4: # 4 варианта ответа для примера
       await update.message.reply_text(f"Введите вариант ответа #{context.user_data['option_counter'] + 1}:")
    else:
      await update.message.reply_text("Введите номер правильного варианта ответа (1,2,3 или 4):")
      context.user_data['create_test_stage'] = 'correct_option'

async def process_test_correct_option(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
     """Обработчик номера правильного ответа."""
     try:
       correct_option_index = int(update.message.text) - 1
       if 0 <= correct_option_index < len(context.user_data['new_test']['current_question']['options']):
           context.user_data['new_test']['current_question']['correct_option_index'] = correct_option_index
           context.user_data['new_test']['questions'].append(Question(
                 text = context.user_data['new_test']['current_question']['text'],
                 options=context.user_data['new_test']['current_question']['options'],
                 correct_option_index = correct_option_index
           ))
           context.user_data['question_counter'] += 1
           await update.message.reply_text(f"Вопрос {context.user_data['question_counter']} добавлен. Если хотите добавить еще вопрос, введите его текст, иначе введите 'готово' чтобы закончить")
           context.user_data['create_test_stage'] = 'next_question_or_done'
       else:
          await update.message.reply_text("Некорректный номер варианта ответа. Попробуйте снова.")

     except ValueError:
       await update.message.reply_text("Некорректный ввод, введите целое число")

async def process_test_next_question_or_done(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
     """Обработчик решения, создавать еще вопрос или нет"""
     if update.message.text.lower() == "готово":
         new_test = Test(
             title = context.user_data['new_test']['title'],
             questions = context.user_data['new_test']['questions']
             )   
         tests[new_test.id] = new_test
         await update.message.reply_text("Тест успешно создан!")
         context.user_data.clear() # Clear user data
     else:
         context.user_data['create_test_stage'] = 'question_text'
         context.user_data['new_test']['current_question'] = {}
         await update.message.reply_text("Введите следующий вопрос:")
async def results_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Обработчик команды /results для просмотра результатов"""
    user_id = update.effective_user.id
    if user_id in user_results:
        results = user_results[user_id]
        message = "Ваши результаты:\n"
        for test_id, result in results.items():
            test = tests.get(test_id)
            if test:
                 score = result.calculate_score(test)
                 message += f"- Тест: {test.title}, Балл: {score}/{len(test.questions)}\n"
        await update.message.reply_text(message)
    else:
        await update.message.reply_text("У вас пока нет результатов.")
async def handle_test_callback(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Обработчик нажатия на кнопку теста"""
    query = update.callback_query
    await query.answer()
    test_id = query.data.split("_")[1]
    test = tests.get(test_id)
    if test:
        context.user_data['current_test_id'] = test_id
        context.user_data['current_question_index'] = 0
        context.user_data['user_answers'] = {} # Очистка ответов для нового теста
        await show_question(update, context)
    else:
        await query.message.reply_text("Тест не найден")

async def show_question(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Отображает текущий вопрос теста"""
    query = update.callback_query
    if query:
         await query.answer() 
    test_id = context.user_data['current_test_id']
    test = tests.get(test_id)
    question_index = context.user_data['current_question_index']

    if test and question_index < len(test.questions):
        question = test.questions[question_index]
        keyboard = []
        for i, option in enumerate(question.options):
            keyboard.append([InlineKeyboardButton(text=option, callback_data=f"answer_{i}")])
        reply_markup = InlineKeyboardMarkup(keyboard)
        if query:
             await query.message.reply_text(f"Вопрос {question_index + 1}/{len(test.questions)}:\n{question.text}", reply_markup=reply_markup)
        else:
              await update.message.reply_text(f"Вопрос {question_index + 1}/{len(test.questions)}:\n{question.text}", reply_markup=reply_markup)
    else:
        await finish_test(update, context)
async def handle_answer_callback(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Обработчик нажатия на кнопку с вариантом ответа"""
    query = update.callback_query
    await query.answer()
    answer_index = int(query.data.split("_")[1])
    question_index = context.user_data['current_question_index']
    user_answers = context.user_data['user_answers']
    user_answers[question_index] = answer_index
    context.user_data['current_question_index'] += 1
    await show_question(update, context)

async def finish_test(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Завершает тест и сохраняет результаты."""
    query = update.callback_query
    user_id = update.effective_user.id
    test_id = context.user_data['current_test_id']
    user_answers = context.user_data['user_answers']
    test = tests.get(test_id)


    if user_id not in user_results:
       user_results[user_id] = {}

    result = UserResult(user_id, test_id)
    result.answers = user_answers

    user_results[user_id][test_id] = result

    score = result.calculate_score(test)

    await (query.message.reply_text if query else update.message.reply_text)(
      f"Тест завершен! Ваш балл: {score}/{len(test.questions)}."
    )
    context.user_data.clear()


async def text_message_handler(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
  """Обработчик текстовых сообщений для этапов создания теста."""
  if 'create_test_stage' in context.user_data:
      stage = context.user_data['create_test_stage']
      if stage == 'title':
          await process_test_title(update, context)
      elif stage == 'question_text':
         await process_test_question_text(update, context)
      elif stage == 'option':
         await process_test_option(update, context)
      elif stage == 'correct_option':
         await process_test_correct_option(update, context)
      elif stage == 'next_question_or_done':
         await process_test_next_question_or_done(update,context)
      else:
         await update.message.reply_text("Неожиданный ввод, используй команды")
  else:
      await update.message.reply_text("Я не понимаю, используй команды")

async def error_handler(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
  """Обработчик ошибок."""
  logger.error(f"Обновление '{update}' вызвало ошибку '{context.error}'")
  await update.message.reply_text("Произошла ошибка. Попробуйте еще раз.")

# --- Запуск бота ---
def main() -> None:
    TOKEN = '7044996466:AAEVOqQuwjKQma91-vMdjXanRMmm4WbdwKY' 
    application = ApplicationBuilder().token(TOKEN).build()

    application.add_handler(CommandHandler("start", start_command))
    application.add_handler(CommandHandler("tests", tests_command))
    application.add_handler(CommandHandler("results", results_command))
    application.add_handler(CommandHandler("create_test",create_test_command))
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, text_message_handler))

    application.add_handler(CallbackQueryHandler(handle_test_callback, pattern="^test_"))
    application.add_handler(CallbackQueryHandler(handle_answer_callback, pattern="^answer_"))

    application.add_error_handler(error_handler)

    application.run_polling()

if __name__ == '__main__':
    main()
```
