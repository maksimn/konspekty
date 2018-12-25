# Android 

## Внедрение зависимостей с использованием Toothpick

Плюс использования DI - в переиспользовании на уровне объектов. Если вы имеете Activity, Fragment и View, которым всем нужен доступ к одному и тому же объекту состояния, то вам не нужно получать это состояние в активити и передавать его во фрагмент, из фрагмента - в адаптер, из адаптера - во вьюху. Вы можете просто объявить, что данная сущность будет внедряться в каждом требуемом случае - во фрагменте, в активити и т. д. 

---

Пример внедрения зависимости:

```java
public class SmoothieMachine {

    // Внедрение поля
    @Inject IceMachine iceMachine; // зависимость
	
	public void doSmoothie() {
	    iceMachine.makeIce();
	}
}

@Singleton // аннотация - будет создан единственный экзмепляр на всё приложение 
public class IceMachine {
    Foo foo;
	
	// Внедрение конструктора
	@Inject
	public IceMachine(Foo foo) {
	    this.foo = foo;
	}
}
```

Вы не создаете зависимости "своими руками" -- это ответственность библиотеки. 

2-й случай означает, что когда вы пожелаете экземпляр IceMachine, то Toothpick сначала создаст объект Foo foo и передаст его конструктору, и после это сможет вернуть экземпляр IceMachine. 

Внедрение через конструктор нужно использовать, если вам нужна зависимость сразу с момента создания объекта. А иначе проще использовать внедрение через поле.

Есть поддержка ленивого создания объектов.

---

### Scope

Что это такое? В Даггере он скрыт. 

Scope - это место, где вы выполняете внедрение. В Toothpick с ним нужно работать в явном виде, вы обязаны так делать.

Пример

```java
public class MyApplication extends Application {
    @Inject Machine machine;
	
	@Override
	protected void onCreate() {
	    super.onCreate();
		
		Scope appScope = Toothpick.openScope(this); // здесь задается имя scope - через экземпляр приложения
		appScope.installModules(...);
        Toothpick.inject(this, appScope); // внедрить объект приложения в пределах application scope
	}
}
```

Этот scope живет в течении всего времени жизни приложения.

Внедрение в appScope означает, что в пределах application scope вы должны найти все ваши зависимости. 

Что находится в __scope__? В scope находятся __bindings__ (указания, какую реализацию внедрять для данной зависимости). 

`Machine` --> `IceMachine`

Связь между bindings и scopes - __modules__. Вы определяете ваши биндинги в модуле, и инсталлируете модуль в scope.

```java
public class MyApplication extends Application {
    @Inject Machine machine;
	
	@Override
	protected void onCreate() {
	    super.onCreate();
		
		Scope appScope = Toothpick.openScope(this); // здесь задается имя scope - через экземпляр приложения
		appScope.installModules(new Module() {{ // сюда можно передать varargs модулей
		    bind(Machine.class).to(IceMachine.class)
		}});
        Toothpick.inject(this, appScope); // внедрить объект приложения в пределах application scope
	}
}
```

Scope'ы образуют одно дерево.

```
[Application Scope]
      |
      |
[Activity1 Scope]
```

Обычно создается Application Scope в классе Application. И внутри активити вы делаете то же самое. Когда Активити появляется на экране, то оно имеет свой собственный Scope, являющийся потомком Application Scope. Это означает, что все биндинги, которые доступны в Application Scope, будут также доступны внутри активити. И внутри activity scope можно определить свои биндинги. То есть всё, что доступно глобально, на уровне приложения - например, класс конфигурации, экземпляр какого-то состояния, Android Service - всё это будет доступно внутри Activity Scope. 

При смерти активити нужно уничтожить его Scope (destroy scope). И объекты из scope будут собраны сборщиком мусора. И оно пересоздастся заново при повторном создании активити. 

Точно также может быть [Service 1 Scope] - область видимости фоновой андроид-службы, потомок Application Scope. [Fragment1 Scope] как потомок Activity1 Scope. Это может быть еще ниже - scope для AsyncTask и для чего угодно. 

Как образовать дерево из scope? Например, в Активити

```kotlin
public class LemonActivity extends Activity {
    @Inject Machine machine;
	@Inject Context context;
	
	@Override
	protected void onCreate(Bundle savedInstanceState) {
	    super.onCreate(savedInstanceState);
		Scope scope = Toothpick.openScopes(getApplication(), this); // 1
		scope.installModules(...);
		Toothpick.inject(this, scope);
	}
}
``` 

1 - указывается родительский scope. 

Идея такого дерева scope'ов состоит в том, чтобы организовать память таким образом, чтобы вы можете поместить много вещей внутри scope.

В Application Scope - синглтоны для всего приложения. 

В Activity Scope - синглтоны для активити. Это могут быть состояния, которые вы шарите между фрагментами и вью. 

__Для определения синглтонов__ внутри __модулей__ используется метод `bind(Machine.class).toInstance(new IceMachine());`.

Или вместо этого можно использовать аннотации

```java
@Singleton
public class IceMachine {
}

@ActivitySingleton
public class SmoothieMachine {
}
```

Тогда Toothpick понимает, что каждый раз, когда вы напрямую внедряете IceMachine, он создается как синглтон и потом подвергается очистке. То же самое - на уровне Активити.

Итог - есть 2 разных способа создания синглтонов. 

### MVP & State preservation

Это более продвинутая (сложная) тема. 

Мы хотим сохранять состояние неизменным при смене конфигурации (повороте экрана) устройства. 

Сериализуемые объекты можно сохранить в бандл, и потом десериализовать. 

А несериализуемые? Например соединение - с сетью или базой данных. Вы хотите, чтобы они остались функционирующими при смене конфигурации. 

Можно сделать scope, который будет областью RAM, которая будет переживать момент поворота экрана. Как это сделать?

```
[Application scope]
    |
    |
[Activity scope]
```

Что есть ввести промежуточный scope, не зависящий от жизненного цикла активити?

```
[Application scope]
    |
    |
[Presenter scope] - (экземпляр презентера)
    |
    |
[Activity scope]
```

Внутри активити:

```java
public class MyActivity extends Activity {

    @Inject MyPresenter presenter;
	
	@Override
	protected void onCreate(Bundle savedInstanceState) {
	    super.onCreate(savedInstanceState);
		Scope scope = Toothpick.openScopes(getApplication(),
		                         PresenterSingleton.class,
                                 this);
		Toothpick.inject(this, scope);
	}
}
```

и тогда это будет состояние, которое переживет поворот экрана. Этот код задает ветку в дереве Toothpick scope'ов, начинающуюся с Application scope, и содержащую Presenter, который будет посередине между Application scope и Activity scope. 

В действительности это сокращение - вы можете дать любое имя этому scope-in-the-middle. Но если вы дадите ему имя аннотации PresenterSingleton, то вы можете использовать один и тот же класс для аннотирования чего-либо, и он скажет Toothpick'у: когда я хочу сделать внедрение данной штуки, пожалуйста, сделай его синглтоном из данного scope с данным именем. Поэтому Toothpick автоматически положит ваш синглтон в Presenter scope. Ничего другого будет не нужно делать.

Есть API для задания другого имени, так что вы можете его использовать если хотите.

```
[Presenter scope]
```
```java
@PresenterSingleton
public class MyPresenter {
    String dealId;
	int quantity;
}
```

Так вы получите один и тот же экземпляр, который переживет ротацию. 

Есть и другая "школа мысли", как достигнуть этого.

### Mutli activity flows

Пусть несколько активити образуют purchase flow, по которому должен пройти пользователь

```
[Activity 1]  ->  [Activity 2]  ->  [Activity 3]		 
```

в пределах которого имеется корзина для покупок. В самом андроиде нет объекта, который бы представил такой flow.

Первый способ работы - создать корзину в Activity 1 и передавать от одного активити в другое, используя Intent'ы. Это значит, что ваша корзина Serializable или Parcelable. 

Но тут может быть сложное бранчевание с переходами не только вперед, но и назад к прошлым активити. И тут уже будет сложно работать с интентами и корзиной. 

Другой подход - самостоятельная сериализация корзины на диск (файл, БД и др.). Это не очень быстро и удобно. 

Третий подход - использовать DI:

```
Application scope
     |
     |
Flow scope --- \  --------------- \
     |           \                  \ 
	 |             \                  \
Activity 1 scope    Activity 2 scope   Activity 3 scope
``` 

Корзина будет в памяти во flow scope и расшарена между активити. И выживать при ротациях.


```java
public class MyActivity extends Activity {
    
	@Inject ShoppingCart shoppingCart;
	
	@Override
	protected void onCreate(Bundle savedInstanceState) {
	    super.onCreate(savedInstanceState);
		Scope scope = openScopes(getApplication(),
		                         FlowSingleton.class,
                                 this); // activity name
		Toothpick.inject(this, scope);
	}
}

@FlowSingleton // аннотация означает, что этот класс - часть flow scope
public class ShoppingCart {
    List<PurchaseItem> purchases...
}
```

Тогда объект корзины будет доступен во всех активити. В конце нужно не забыть освободить flow scope.

### Тестирование с Toothpick

Mockito, EasyMock - для подмены реализаций можно выбрать что угодно.