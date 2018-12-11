# Mediascope iOS SDK

Фреймворк MediascopeSDK предназначен для загрузки и измерения видео-рекламы в формате [VAST 2.0/3.0](https://www.iab.com/guidelines/digital-video-ad-serving-template-vast-3-0/).
SDK состоит из двух модулей:

**MSPVideoAdLoader** - загружает данные по указанному адресу и отдает модели креативов в приложение

**MSPVideoAdTracker** - производит измерения и трекинг событий по указанному креативу

1. [Подключение SDK к проекту](#подключение-sdk-к-проекту)
2. [Получение данных через MSPVideoAdLoader](#получение-данных-через-mspvideoadloader)
3. [Трекинг событий через MSPVideoAdTracker](#трекинг-событий-через-mspvideoadtracker)
4. [Включение режима отладки](#включение-режима-отладки)

### Подключение SDK к проекту
Минимальная поддерживаемая версия iOS: 8.0

Подключение с использованием CocoaPods:

Добавьте в Podfile зависимость:
```
pod 'MediascopeSDK'
```
Запустите 'pod install' или 'pod update'.

### Получение данных через MSPVideoAdLoader
Для загрузки данных видеорекламы используется класс MSPVideoAdLoader. Поддерживается загрузка и обработка данных в формате VAST 2.0/3.0.

Для каждой загрузки должен быть создан свой экземпляр MSPVideoAdLoader.

После загрузки данных будет вызван установленный экземпляру MSPVideoAdLoader блок. В случае успешной загрузки данных в него будет передан объект MSPVideoAdContainer, содержащий рекламные креативы. В случае каких-либо ошибок при загрузке или отсутствию данных о рекламе в блок будет передана ошибка, объект MSPVideoAdContainer в этом случае будет равен nil.

#### Инициализация и загрузка данных
Инициализация лоадера происходит через статический конструктор, в качестве параметра для которого указывается url-адрес рекламного сервиса, от которого должны быть получены данные. Далее вызывается метод load, которому нужно передать блок кода, обрабатывающий результат загрузки.
```objective-c
MSPVideoAdLoader *videoAdLoader = [MSPVideoAdLoader loaderForUrl:serverUrl];
[videoAdLoader loadWithCompletionBlock:^(MSPVideoAdContainer *_Nullable videoAdContainer, NSString *_Nullable error)
{
	if (videoAdContainer)
	{
		[self handleData:videoAdContainer];
	}
	else
	{
		NSLog(@"video ad error: %@", error);
	}
}];
```

#### Работа с загруженными рекламными креативами
После успешной загрузки, SDK предоставляет данные о загруженных креативах и компаньонах в виде типизированных объектов, которые должны быть использованы приложением для показа видеорекламы.

Для рекламных креативов SDK берет на себя всю работу по трекингу событий и обработке кликов, приложению для этого делать ничего не требуется. Но это не относится к баннерам-компаньонам - их события и клики должны обрабатываться самим приложением.

##### Объект MSPVideoAdContainer
Объект MSPVideoAdContainer содержит загруженные рекламные креативы и компаньоны и предоставляет соответствующие свойства.
```objective-c
@property(nonatomic, readonly) NSArray<MSPVideoAdCreative *> *creatives;
```
Список загруженных рекламных креативов в виде объектов MSPVideoAdCreative, отсортированных по sequenceId.

```objective-c
@property(nonatomic, readonly) NSArray<MSPVideoAdCompanion *> *companions;
```
Список загруженных компаньонов в виде объектов MSPVideoAdCompanion.

##### Объект MSPVideoAdCreative
Объект MSPVideoAdCreative содержит информацию о загруженном рекламном креативе и его видеофайлах.

Объект MSPVideoAdCreative не содержит события трекинга и клик-ссылку перехода на рекламируемый объект - события и переход по клику обрабатываются SDK, для этого предназначен класс [MSPVideoAdTracker](#трекинг-событий-через-mspvideoadtracker).

Доступны следующие cвойства:
```objective-c
@property(nonatomic, readonly) NSTimeInterval duration;
```
Длительность рекламного креатива в секундах.

```objective-c
@property(nonatomic, readonly) NSTimeInterval skipOffset;
```
Длительность воспроизведения в секундах, после которой рекламный креатив может быть закрыт пользователем.

```objective-c
@property(nonatomic, readonly, nullable) NSString *linkTxt;
```
Текст действия (call-to-action text, "Перейти на сайт", "Скачать приложение" и т.д.).

```objective-c
@property(nonatomic, readonly) BOOL showControls;
```
Показывать ли элементы управления плеером.

```objective-c
@property(nonatomic, readonly) BOOL isClickable;
```
Является ли рекламный креатив кликабельным.

```objective-c
@property(nonatomic, readonly) NSArray<MSPVideoFile *> *videoFiles;
```
Список видеофайлов креатива в виде объектов MSPVideoFile. Каждый объект MSPVideoFile содержит все поля предусмотренные стандартом VAST. Подробное описание доступных полей вы найдете в документации по стандарту [VAST](https://www.iab.com/guidelines/digital-video-ad-serving-template-vast-3-0/).

##### Объект MSPVideoAdCompanion
Объект MSPVideoAdCompanion предназначен для показа баннеров-компаньонов. В отличие от креативов, для компаньонов Mediascope SDK не берет на себя обработку событий и кликов. Работа с баннерами компаньонами должна быть реализована полностью на стороне приложения.

Объект VideoAdCompanion содержит все поля предусмотренные стандартом VAST. Подробное описание доступных полей вы найдете в документации по стандарту [VAST](https://www.iab.com/guidelines/digital-video-ad-serving-template-vast-3-0/).
События трекинга представлены объектами MSPVideoAdEvent и доступны через свойство trackingEvents. Каждый объект содержит два поля:
- type - тип события трекинга, соответствует имени события по стандарту VAST.
- url - url-адрес, который должен быть вызван при наступлении события.

### Трекинг событий через MSPVideoAdTracker
Для трекинга событий и измерения видимости видеоплеера во время воспроизведения рекламного видео используется класс MSPVideoAdTracker.

Для каждого рекламного креатива полученного из MSPVideoAdLoader должен быть создан свой экземпляр MSPVideoAdTracker.

#### Инициализация
Инициализация трекера происходит через статический конструктор, в качестве параметра для которого указывается объект MSPVideoAdCreative, полученный из MSPVideoAdLoader.
```objective-c
MSPVideoAdTracker *_videoAdTracker;
...
...

_videoAdTracker = [MSPVideoAdTracker trackerWithVideoAdCreative:videoAdCreative];
```
Далее в трекере требуется зарегистрировать View видеоплеера. Это необходимо для измерения видимости рекламного видео во время воспроизведения.
```objective-c
[_videoAdTracker registerPlayerView:playerView];
```
Ссылку на экземпляр videoAdTracker необходимо сохранять на протяжении всего воспроизведения рекламного креатива и вызывать в трекере методы соответствующие наступаемым событиям.

#### Методы трекинга событий

```objective-c
- (void)trackPause;
```
Должен быть вызван при постановке рекламного видео на паузу.

```objective-c
- (void)trackResume;
```
Должен быть вызван при возобновлении воспроизведения рекламного видео.

```objective-c
- (void)trackComplete;
```
Должен быть вызван после полного проигрывания рекламного видео.

```objective-c
- (void)trackProgress:(NSTimeInterval)progress duration:(NSTimeInterval)duration;
```
Должен вызываться многократно по мере воспроивзедения рекламного видео не реже, чем раз в секунду.

Параметры:
- position - текущая позиция воспроизведения рекламного видео в секундах.
- duration - общая длительность рекламного видео в секундах.

```objective-c
- (void)trackRewind:(NSTimeInterval)progress;
```
Должен быть вызван при перемотке рекламного видео назад (если такое позволяется в приложении).

Параметр:
- position - позиция в секундах на которую было перемотано рекламное видео.

```objective-c
- (void)trackClose;
```
Должен быть вызван при закрытии рекламного видео пользователем.

```objective-c
- (void)trackFullscreen:(BOOL)isFullscreen;
```
Должен быть вызван при переходе в полноэкранный режим (isFullscreen=YES) и выходе из него (isFullscreen=NO).

```objective-c
- (void)trackMute:(BOOL)isMuted;
```
Должен быть вызван при выключении (isMuted=YES) и включении (isMuted=NO) звука.

```objective-c
- (void)trackError;
```
Должен быть вызван при каких-либо ошибках воспроизведения рекламного видео.

#### Обработка кликов

```objective-c
- (void)handleClick;
```
Должен быть вызван при клике пользователем на кнопку или плашку с призывом к действию.

Метод засчитает событие клика и выполнит перенаправление пользователя на сайт или в магазин приложений исходя из целевого адреса креатива.

### Включение режима отладки
В режиме отладки SDK будет сообщать в консоль о своих действиях. Сообщения от SDK можно фильтровать по тегу [mediascope].

Режим отладки можно включить через статический метод классов MSPVideoAdLoader или MSPVideoAdTracker. Включение режима отладки имеет глобальное действие и применяется ко всем компонентнам SDK.
```objective-c
[MSPVideoAdLoader setLogEnabled:YES];
```
или
```objective-c
[MSPVideoAdTracker setLogEnabled:YES];
```