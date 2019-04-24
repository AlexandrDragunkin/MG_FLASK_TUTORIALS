# *OAuth аутентификация в приложении Flask* #
Эта статья является бонусом к новому циклу статей Flask Mega-Tutorial (2018).
Автор тот же Мигель Гринберг. Статья не новая, но не утратила своей актуальности. 

[blog.miguelgrinberg.com](http://blog.miguelgrinberg.com "blog.miguelgrinberg.com")

Технологии OAuth уже больше 10 лет, и 99% процентов интернет-аудитории имеет аккаунт минимум на одном из ресурсов, поддерживающих OAuth. Кнопка «Войти через» есть почти на каждом ресурсе? Разберемся как это делается с применением микрофреймворка Flask.



<center>![](https://habrastorage.org/webt/kr/s9/vf/krs9vfybluzgbsolq4vy091xgf0.png)</center>

Многие веб-сайты предоставляют пользователям возможность упрощенной регистрации в "один клик", используя стороннюю службу аутентификации, с использованием учетной записи пользователя, в каком либо из известных социальных сервисов. В моем старом курсе [Flask Mega-Tutorial](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-v-user-logins-legacy) я показал вам, как использовать один из этих протоколов, называемый *OpenID*, ( который ныне почил с миром *прим. переводчика*).

<center>![](https://habrastorage.org/webt/yf/rp/02/yfrp028vv-mcsmzwftlqz0cvjje.png)</center>

В этой статье я хочу дать вам введение в протокол [OAuth](http://oauth.net/), который в наши дни заменил *OpenID* как предпочтительный сторонний механизм аутентификации. Я также покажу вам полное приложение *Flask*, которое реализует функции *"Sign In with Facebook"* и *"Sign In with Twitter"*. С этими двумя реализациями в качестве примера вам будет легко добавить любые другие провайдеры *OAuth*, которые могут вам понадобиться.




> *Прим. переводчика:* Еще пара ссылок о том как устроен OAuth [здесь](https://habrahabr.ru/post/145988/), [здесь](https://habrahabr.ru/company/mailru/blog/115163/)

## Краткое введение в OAuth ##
Лучший способ представить *OAuth* - перечислить список событий, которые происходят во время входа:

1. **Обработка отправки формы.** Пользователь переходит на домашнюю страницу приложения, например http://*www.example.com*, и нажимает кнопку *"Sign In with Facebook"*, которая ссылается на маршрут приложения, например  *http://www.example.com/authorize/facebook*.
- **Fetch Request Token (внутренний запрос).** Сервер получает запрос и отвечает перенаправлением на URL авторизации *OAuth Facebook*. Все провайдеры OAuth должны документировать URL-адрес для перенаправления пользователя.
 - В запросе передается *Consumer key* — «логин приложения», а сам запрос подписывается при помощи *Consumer secret* — «пароля приложения», что защищает его от подделки.
 - В ответ Provider генерирует и возвращает «заполненный мусором» токен, который называется Request Token. 
- **Redirect to Authorization (через редирект в браузере).** Теперь пользователю предлагается войти в систему Facebook (если уже не вошел в систему). Затем предоставляется запрос об обмене информацией, когда пользователю необходимо предоставить разрешение Facebook для совместного использования запрошенной информации с исходным приложением. Все это делается на веб-сайте Facebook и является частной транзакцией между Facebook и пользователем, приложение не участвует.
- **Fetch Access Token (внутренний запрос).** После того, как пользователь принимает запрос на обмен информацией, Facebook перенаправляет обратно в приложение по предварительно настроенному URL-адресу обратного вызова, например *http://www.example.com/callback/facebook*. Строка запроса URL-адреса переадресации включает в себя код авторизации, который приложение может использовать для доступа к API Facebook от имени пользователя.
- **Call API (внутренний запрос).** Приложение использует API Facebook для получения информации о пользователе. Особый интерес представляет собой уникальный идентификатор пользователя ( *Shared Secret* ), который может использоваться для регистрации пользователя в базе данных приложения после регистрации пользователя для входа в систему.

Обмен между приложением и сторонней службой не является тривиальным, но для пользователя это чрезвычайно просто, поскольку все, что нужно пользователю, - это войти на сайт третьей стороны и дать разрешение на обмен информацией с помощью приложения.

В настоящее время используются две версии протокола OAuth, как в соответствии с описанным выше общим процессом, так и с некоторыми различиями в реализации. *OAuth 1.0a*, используемый Twitter, является самым сложным из двух. *OAuth 2*, используемый Facebook, представляет собой несовместимую пересмотренную версию протокола, которая устраняет большую часть сложности версии 1.0a, полагаясь на защищенный HTTP для шифрования.

## Регистрация у провайдеров OAuth ##

Прежде чем приложение сможет использовать стороннего поставщика OAuth, его необходимо зарегистрировать. Для Facebook и Twitter это делается на своих соответствующих сайтах разработчиков с созданием "app", которое представляет приложение для пользователей этих сайтов.

Создать приложение для Facebook вы можете здесь [https://developer.facebook.com](https://developer.facebook.com). 

Жмем "НАЧАТЬ" и NEXT. По ходу заполняем различную информацию о себе.

![](https://habrastorage.org/webt/vd/kk/ry/vdkkrytlp3lu11zodu7buxs9y3q.png)
![](https://habrastorage.org/webt/q5/or/ed/q5oredyovzvbdl5ogjotzavjxy0.png)

Выбираем "Вход через Facebook"

![](https://habrastorage.org/webt/6z/ku/tg/6zkutgt_jszixhm6cu33s91jki8.png)

Из списка возможных приложнеий выберите тип "WWW/Website". 

![](https://habrastorage.org/webt/w6/-k/8n/w6-k8nnqtpdqte-xs9jp835e8fu.png)

Укажите URL-адрес приложения, который в случае его запуска на вашем компьютере будет `http://localhost:5000`.

![](https://habrastorage.org/webt/el/5i/ai/el5iainvbxe1bs6-fhqqer4u_gm.png)


## Пример аутентификации OAuth ##
В следующих разделах я собираюсь описать относительно простое приложение Flask, которое реализует аутентификацию Facebook и Twitter.

Я покажу вам важные части приложения в статье, но полное приложение доступно в этом репозитории **GitHub**: [https://github.com/miguelgrinberg/flask-oauth-example](https://github.com/miguelgrinberg/flask-oauth-example). В конце этой статьи я покажу вам инструкции по ее запуску.

## User Model ##
Пользователи в примере приложения хранятся в базе данных SQLAlchemy. Приложение использует расширение [Flask-SQLAlchemy](https://pythonhosted.org/Flask-SQLAlchemy/) для работы с базой данных и расширение [Flask-Login](https://flask-login.readthedocs.org/en/latest/) для отслеживания зарегистрированных пользователей.
	
	from flask.ext.sqlalchemy import SQLAlchemy
	from flask.ext.login import LoginManager, UserMixin
	
	db = SQLAlchemy(app)
	lm = LoginManager(app)
	
	class User(UserMixin, db.Model):
	    __tablename__ = 'users'
	    id = db.Column(db.Integer, primary_key=True)
	    social_id = db.Column(db.String(64), nullable=False, unique=True)
	    nickname = db.Column(db.String(64), nullable=False)
	    email = db.Column(db.String(64), nullable=True)
	
	@lm.user_loader
	def load_user(id):
	    return User.query.get(int(id))

В базе данных есть отдельная таблица для пользователей *users*, которая в дополнение к `id`, который является основным ключом, содержит три столбца:

- `social_id`: строка, которая определяет уникальный идентификатор из сторонней службы аутентификации, используемой для входа в систему.


- `nickname`: псевдоним для пользователя. Должен быть определен для всех пользователей и не обязательно должен быть уникальным.


- `email`: адрес электронной почты пользователя. Этот столбец не является обязательным.

Класс User  наследуется от `UserMixin` из Flask-Login, который  дает ему методы, требуемые этим расширением. Функция обратного вызова `user_loader`, также требуемая Flask-Login, загружает пользователя по его первичному ключу.

## Реализация OAuth ##
Для Python существует несколько клиентских пакетов OAuth. В этом примере я решил использовать [Rauth](https://rauth.readthedocs.io/en/latest/). Однако, даже при использовании пакета OAuth существует множество аспектов проверки подлинности провайдерами, что затрудняет задачу.

Прежде всего, существуют две версии протокола OAuth, которые широко используются. Но даже среди провайдеров, использующих одну и ту же версию OAuth, есть много деталей, которые не являются частью спецификации и должны выполняться в соответствии с собственной документацией.

По этой причине я решил реализовать слой абстракции поверх Rauth, так что приложение Flask может быть написано в общем виде. Ниже представлен простой базовый класс, в котором будут написаны конкретные реализации провайдера:

	
	class OAuthSignIn(object):
	    providers = None
	
	    def __init__(self, provider_name):
	        self.provider_name = provider_name
	        credentials = current_app.config['OAUTH_CREDENTIALS'][provider_name]
	        self.consumer_id = credentials['id']
	        self.consumer_secret = credentials['secret']
	
	    def authorize(self):
	        pass
	
	    def callback(self):
	        pass
	
	    def get_callback_url(self):
	        return url_for('oauth_callback', provider=self.provider_name,
	                       _external=True)
	
	    @classmethod
	    def get_provider(self, provider_name):
	        if self.providers is None:
	            self.providers = {}
	            for provider_class in self.__subclasses__():
	                provider = provider_class()
	                self.providers[provider.provider_name] = provider
	        return self.providers[provider_name]
	
	class FacebookSignIn(OAuthSignIn):
	    pass
	
	class TwitterSignIn(OAuthSignIn):
    pass


Базовый класс `OAuthSignIn` определяет структуру, которой должны следовать подклассы, реализующие услуги каждого провайдера. Конструктор инициализирует имя провайдера, а также идентификатор приложения и секретный код, назначенные им, и полученные из конфигурации. Ниже приведен пример конфигурации приложения (конечно, вам нужно будет заменить эти коды своим собственным):
	
	app.config['OAUTH_CREDENTIALS'] = {
	    'facebook': {
	        'id': '470154729788964',
	        'secret': '010cc08bd4f51e34f3f3e684fbdea8a7'
	    },
	    'twitter': {
	        'id': '3RzWQclolxWZIMq5LJqzRZPTl',
	        'secret': 'm9TEd58DSEtRrZHpz2EjrV9AhsBRxKMo8m3kuIZj3zLwzwIimt'
	    }
	}

На верхнем уровне есть два важных события, поддерживаемых этим классом, которые являются общими для всех провайдеров OAuth:

- Инициирование процесса аутентификации. Для этого приложение должно перенаправить на веб-сайт провайдера, чтобы позволить пользователю аутентифицироваться там. Это представлено методом` authorize()`.
- Как только аутентификация завершена, провайдер перенаправляет обратно приложение. Это обрабатывается методом `callback()`. Поскольку у провайдера нет прямого доступа к внутренним методам приложения, он будет перенаправляться на URL-адрес, который его вызовет. URL-адрес, который поставщик должен перенаправить, возвращается методом `get_callback_url()` и создается с использованием имени провайдера, так что каждый провайдер`` получает свой выделенный маршрут.

Метод `get_provider()` используется для поиска правильного экземпляра `OAuthSignIn` с именем поставщика. Этот метод использует интроспекцию для поиска всех подклассов `OAuthSignIn`, а затем сохраняет экземпляр каждого в словаре.

## Аутентификация OAuth с помощью Rauth ##
Rauth представляет провайдеры OAuth с объектом класса `OAuth1Service` или `OAuth2Service`, в зависимости от версии используемого протокола. Я создаю объект этого класса в подклассе `OAuthSignIn` каждого провайдера. Реализации для Facebook и Twitter показаны ниже:
	
	class FacebookSignIn(OAuthSignIn):
	    def __init__(self):
	        super(FacebookSignIn, self).__init__('facebook')
	        self.service = OAuth2Service(
	            name='facebook',
	            client_id=self.consumer_id,
	            client_secret=self.consumer_secret,
	            authorize_url='https://graph.facebook.com/oauth/authorize',           
	            access_token_url='https://graph.facebook.com/oauth/access_token',
	            base_url='https://graph.facebook.com/'
	        )
	
	class TwitterSignIn(OAuthSignIn):
	    def __init__(self):
	        super(TwitterSignIn, self).__init__('twitter')
	        self.service = OAuth1Service(
	            name='twitter',
	            consumer_key=self.consumer_id,
	            consumer_secret=self.consumer_secret,
	            request_token_url='https://api.twitter.com/oauth/request_token',
	            authorize_url='https://api.twitter.com/oauth/authorize',
	            access_token_url='https://api.twitter.com/oauth/access_token',
	            base_url='https://api.twitter.com/1.1/'
	        )

Для Facebook, который реализует OAuth 2, используется класс `OAuth2Service`. Объект службы инициализируется именем службы и несколькими аргументами, специфичными для OAuth. Аргументами `client_id` и `client_secret` являются те, которые назначены приложению на сайте разработчика Facebook. `Authorize_url` и `access_token_url` - это URL-адреса, определенные Facebook для приложений, к которым необходимо подключиться во время процесса аутентификации. Наконец, `base_url` задает URL-адрес префикса для любых вызовов API Facebook после завершения проверки подлинности.

Twitter реализует OAuth 1.0a, поэтому используется класс `OAuth1Service`. В OAuth 1.0a идентификационные и секретные коды называются `consumer_key` и `consumer_secret`, но в остальном идентичны по функциональности аналогам OAuth 2. Протокол OAuth 1 требует, чтобы провайдеры отображали три URL вместо двух, есть дополнительный запрос `request_token_url`. Аргументы `name` и `base_url` идентичны аргументам, используемым в службах OAuth 2. Следует отметить, что Twitter предлагает два параметра для параметра `authorize_url`. URL, показанный выше, `https://api.twitter.com/oauth/authorize`, является самым безопасным, так как он представит пользователю окно, в котором нужно будет разрешить приложению получать доступ к Twitter каждый раз. Изменение этого URL-адреса на `https://api.twitter.com/oauth/authenticate` заставит Twitter запрашивать разрешение только в первый раз, а затем тихо разрешит доступ до тех пор, пока пользователь не выйдет из Twitter.

Обратите внимание, что нет стандартизации для точки входа URL-адресов, провайдеры OAuth определяют их по своему усмотрению. Чтобы добавить нового поставщика OAuth, вам нужно будет получить эти URL-адреса из его документации.   

## Фаза авторизации OAuth ##
Когда пользователь нажимает ссылку «Вход в систему с ...» для инициирования аутентификации OAuth, вызов следует по такому маршруту:
	
	@app.route('/authorize/<provider>')
	def oauth_authorize(provider):
	    if not current_user.is_anonymous():
	        return redirect(url_for('index'))
	    oauth = OAuthSignIn.get_provider(provider)
	    return oauth.authorize()

Этот маршрут сначала гарантирует, что пользователь не вошел в систему, а затем просто получает подкласс `OAuthSignIn`, соответствующий данному поставщику, и вызывает его метод `authorize()`, чтобы инициировать процесс. Ниже показана реализация `authorize()` для Facebook и Twitter:
	
	class FacebookSignIn(OAuthSignIn):
	    # ...
	    def authorize(self):
	        return redirect(self.service.get_authorize_url(
	            scope='email',
	            response_type='code',
	            redirect_uri=self.get_callback_url())
	        )
	
	class TwitterSignIn(OAuthSignIn):
	    # ...
	    def authorize(self):
	        request_token = self.service.get_request_token(
	            params={'oauth_callback': self.get_callback_url()}
	        )
	        session['request_token'] = request_token
	        return redirect(self.service.get_authorize_url(request_token[0]))

Для провайдеров OAuth 2, таких как Facebook, реализация просто выдает перенаправление на URL-адрес, созданный объектом службы `rauth`. Область действия зависит от Поставщика, в этом конкретном случае я прошу, чтобы Facebook предоставил электронную почту пользователя. В response_type='code' аргумент говорит oauth-провайдеру, что приложение представляет собой веб-приложение (есть и другие возможные значения для различных процессов аутентификации). Наконец, аргумент redirect_uri назначает маршрут приложения, который должен вызвать поставщик после завершения проверки подлинности.

Поставщики OAuth 1.0 a используют несколько более сложный процесс, который включает получение маркера запроса от поставщика, который представляет собой список из двух элементов, первый из которых затем используется в качестве аргумента в перенаправлении. Весь маркер запроса сохраняется в пользовательском сеансе, так как он будет снова необходим в обратном вызове.

## Фаза обратного вызова Callback OAuth ##
Поставщик OAuth перенаправляет обратно в приложение после аутентификации пользователя и дает разрешение на обмен информацией. Маршрут, который обрабатывает этот обратный вызов, показан ниже:
	
	@app.route('/callback/<provider>')
	def oauth_callback(provider):
	    if not current_user.is_anonymous():
	        return redirect(url_for('index'))
	    oauth = OAuthSignIn.get_provider(provider)
	    social_id, username, email = oauth.callback()
	    if social_id is None:
	        flash('Authentication failed.')
	        return redirect(url_for('index'))
	    user = User.query.filter_by(social_id=social_id).first()
	    if not user:
	        user = User(social_id=social_id, nickname=username, email=email)
	        db.session.add(user)
	        db.session.commit()
	    login_user(user, True)
	    return redirect(url_for('index'))

Этот маршрут создает экземпляр класса поставщика `OAuthSignIn` и вызывает его метод `callback()`. Этот метод имеет функцию завершения аутентификации с провайдером и получения информации о пользователе. Возвращаемое значение представляет собой кортеж с тремя значениями, уникальный идентификатор (называемый `social_id`, чтобы отличать его от первичного ключа `id`), псевдоним пользователя и адрес электронной почты пользователя. Идентификатор и псевдоним являются обязательными, но в этом примере приложения я сделал электронное письмо необязательным, так как Twitter никогда не делится этой информацией с приложениями.

Пользователь просматривается в базе данных по полю `social_id`, и если не найден, новый пользователь добавляется в базу данных с информацией, полученной от провайдера, эффективно регистрируя новых пользователей автоматически. Затем пользователь регистрируется с помощью функции `login_user()` в Flask-Login и, наконец, перенаправляется на домашнюю страницу.

Реализация метода `callback()` для провайдеров OAuth для Facebook и Twitter показана ниже:
	
	class FacebookSignIn(OAuthSignIn):
	    # ...
	    def callback(self):
	        def decode_json(payload):
	            return json.loads(payload.decode('utf-8'))
	
	        if 'code' not in request.args:
	            return None, None, None
	        oauth_session = self.service.get_auth_session(
	            data={'code': request.args['code'],
	                  'grant_type': 'authorization_code',
	                  'redirect_uri': self.get_callback_url()},
	            decoder=decode_json
	        )
	        me = oauth_session.get('me').json()
	        return (
	            'facebook$' + me['id'],
	            me.get('email').split('@')[0],  # Facebook does not provide
	                                            # username, so the email's user
	                                            # is used instead
	            me.get('email')
	        )
	
	class TwitterSignIn(OAuthSignIn):
	    # ...
	    def callback(self):
	        request_token = session.pop('request_token')
	        if 'oauth_verifier' not in request.args:
	            return None, None, None
	        oauth_session = self.service.get_auth_session(
	            request_token[0],
	            request_token[1],
	            data={'oauth_verifier': request.args['oauth_verifier']}
	        )
	        me = oauth_session.get('account/verify_credentials.json').json()
	        social_id = 'twitter$' + str(me.get('id'))
	        username = me.get('screen_name')
	        return social_id, username, None   # Twitter does not provide email

В методе `callback()` передается токен проверки, который приложение может использовать для связи с API-интерфейсом провайдера. В случае OAuth 2 это происходит как аргумент `code` , тогда как для OAuth 1.0a это `oauth_verifier`, оба заданные в строке запроса.
Этот код используется для получения `oauth_session` из объекта службы  *rauth*.

Обратите внимание, что в последних версиях API Facebook токен сеанса возвращается в формате JSON. Формат по умолчанию, ожидаемый rauth для этого токена, должен указываться в строке запроса. По этой причине, необходимо добавить аргумент `decoder` , который декодирует содержимое JSON. В Python 2 достаточно передать `json.loads`, но в Python 3 нам нужен дополнительный шаг, потому что полезная нагрузка возвращается как *байты*, которые json-парсер не понимает. Преобразование из байтов в строку выполняется во внутренней функции decode_json.

Объект `oauth_session` может использоваться для предоставления API-запросов поставщику. Здесь он используется для запроса информации о пользователе, которая должна предоставляться определенным провайдером. Facebook предоставляет идентификатор пользователя и адрес электронной почты, но не дает имен пользователей, поэтому имя пользователя для приложения создается из левой части адреса электронной почты. Twitter предоставляет идентификатор и имя пользователя, но не поддерживает электронную почту, поэтому электронное письмо возвращается как `None`. 

Данные, полученные от провайдера, окончательно возвращаются в виде трехэлементного кортежа для функции просмотра. Обратите внимание, что в обоих случаях значение `id` от провайдера добавляется с `«facebook $»` или `«twitter $»` до его возврата, чтобы сделать его уникальным для всех поставщиков. Поскольку это то, что приложение будет хранить как `social_id` в базе данных, необходимо сделать это, чтобы два поставщика, которые назначили один и тот же `id` двум различным пользователям, не конфликтовали в базе данных приложения.

## Вывод ##
Как я уже упоминал выше, пример приложения позволяет любому пользователю регистрироваться и входить в систему с помощью учетной записи Facebook или Twitter. Приложение демонстрирует, как регистрировать пользователей без необходимости вводить какую-либо информацию, все, что им нужно сделать, это войти в систему с провайдером и разрешить совместное использование информации.
Если вы хотите попробовать этот пример, вам необходимо выполнить некоторые подготовительные шаги:

Клонировать или загрузить репозиторий проекта: [https://github.com/miguelgrinberg/flask-oauth-example](https://github.com/miguelgrinberg/flask-oauth-example)
Создать виртуальную среду и установите пакеты по списку в файле *requirements.txt* (вы можете использовать Python 2.7 или 3.4).
Зарегистрируйте «приложение» с помощью Facebook и Twitter,
как описано выше.
Отредактируйте *app.py* с идентификатором и секретными кодами ваших приложений Facebook и Twitter.
После выполнения этих инструкций вы можете запустить приложение с помощью *python app.py*, а затем запустить http://localhost: 5000 в своем браузере.

Надеюсь, что эта статья полезна в демистификации OAuth.
Если у вас есть вопросы, напишите их ниже.

Miguel

