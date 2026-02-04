# Часть 1. Архитектура компонента

## QuestionCard (React component)

    ├── QuestionHeader (questionId, metadata)
    ├── QuestionContent (TipTapRenderer + KaTeX integration)
    ├── AnswerOptions
    │ ├── OptionItem (кликабельные варианты)
    │ │ ├── TextOption
    │ │ ├── ImageOption (если есть)
    │ │ └── MathOption (KaTeX для формул)
    │ └── LoadingSkeleton (при загрузке)
    ├── ActionBar
    │ ├── CheckButton (проверить ответ)
    │ ├── HintButton (опционально)
    │ └── NextButton (после проверки)
    ├── Explanation (conditional, TipTapRenderer + KaTeX)
    └── ErrorBoundary (обработка ошибок рендеринга)

## Локальное состояние (useState в QuestionCard)

    const [selectedAnswer, setSelectedAnswer] = useState(null); // ID выбранного варианта
    const [isChecking, setIsChecking] = useState(false); // Флаг отправки на проверку
    const [localIsChecked, setLocalIsChecked] = useState(false); // Локальный флаг проверки
    const [showExplanation, setShowExplanation] = useState(false); // Показать объяснение
    const [error, setError] = useState(null); // Ошибки валидации/загрузки

## Глобальное состояние (Redux / Zustand / Context API)

    // questionStore (Zustand пример)
    const useQuestionStore = create((set) => ({
    currentQuestion: null, // Текущий вопрос
    questionsCache: new Map(), // Кеш загруженных вопросов
    userAnswers: new Map(), // История ответов {questionId: {selected, isCorrect, timestamp}}
    apiLoading: false, // Общий флаг загрузки API

    // Actions
    setCurrentQuestion: (question) => set({ currentQuestion: question }),
    addToCache: (id, question) => set((state) => {
    const newCache = new Map(state.questionsCache);
    newCache.set(id, question);
    return { questionsCache: newCache };
    }),
    }));

    1. где хранится selectedAnswer?
    Хранится локально в состоянии компонента через useState

    2. где хранится isChecked
    Два уровня состояния:
    Локальный (localIsChecked): useState в компоненте
    Показывает, проверен ли текущий вопрос прямо сейчас
    Сбрасывается при смене вопроса

    Глобальный (serverIsChecked): в Redux/Zustand хранилище
    Показывает, был ли вопрос проверен ранее
    Сохраняется при навигации между вопросами

    3. что сбрасывается при смене questionId

    Полностью сбрасывается локальное состояние:

    jsx
    useEffect(() => {
    // Сброс при изменении questionId
    setSelectedAnswer(null);
    setLocalIsChecked(false);
    setShowExplanation(false);
    setIsChecking(false);
    setError(null);

    // Загружаем новый вопрос
    loadQuestion(newQuestionId);
    }, [questionId]);

    4. что будет, если пользователь кликает очень быстро
    Race condition: ответ API на старый вопрос может прийти позже и перезаписать состояние нового
    Некорректный UI: отобразится выбор/результат для неактивного вопроса
    Утечки памяти: незавершенные запросы, неотменённые side effects
    Множественные перерендеры: производительность страдает

