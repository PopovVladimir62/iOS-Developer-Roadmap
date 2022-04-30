# Жизненный цикл прилoжения.

1. `Not running` - (не запущенное) — приложение не было запущено или его работа была прекращена.
2. `Inactive` - (неактивное) — приложение работает, но не принимает события (например, когда пользователь заблокировал телефон при запущенном приложении).
3. `Active` - (активное) — нормальное состояние приложения при его работе.
4. `Background` - (фоновое) — приложение больше не на дисплее, но оно все еще выполняет код.
5. `Suspended` - (приостановленное) — приложение занимает память, но не выполняет код.

<p align="center" width="100%">
    <img src="https://user-images.githubusercontent.com/47610132/166124137-f8ba5d13-e1ed-460b-97d9-0eff6b6fec5c.png">
</p>

<p align="center" width="100%">
    <img src="https://user-images.githubusercontent.com/47610132/166124170-75a9ce1d-a6a3-466b-8c12-31abb25691ad.png">
</p>

# Жизненный цикл UIViewController

1. `load view` — создает вью, которой управляет контроллер. Вызывается при создании контроллера. Вы можете переопределить этот метод, чтобы создать свои вью вручную, здесь присваивается корневое view для иерархии представлений.
2. `viewDidLoad` — вью создано и загружено в память, но нет bounds. Хорошее место для инициализации и настройки объектов, используемых во вью контроллере.
3. `viewWillAppear` — вью будет добавлено в иерархию, определены bounds, но ориентация экрана не определена. Вызывается каждый раз, когда появляется вью.
4. `viewWillLayoutSubviews` — вызывается каждый раз, когда frame изменился, например, при смене ориентации. Если вы не используете autoresizing masks или constaints, вы, вероятно, хотите обновить сабвью здесь.
5. `viewDidLayoutSubviews` — вызывается уведомить контроллер, что его вью только что залэйаутил сабвью.
6. `viewDidAppear` — вью добавлено в иерахию и появилось на экране. Хорошее место для выполнения задач, связанных с анимацией вью. Метод вызывается после того, как анимация загрузки вью закончена. 
Иногда хорошим кейсом в этом методе будет вытаскивать данные из кордаты и отображать на вью или запрашивать данные с сервера.
Так же, в этом жизненном цикле, определены bounds.
7. `viewWillDissapear` — вью уходит с экрана. Вызывается как при закрытии вью контроллера, так и при переходе дальше по иерархии, например, при пуше нового контроллера в NavigationController.
8. `viewDidDissapear` — вью ушло с экрана. Вызывается как при закрытии вью контроллера, так и при переходе дальше по иерархии.

<p align="center" width="100%">
    <img src="https://user-images.githubusercontent.com/47610132/166124339-1ef6b6ce-0148-4316-baa0-944e1693dfdd.png">
</p>