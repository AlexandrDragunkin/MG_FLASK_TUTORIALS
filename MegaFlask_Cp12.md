# Мега-Учебник Flask, Часть XII: Даты и время (издание 2018) #

### *Miguel Grinberg* ###

----

![](https://habrastorage.org/webt/jl/jn/bb/jljnbbjr-ejh473xy_eccsmknpk.png) [Туда](https://habrahabr.ru/post/349060/)  [Сюда ](https://habrahabr.ru/post/349604/) ![](https://habrastorage.org/webt/rw/dy/-g/rwdy-grsvbpcetjttrmecdkxtlk.png)



Это двенадцатая часть серии Мега-Учебник Flask, в которой я расскажу вам, как работать с датой и временем таким образом, что бы пользователи, не зависели от того, в каком часовом поясе они находятся.

Для справки ниже приведен список статей этой серии.

<cut />

<spoiler title="Оглавление">

- [**Глава 1: Привет, мир!**](https://habrahabr.ru/post/346306/)
- [**Глава 2: Шаблоны**](https://habrahabr.ru/post/346340/)
- [**Глава 3: Веб-формы**](https://habrahabr.ru/post/346342/)
- [**Глава 4: База данных**](https://habrahabr.ru/post/346344/)
- [**Глава 5: Пользовательские логины**](https://habrahabr.ru/post/346346/)
- [**Глава 6: Страница профиля и аватары**](https://habrahabr.ru/post/346348/)
- [**Глава 7: Обработка ошибок**](https://habrahabr.ru/post/346880/)
- [**Глава 8: Подписчики, контакты и друзья**](https://habrahabr.ru/post/347450/)
- [**Глава 9: Разбивка на страницы**](https://habrahabr.ru/post/347926/)
- [**Глава 10: Поддержка электронной почты**](https://habrahabr.ru/post/348566/)
- [**Глава 11: Реконструкция**](https://habrahabr.ru/post/349060/)
- [**Глава 12: Дата и время**](https://habrahabr.ru/post/349604/)(Эта статья)
- Глава 13: I18n и L10n (доступно 27 февраля 2018 года)
- Глава 14: Ajax (доступно 6 марта 2018 года)
- Глава 15: Улучшение структуры приложения (доступно 13 марта 2018 года)
- Глава 16: Полнотекстовый поиск (доступен 20 марта 2018 года)
- Глава 17: Развертывание в Linux (доступно 27 марта 2018 года)
- Глава 18: Развертывание на Heroku (доступно 3 апреля 2018 года)
- Глава 19: Развертывание на Docker контейнерах (доступно 10 апреля 2018 года)
- Глава 20: Магия JavaScript (доступна 17 апреля 2018 года)
- Глава 21: Уведомления пользователей (доступно 24 апреля 2018 года)
- Глава 22: Справочные задания (доступны 1 мая 2018 года)
- Глава 23: Интерфейсы прикладного программирования (API) (доступно 8 мая 2018 г.)

</spoiler>
*Примечание 1: Если вы ищете старые версии данного курса, это [здесь](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-i-hello-world-legacy "здесь").*

*Примечание 2: Если вдруг Вы захотели бы выступить в поддержку моей(Мигеля) работы, или просто не имеете терпения дожидаться статьи неделю, я (Мигель Гринберг)предлагаю полную версию данного руководства(на английском языке) в виде электронной книги или видео. Для получения более подробной информации посетите [learn.miguelgrinberg.com](http://learn.miguelgrinberg.com "learn.miguelgrinberg.com").*

Один из аспектов моего приложения Microblog, который я игнорировал в течение длительного времени -- отображение даты и времени. До сих пор я просто позволял Python отобразить объект `datetime` в модели `User` и полностью проигнорировал это в модели `Post`.

*Ссылки GitHub для этой главы:*  [Browse](https://github.com/miguelgrinberg/microblog/tree/v0.12), [Zip](https://github.com/miguelgrinberg/microblog/archive/v0.12.zip), [Diff](https://github.com/miguelgrinberg/microblog/compare/v0.11...v0.12).

## Адский часовой пояс ##
Использование функционала Python на сервере для отображения даты и времени, которые отображены пользователям в их веб-браузерах, не очень хорошая идея. Рассмотрим следующий пример. Я пишу это в 16:06 28 сентября 2017. Мой часовой пояс в то время, когда я пишу это, - это PDT (или UTC-7, если хотите). В интерпретаторе Python я получаю следующее:

    >>> from datetime import datetime
    >>> str(datetime.now())
    '2017-09-28 16:06:30.439388'
    >>> str(datetime.utcnow())
    '2017-09-28 23:06:51.406499'

Вызов `datetime.now()` возвращает правильное время для моего местоположения, а вызов `datetime.utcnow()` возвращает время в часовом поясе UTC. Если бы я мог попросить людей, живущих в разных частях света, запустить этот код в то же время со мной, то функция `datetime.now()` вернула бы разные результаты для каждого человека, но `datetime.utcnow()` всегда будет возвращать одно значение в то же время, независимо от местоположения. Итак, какой вариант, по вашему мнению, лучше использовать в веб-приложении, которое, скорее всего, будет доступно пользователям по всему миру?

Очевидно, что сервер должен управлять временем, которое не зависит от местоположения. Если это приложение будет развиваться и появится необходимость использования нескольких production серверов в разных регионах по всему миру, я бы не хотел, чтобы каждый сервер записывал временные метки в базу данных в соответствии  разными часовыми поясами, потому что это затруднит работу с этими данными. Поскольку UTC является наиболее используемым единообразным часовым поясом и поддерживается в классе `datetime`, это то, что надо! И я буду это использовать.

Но в этом варианте появляется одна важная проблема. Для пользователей в разных часовых поясах будет сложно определить, когда реально была сделана запись, если они видят время в часовом поясе UTC. Им нужно заранее понять, что время отображается в UTC и надо суметь мысленно настроить время на свой часовой пояс. Представьте себе пользователя в часовом поясе PDT, который отправляет что-то в 3:00 вечера, и сразу же видит, что сообщение появляется с временем UTC в 22:00 или, точнее, 22:00. Это будет выглядеть запутанно.

Хотя стандартизация временных меток UTC имеет большой смысл с точки зрения сервера, но создает сложности с удобством использования для пользователей. Цель этой главы - решить эту проблему, сохранив все временные метки, управляемые на сервере в UTC.

## Конверсия часовых поясов ##
Очевидным решением этой проблемы является преобразование всех временных меток из  UTC в локальное время каждого пользователя. Это позволяет серверу продолжать использовать UTC для согласованности, в то время как преобразование «на лету», адаптированное для каждого пользователя, решает проблему удобства. Сложность этого решения заключается в определении местоположение каждого пользователя.

На многих веб-сайтах есть страница конфигурации, где пользователи могут указывать свой часовой пояс. Это потребует от меня добавления новой страницы с формой, в которой я представляю пользователям выпадающий список со списком часовых поясов. Пользователям будет предложено ввести свой часовой пояс, когда они впервые обращаются к сайту, в рамках их регистрации.

Хотя это достойное решение, которое решает проблему, немного странно просить пользователей ввести часть информации, которую они уже настроили в своей операционной системе. Кажется, было бы более эффективно, если бы я мог просто узнать настройку часового пояса у локального компьютера.

Как выясняется, веб-браузер знает часовой пояс пользователя и предоставляет его через стандартные API JavaScript. На самом деле существует два способа использовать информацию о часовом поясе, доступную через JavaScript:

- Подход «старой школы» заключался бы в том, чтобы веб-браузер каким-то образом отправлял информацию о часовом поясе на сервер, когда пользователь впервые регистрируется в приложении. Это можно сделать с помощью вызова [Ajax](http://en.wikipedia.org/wiki/Ajax_(programming)) или, что еще проще, с [тегом meta refresh](http://en.wikipedia.org/wiki/Meta_refresh). Как только сервер определил часовой пояс, он может сохранить его в сеансе  или записать его в запись пользователя в базе данных, а затем настроить все временные метки во время отображения шаблонов.

- Подход «новой школы» состоял бы в том, чтобы не изменить ситуацию на сервере и позволить конвертировать с UTC в локальный часовой пояс на клиенте, используя JavaScript.
Оба варианта рабочие, но второй имеет большое преимущество. Знать часовой пояс не всегда достаточно, чтобы представить дату и время в ожидаемом пользователем формате. Браузер также имеет доступ к конфигурации локали системы, которая определяет такие вещи, как AM/PM и 24-часовой формат, DD/MM/YYYY против MM/DD/YYYY и многие другие культурные или региональные стили.

И если и этого недостаточно, есть ещё одно преимущество в пользу «новой школы». Существует библиотека с открытым исходным кодом, которая делает все это!

## Знакомство с  Moment.js и Flask-Moment ##
[Moment.js](http://momentjs.com/) - небольшая библиотека JavaScript с открытым исходным кодом переводит задачу отображения даты и времени совсем на другой уровень, так как предоставляет все мыслимые варианты форматирования. И некоторое время назад я создал Flask-Moment, небольшое расширение Flask, которое упростило включение Moment.js в ваше приложение.

Итак, начнем с установки Flask-Moment:

	(venv) $ pip install flask-moment

Это расширение добавляется в приложение Flask обычным способом:

> `app/__init__.py`: Создаем экземпляр Flask-Moment.

    # ...
    from flask_moment import Moment
    
    app = Flask(__name__)
    # ...
    moment = Moment(app)

В отличие от других расширений, Flask-Moment работает вместе с *moment.js*, поэтому все шаблоны приложения должны включать эту библиотеку. Чтобы эта библиотека всегда была доступна, я собираюсь добавить ее в базовый шаблон. Это можно сделать двумя способами. Казалось бы, что самый прямой способ - явно добавить тег `<script>`, который импортирует библиотеку. Но Flask-Moment предлагает вариант еще проще, предоставляя функцию `moment.include_moment()`, которая генерирует тег `<script>`:

> app/templates/base.html: Добавляем moment.js в базовый шаблон.

	...
	
	{% block scripts %}
	    {{ super() }}
	    {{ moment.include_moment() }}
	{% endblock %}

Блок `scripts`, который я здесь добавил, представляет собой еще один, экспортируемый базовым шаблоном Flask-Bootstrap. Это место, в которое должны быть включены импорт(ы) JavaScript. Этот блок отличается от предыдущего тем, что он уже поставляется с некоторым содержимым, определенным в базовом шаблоне. Все, что я хочу сделать, это добавить библиотеку moment.js, не теряя базового содержимого. И это достигается с помощью инструкции `super()`, которая сохраняет контент из базового шаблона. Если вы определяете блок в своем шаблоне без использования `super()`, то любой контент, определенный для этого блока в базовом шаблоне, будет потерян.

## Использование Moment.js ##
Moment.js предоставляет браузеру доступный `moment` class. Первым шагом для создания временной метки является создание объекта этого класса, передачей требуемой метки времени в формате [ISO 8601](http://en.wikipedia.org/wiki/ISO_8601). Вот пример:

	t = moment('2017-09-28T21:45:23Z')

Если вы не знакомы со стандартным форматом ISO 8601 для дат и времени, формат выглядит следующим образом:  `{{ year }}-{{ month }}-{{ day }}T{{ hour }}:{{ minute }}:{{ second }}{{ timezone }}`. Я уже решил, что я буду работать только с часовым поясом UTC, поэтому последняя часть всегда будет `Z`, которая представляет собой UTC в стандарте ISO 8601.

Объект `moment` предоставляет несколько методов для разных вариантов отображения. Ниже приведены некоторые из наиболее распространенных вариантов:

	moment('2017-09-28T21:45:23Z').format('L')
	"09/28/2017"
	moment('2017-09-28T21:45:23Z').format('LL')
	"September 28, 2017"
	moment('2017-09-28T21:45:23Z').format('LLL')
	"September 28, 2017 2:45 PM"
	moment('2017-09-28T21:45:23Z').format('LLLL')
	"Thursday, September 28, 2017 2:45 PM"
	moment('2017-09-28T21:45:23Z').format('dddd')
	"Thursday"
	moment('2017-09-28T21:45:23Z').fromNow()
	"7 hours ago"
	moment('2017-09-28T21:45:23Z').calendar()
	"Today at 2:45 PM"

В этом примере создается объект `moment`, инициализированный строкой 28 сентября 2017 года в 21:45 по UTC. Вы можете видеть, что все параметры, которые я пробовал выше, отображаются в UTC-7, который является часовым поясом, настроенным на моем компьютере. Вы можете ввести приведенные выше команды в консоль вашего браузера, убедившись, что страница, на которой вы открываете консоль, содержит moment.js. Вы можете сделать это в микроблоге, если вы внесли изменения выше, включив moment.js, а также на [https://momentjs.com/](https://momentjs.com/).

Обратите внимание, как разные методы создают разные представления. При помощи `format()` вы управляете форматом вывода с помощью строки имени формата, аналогичной функции [strftime](https://docs.python.org/3.6/library/time.html#time.strftime) из Python. Методы `fromNow()` и `calendar()` интересны тем, что они отображают временную метку по отношению к текущему времени, поэтому вы получаете строку вывода, такую как «минута назад» или «через два часа» и т.д.

Если бы вы работали непосредственно в JavaScript, приведенные выше вызовы возвратили строку с отображением временной метки. Значит для добавления этого текста в нужное место на странице, требуется использование JavaScript для работы с [DOM](https://en.wikipedia.org/wiki/Document_Object_Model). Расширение Flask-Moment значительно упрощает использование moment.js, позволяя  вашим шаблонам включающим объект `moment`, включать требуемую магию JavaScript, чтобы отобразить время на странице правильным образом.

Давайте посмотрим на метку времени, которая появляется на странице профиля. Текущий шаблон *user.html* позволяет Python генерировать строковое представление времени. Теперь я могу сделать эту метку времени с помощью Flask-Moment следующим образом:

> app/templates/user.html: Метка времени с использованием moment.js.

                {% if user.last_seen %}
                <p>Last seen on: {{ moment(user.last_seen).format('LLL') }}</p>
                {% endif %}

Как вы можете видеть, Flask-Moment использует синтаксис, похожий на синтаксис библиотеки JavaScript, с одним  маленьким отличием, которым является то, что аргумент `moment()` теперь является объектом Python `datetime`, а не строкой ISO 8601. Данный  вызов moment() в шаблоне также автоматически генерирует необходимый код JavaScript для вставки оказываемых меток в нужное место DOM.

Второе место, где можно воспользоваться Flask-Moment и moment.js находится в *`_post.html`* sub-шаблоне, который вызывается из индексной и пользовательской страниц. В текущей версии шаблона каждому сообщению предшествовала строка "username says:" . Теперь я могу добавить временную метку с помощью `fromnow()`:


> app/templates/_post.html: Отметка времени субшаблоне сообщения.

                <a href="{{ url_for('user', username=post.author.username) }}">
                    {{ post.author.username }}
                </a>
                said {{ moment(post.timestamp).fromNow() }}:
                <br>
                {{ post.body }}

Вот так выглядят обе эти метки времени при визуализации с помощью Flask-Moment и moment.js:

![](https://habrastorage.org/webt/mm/lh/c1/mmlhc17urv9gi3mgxjak0q79m04.png)


![](https://habrastorage.org/webt/jl/jn/bb/jljnbbjr-ejh473xy_eccsmknpk.png) [Туда](https://habrahabr.ru/post/349060/)  [Сюда ](https://habrahabr.ru/post/349604/) ![](https://habrastorage.org/webt/rw/dy/-g/rwdy-grsvbpcetjttrmecdkxtlk.png)