# Часть 2. Псевдокод логики

    // Основные состояния
    const [selectedAnswer, setSelectedAnswer] = useState(null);
    const [isChecking, setIsChecking] = useState(false);
    const [isChecked, setIsChecked] = useState(false);
    const [showExplanation, setShowExplanation] = useState(false);
    const [questionData, setQuestionData] = useState(null);
    const [isLoading, setIsLoading] = useState(false);

    // Получение данных из глобального хранилища
    const { currentQuestionId, addUserAnswer } = useQuestionStore();

    // Проверка доступности кнопок
    const isCheckButtonDisabled = !selectedAnswer || isChecking || isChecked;
    const isNextButtonDisabled = !isChecked;
    const areOptionsDisabled = isChecked || isChecking;

    // Сброс состояния при смене вопроса
    useEffect(() => {
    const resetState = () => {
    setSelectedAnswer(null);
    setIsChecking(false);
    setIsChecked(false);
    setShowExplanation(false);
    };

    resetState();
    loadQuestionData(currentQuestionId);
    }, [currentQuestionId]);

    // Выбор ответа
    const handleSelectAnswer = (answerId) => {
    // Можно выбирать только если ответ еще не проверен
    if (!isChecked && !isChecking) {
    setSelectedAnswer(answerId);
    }
    };

    // Проверка ответа
    const handleCheckAnswer = async () => {
    if (!selectedAnswer || isChecking || isChecked) return;

    setIsChecking(true);

    try {
    // Отправка запроса на проверку с защитой от race condition
    const result = await api.checkAnswer({
    questionId: currentQuestionId,
    answer: selectedAnswer
    });

        // Сохраняем результат в глобальное хранилище
        addUserAnswer({
        questionId: currentQuestionId,
        answer: selectedAnswer,
        isCorrect: result.isCorrect,
        explanation: result.explanation
        });

        setIsChecked(true);
        setIsChecking(false);

        // Автоматически показываем explanation если ответ неправильный
        if (!result.isCorrect) {
        setShowExplanation(true);
        }

    } catch (error) {
    setIsChecking(false);
    showErrorToast('Ошибка проверки ответа');
    }
    };

    // Показ/скрытие объяснения
    const handleToggleExplanation = () => {
    // Можно показывать explanation только после проверки
    if (isChecked) {
    setShowExplanation(!showExplanation);
    }
    };

    // Переход к следующему вопросу
    const handleNextQuestion = () => {
    if (!isChecked) return;

    // Переход осуществляется через глобальное состояние
    goToNextQuestion();
    };

    // Рендер компонента
    return (

    <div>
        {/* Заголовок вопроса */}
        <QuestionHeader
        questionId={currentQuestionId}
        isLoading={isLoading}
        />

        {/* Контент вопроса с TipTap */}
        <QuestionContent
        content={questionData?.content}
        isLoading={isLoading}
        />

        {/* Варианты ответов */}
        <AnswerOptions
        options={questionData?.options}
        selectedAnswer={selectedAnswer}
        isCorrect={isChecked && userAnswer?.isCorrect}
        isChecked={isChecked}
        disabled={areOptionsDisabled}
        onSelect={handleSelectAnswer}
        />

        {/* Панель действий */}
        <ActionBar>
        <CheckButton
            onClick={handleCheckAnswer}
            disabled={isCheckButtonDisabled}
            loading={isChecking}
        />

        {isChecked && (
            <>
            <ShowExplanationButton
                onClick={handleToggleExplanation}
                isActive={showExplanation}
            />

            <NextButton
                onClick={handleNextQuestion}
                disabled={isNextButtonDisabled}
            />
            </>
        )}
        </ActionBar>

        {/* Блок с объяснением (условный рендеринг) */}
        {showExplanation && isChecked && (
        <Explanation
            content={userAnswer?.explanation}
            isCorrect={userAnswer?.isCorrect}
        />
        )}

        {/* Индикатор загрузки при быстрых кликах */}
        {isLoading && <LoadingOverlay />}
    </div>
    );

# Часть 3. Edge cases и UX

1. Explanation отсутствует

- UI поведение:
- Кнопка "Показать объяснение" не отображается или disabled
- При клике на "Проверить" (если ответ неверный) показывается:
- "К этому вопросу пока нет объяснения"
- Если ответ правильный — сообщение не показывается
- В истории ответов сохраняется флаг hasExplanation: false

2. В stem только формулы (без текста)

- UI поведение:
- Рендерится только KaTeX блок(и)
- Сохраняются стандартные отступы и контейнер
- Добавляется aria-label: "Вопрос содержит математические формулы"
- Проверка доступности: формулы имеют alt-текст описания

3. В stem очень длинный текст

- UI поведение:
- Максимальная высота блока с контентом (например, 400px)
- Автоматическое появление вертикального скроллбара
- Чёткий визуальный индикатор обрезанного контента (градиент fade-out)
- Кнопка "Развернуть/Свернуть" для полного просмотра
- Сохранение положения скролла при смене ответа/проверке

4. KaTeX упал с ошибкой

- UI поведение:
- Fallback на текстовое представление формулы (например, \sqrt{x})
- Индикатор ошибки: [Ошибка отображения формулы: \sqrt{x}]
- Кнопка "Повторить рендеринг" рядом с ошибкой
- Логирование ошибки в консоль и отправка в мониторинг
- Для пользователя: сообщение "Некоторые формулы могут отображаться некорректно"

5. Пользователь меняет ответ после check

- UI поведение:
- Запрещено по умолчанию — варианты заблокированы после проверки
- Альтернативный вариант (если разрешено):
- Кнопка "Изменить ответ" появляется после проверки
- При клике: сброс isChecked и showExplanation
- Обновление глобального хранилища: пометка как изменённый ответ
- В статистике учитывается последний выбранный вариант

6. Пользователь в demo режиме

- UI поведение:
- Explanation: скрыт полностью или заблюрен с оверлеем
- Текст поверх blurred explanation:
