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

