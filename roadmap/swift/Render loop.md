Render loop — это цикл отрисовки в системе iOS.

![image](https://user-images.githubusercontent.com/47610132/185236239-fbf5b0a5-32d3-4856-9b98-10cead07ead4.png)

Жизненный цикл у него такой:

- Получаем событие
- Создаем render tree
- Отправляем на Render Server
- Меняем кадр

![image](https://user-images.githubusercontent.com/47610132/185236334-5212e180-29aa-495a-8142-f77d6a578532.png)

**Event**

Сначала происходит какое-то событие (Тач, колбэк с сети, действие с клавиатуры, таймеры)

![image](https://user-images.githubusercontent.com/47610132/185236504-dde97566-5e11-469f-8840-2ef251dd6abe.png)

События вызываются из любого места в иерархии.
Допустим мы хотим изменить `bounds` нашей вьюхи, то Core Animation вызывает метод `setNeedsLayout`. 
Система понимает, что нужно вызвать апдейт лайут реквеста.

![image](https://user-images.githubusercontent.com/47610132/185236599-d0ad4c70-9307-4df5-a640-53395e14b29f.png)

**Commit Transaction**

![image](https://user-images.githubusercontent.com/47610132/185237016-0004db78-0f1f-43d5-a973-b49127f4d5e7.png)

В основе всех обновлений слоев и анимаций лежит Core Animation. Этот фреймворк работает не только внутри самого приложения, но и между другими приложениями. 
Когда мы переключаемся из приложения в приложение.

Сама анимация происходит в другом этапе, за рамками нашего приложения. Этот этап называется `render server`.

На этапе же commit transaction происходит подготовка layer tree и его неявная транзакция для обновления.

Транзакции — это механизм, который Core Animation использует для обновления свойств. Любые свойства наших слоев не изменяются мгновенно, а вместо этого подготавливаются в транзакцию и ждут своего коммита

Когда мы хотим выполнить анимацию, то сначала проходим 4 этапа:

- **Layout —** На этом этапе мы подготавливаем вьюхи, их свойства (frame, background color, border и другие)Как только лайаут расчитался, система вызывает метод setNeedsDisplay.
- **Display —** обновляет CGContext. Это рисование может включать вызов функций drawRect, drawLayer каждых subviews
- **Prepare —** На этом этапе Core Animation готовится отправить данные анимации на render server. Здесь происходит подготовка и декодинг картинок
- **Commit —** Это заключительный этап, когда Core Animation упаковывает слои и свойства анимации и отправляет их через [Interprocess communication](https://medium.com/@ali.pourhadi/ipc-mach-message-cab64ff1b569) (IPC) на render server для отображения.

Core Animation объединяет изменения в транзакцию, кодирует их и фиксирует на render server.

Окей, мы подготовили наш render tree и отдали их следующему этапу.

### **Render Server**

Теперь мы на рендер сервере — это отдельный процесс, который вызывает методы отрисовки для GPU с использованием OpenGL или Metal. Он отвечает за рендер наших слоев в изображение

**Render Prepare**

На этом этапе происходит пробег про layer tree и мы подготавливаем layer pipeline для выполнения его на GPU. Рекурсивно пробегаясь от родительского слоя к дочернему.

![image](https://user-images.githubusercontent.com/47610132/185237300-6fb11374-cfed-4657-aa2d-b02e44cf070c.png)

**Render Execute**

После layers pipeline передан на отрисовку. Где каждый слой будет собран в финальную текстуру. До этого момента все вычисления происходили в CPU, но дальше работа перешла в руки GPU.

![image](https://user-images.githubusercontent.com/47610132/185237394-7ed64bb3-e9b1-4eae-91e3-9a589e0ee3ab.png)

Некоторые слои будут отображаться дольше, чем обычные. И это чаще всего то самое бутылочное горлышко для оптимизаций.
Как только GPU выполняет отрисовку изображений, то это готово к отображению для следующего VSYNC
VSYNC — это дедлайн для каждой фазы нашего рендер лупа. Каждый VSYNC — меняет нам следующий кадр
Для достижения лучшей оптимизации каждый фрейм распараллеривается. Пока CPU читает кадр номер N, в это время GPU рендерит предыдущий кадр N-1

![image](https://user-images.githubusercontent.com/47610132/185237470-a70b3d5a-08fe-4e05-bce9-cf459455dac4.png)

Мы определили что такое render loop. Теперь определим что же влияет на лаги и просадки кадров.

## **Проблемы с производительностью**

Если мы перегрузим наше приложение или будем плохо следить за ресурсами, то можем столкнуться с такими проблемами:

- Потеря кадров
- Быстрая трата батареи
- Долгая отзывчивость

Поэтому стоит ознакомиться с советами, которые помогут решить проблему.

Как мы помним для render loop’a у нас есть операции на CPU и на GPU.

Мы уже знаем, что здесь работа устроена так, что CPU и GPU работают параллельно друг с другом. Пока CPU читает кадр номер N, 
в это время GPU рендерит предыдущий кадр N-1, и так далее

![image](https://user-images.githubusercontent.com/47610132/185237633-92f2175e-44aa-4dc7-85e3-8d5b7e889b80.png)

Перейдем к основным проблемам и узким горлышкам, которые могут повлиять на производительность
На main thread выполняется код, который отвечает за ивенты типа касания и работу с UI. Он же рендерит экран. В большинстве современных смартфонов рендеринг происходит с частотой 60 кадров в секунду. Это значит, что задачи должны выполняться за 16,67 миллисекунд (1000 миллисекунд/ 60 кадров). Поэтому ускорение работы в Main Thread — важно.
Если какая-то операция занимает больше 16,67 миллисекунд, автоматически происходит потеря кадров, и пользователи приложения заметят это при воспроизведении анимаций. На некоторых устройствах рендеринг происходит ещё быстрее, например, на iPad Pro 2017 частота обновления экрана составляет 120 Гц, поэтому на выполнение операций за один кадр есть всего 8 миллисекунд.

## **Offscreen Rendering**

Что же такое offscreen rendering? По своей сути — это какие-то внеэкранный расчеты.

Под капотом это выглядит следующим образом: во время прорисовки слоя, которому необходима **внеэкранные расчеты**, GPU останавливает процесс визуализации и передает управление CPU. В свою очередь, CPU выполняет все необходимые операции (например, cоздает тень) и возвращает управление GPU с уже прорисованным слоем. GPU визуализирует его и процесс прорисовки продолжается.

Кроме того, offscreen rendering требует выделения дополнительной памяти, для так называемого резервного хранилища. В то же время, она не нужна для прорисовки слоев, где используется аппаратное ускорение.

Нашему GPU потребуется дополнительная обратка в случае, если мы изменим свойства ниже:

### **Тени**

Тут все просто. Рендеру не хватает информации для отрисовки тени, поэтому тут расчет тени происходит отдельно. Будет добавлен дополнительный слой. Этот слой был нарисован первым

![image](https://user-images.githubusercontent.com/47610132/185237716-99f88d81-c93c-4574-abf7-73f303478d94.png)

### **Маски для CALayer**

Рендерер должен отобразить поддерево слоев под маской. Но также нужно избежать перезаписи пикселей за пределами маски.

Поэтому мы будем хранить всю информацию об изображение, пока пиксели под маской не будут рассчитаны и не помещены в финальную текстуру.

Эти закадровые вычисления могут хранить множество пикселей, которые юзер никогда не увидит

![image](https://user-images.githubusercontent.com/47610132/185237782-99efab57-6f01-401d-a5c0-a59685c3cd3b.png)

### **Радиус закругления углов**

Этот тип связан с маской. Закругление углов у слоя также **может** рассчитываются заэкранно.

Если рендерингу не хватает инфы, то он может отрисовать вью полностью

![image](https://user-images.githubusercontent.com/47610132/185237844-934dabd4-7813-4447-b5ec-1f415217f405.png)

Кто-то пишет не использовать параметр `cornerRadius`
. Применение `viewLayer.cornerRadius`
 может привести к offscreen rendering. Вместо этого можно использовать класс `UIBezierPath`

### **Визуальные эффекты**

Эта работа связана с двумя эффектами:

- Яркостью
- Блюр

![image](https://user-images.githubusercontent.com/47610132/185237907-f91a1467-082e-40d3-ae99-c01305d82525.png)

Чтобы применить эффект рендер должен скопировать контент под визуальным эффектом в другую текстуру, что хранится в закадровом буфере. Затем применить визуальный эффект к результату и скопировать обратно в рендер буфер
Эти 4 типа закадровых расчетов сильно замедляют рендер.

**Непрозрачность vs Прозрачность**

**UIView.opaque** — это подсказка для визуализатора, что позволяет рассмотреть изображения в качестве полностью непрозрачной поверхности, 
тем самым улучшая качество отрисовки. Непрозрачность означает: "*Ничего не рисуй под поверхностью*». 
UIView.opaque позволяет пропускать отрисовку нижних слоев изображения и тем самым смешивание цветов не происходит. 
Будет использоваться самый верхний цвет для вью.