---
layout: post
permalink: when-Thor-and-Hulk-meet-dagger2-rxjava-1
title: When the Avengers meet Dagger2, RxJava and Retrofit in a clean way
---

Recently, a lot articles, frameworks, and talks at the android community, are appearing talking about testing and software architecture, as said in the last [Droidcon Spain](http://es.droidcon.com/2015/speakers/), we are focusing on how to do robust applications instead on how to develop specific features. That denotes that the Android framework and the current Android commnunity is reaching some level of maturity.

Today, if you are an Android developer and don't recognize the words [Dagger 2](http://google.github.io/dagger/), [RxJava](https://github.com/ReactiveX/RxJava) or [Retrofit](http://square.github.io/retrofit/), you are missing something, this series will put some focus on giving the basic ideas of how to use these frameworks together with a [Clean Architecture](http://blog.8thlight.com/uncle-bob/2012/08/13/the-clean-architecture.html) perspective.

My first idea was to write a single article, but seeing the large amount of contents that these frameworks provide I've decided to create a series with at least 3 articles.

As always, all the code is published in [GitHub](https://github.com/saulmm/Avengers), please, all the recomendations, errors and comments are welcome, sorry if I don't have much time to answer all :)

![](http://androcode.es/wp-content/uploads/2015/05/avengers_list-e1431571424213.png)

<br>
## Dependency Injectors & Dagger 2

It took some time to find out how this framework works, so I would like to make clear the way in which I have learned to use it.

[Dagger 2](http://google.github.io/dagger/) It's based on the [dependency injection](http://en.wikipedia.org/wiki/Dependency_injection) pattern.

Look at the following snippet:

```java
    // Thor is awesome. He has a hammer!
    public class Thor extends Avenger {
        private final AvengerWeapon myAmazingHammer;

        public Thor (AvengerWeapon anAmazingHammer) {
            myAmazingHammer = anAmazingHammer;
        }

        public void doAmazingThorWork () {
            myAmazingHammer.hitSomeone();
        }
    }
```

Thor needs an `AvengerWeapon` to work correctly, the basic idea of the dependency injection pattern is that Thor would get fewer benefits if he creates its own `AvengerWeapon` than passing them throught his constructor. If Thor creates his own hammer he would be increasing the coupling.

`AvengerWeapon` could be an interface that would be implemented and injected in different ways depending our logic.

In Android, because how the framework is designed, it's not always easy access to the constructors, `Activity` and `Fragment` are examples.

That's where the dependency injectors like (http://google.github.io/dagger/), [Dagger](http://square.github.io/dagger/) or [Guice](https://github.com/google/guice) can bring benefits.

With [Dagger 2](http://google.github.io/dagger/) we would transform the previous code in something like this:

```java
    // Thor is awesome. He has a hammer!
    public class Thor extends Avenger {
        @Inject AvengerWeapon myAmazingHammer;

        public void doAmazingThorWork () {
            myAmazingHammer.hitSomeone();
        }
    }
```

We are not accessing to the Thor's constructor directly, the injector, with a few directives is in charge to build the Thor's hammer

```java
    public class ThorHammer extends AvengerWeapon () {

        @Inject public AvengerWeapon() {

            initGodHammer();
        }
    }
```

The `@Inject` annotation indicates to [Dagger 2](http://google.github.io/dagger/) which constructor has to use to build the Thor's hammer.

<br>
### Dagger 2  

[Dagger 2](http://google.github.io/dagger/) is promoted and implemented by Google, forked from [Dagger](http://square.github.io/dagger/), which it was created by [Square](https://corner.squareup.com/).

First of all it's necessary to configure the annotations processor, the `android-apt` plugin plays that role, allowing to use the annotations processor without inserting it into the final .apk. The processor also configures the source code generated by this processor.

`build.gradle` (In the root of the project)

```javascript

    dependencies {
        ...
        classpath 'com.neenbedankt.gradle.plugins:android-apt:1.4'
    }
```

`build.gradle` (In your android module)

```javascript

    apply plugin: 'com.neenbedankt.android-apt'

    dependencies {
        ...
        apt 'com.google.dagger:dagger-compiler:2.0'
    }
```

<br>
### Components, modules & Avengers

The modules are those who provide dependencies, and components who inject them

Here is an example:

```java
@Module
public class AppModule {
    private final AvengersApplication mAvengersApplication;

    public AppModule(AvengersApplication avengersApplication) {
        this.mAvengersApplication = avengersApplication;
    }

    @Provides @Singleton 
    AvengersApplication provideAvengersAppContext () { 
        return mAvengersApplication; 
    }
    
    @Provides @Singleton 
    Repository provideDataRepository (RestRepository restRepository) { 
        return restRepository; 
    }
}
```

This is the main module, we are interested in its dependencies to survive during the lifetime of the application, a general context and a repository for retrieve the information.

Simple, right?

<br>
With the `@Provides` annotation we are saying to Dagger 2 how a dependency has to be build if required. Otherwhise if we wouldn't indicate a provider for a particular dependency, Dagger 2 will go to find it to the constructor annotated with: `@Inject`.

The modules are used by components to inject dependencies, see the component of this module:


```java
@Singleton @Component(modules = AppModule.class)
public interface AppComponent {

    AvengersApplication app();
    Repository dataRepository();
}
```

This module isn't invoked by any activity or fragment, instead, is obtained by a more complex module to provide these dependencies where are required

```java
AvengersApplication app();
Repository dataRepository();
```

The components must to expose their dependencies to the graph (the dependencies that the module provides), i.e, the dependencies provided by this module **must** to be visible to other components that have this component as a dependency of the other components, if these dependencies are not visible Dagger 2 could not inject them where are required.

Here's our tree of dependencies:
![](http://androcode.es/wp-content/uploads/2015/05/Dagger-graph.png)

```java
@Module
public class AvengersModule {

    @Provides @Activity
    List<Character> provideAvengers() {

        List<Character> avengers = new ArrayList<>(6);
        
        avengers.add(new Character(
            "Iron Man", R.drawable.thumb_iron_man, 1009368));

        avengers.add(new Character(
            "Thor", R.drawable.thumb_thor, 1009664));

        avengers.add(new Character(
            "Captain America", R.drawable.thumb_cap,1009220));

        avengers.add(new Character(
            "Black Widow", R.drawable.thumb_nat, 1009189));

        avengers.add(new Character(
            "Hawkeye", R.drawable.thumb_hawkeye, 1009338));

        avengers.add(new Character(
            "Hulk", R.drawable.thumb_hulk, 1009351));

        return avengers;
    }
}
```

This module will be used to inject their dependencies into a particular activity, in fact will be who is responsible for painting the Avengers list:

```java
@Activity 
@Component(
    dependencies = AppComponent.class, 
    modules = {
        AvengersModule.class, 
        ActivityModule.class
    }
)
public interface AvengersComponent extends ActivityComponent {

    void inject (AvengersListActivity activity);
    List<Character> avengers();
}
```

Again we expose our dependency `List<Character>` to other components, and in this case there is a new method: `void inject (AvengersListActivity activity)`. **At the time this method is called**, the dependencies will be available to be consumed. These are injected into `AvengerListActivity`.

<br>
### All mixed

Our class `AvengersApplication`, will be responsible to provide the component of the application to other components, note that this doesn't order to inject any dependency, only provides the component.

Another reminder is that [Dagger 2](http://google.github.io/dagger/) generates the necessary elements at compile time, if you don't build your project, you aren't going to find the `DaggerAppComponent` class. 

Dagger 2 is who generates this class from your component with the following format: `Dagger`$$`{YourComponent}`.

`AvengersApplication.java`

```java
public class AvengersApplication extends Application {

    private AppComponent mAppComponent;

    @Override
    public void onCreate() {

        super.onCreate();
        initializeInjector();
    }

    private void initializeInjector() {

        mAppComponent = DaggerAppComponent.builder()
            .appModule(new AppModule(this))
            .build();
    }

    public AppComponent getAppComponent() {

        return mAppComponent;
    }
}
```


`AvengersListActivity.java`

```java
public class AvengersListActivity extends Activity 
    implements AvengersView {

    @InjectView(R.id.activity_avengers_recycler) 
    RecyclerView mAvengersRecycler;

    @InjectView(R.id.activity_avengers_toolbar) 
    Toolbar mAvengersToolbar;

    @Inject 
    AvengersListPresenter mAvengersListPresenter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {

        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_avengers_list);
        ButterKnife.inject(this);

        initializeToolbar();
        initializeRecyclerView();
        initializeDependencyInjector();
        initializePresenter();
    }

    private void initializeDependencyInjector() {

        AvengersApplication avengersApplication = 
            (AvengersApplication) getApplication();

        DaggerAvengersComponent.builder()
            .avengersModule(new AvengersModule())
            .activityModule(new ActivityModule(this))
            .appComponent(avengersApplication.getAppComponent())
            .build().inject(this);
    }
```

When the `initializeDependencyInjector()` is in `.inject(this)` [Dagger 2](http://google.github.io/dagger/) starts to work and provides the necessary dependencies, remember that [Dagger 2](http://google.github.io/dagger/) is strict when it comes to make the injection, I mean, the component must be **exactly the same** class who call the `inject()` method of the component.

`AvengersComponent.java`

```java
...
public interface AvengersComponent extends ActivityComponent {
 
    void inject (AvengersListActivity activity);
    List<Character> avengers();
}
```

Otherwise dependencies will be unresolved. In this case, the presenter is initialized with the Avengers that are provided by Dagger 2:

```java
public class AvengersListPresenter implements Presenter, RecyclerClickListener {

    private final List<Character> mAvengersList;
    private final Context mContext;
    private AvengersView mAvengersView;
    private Intent mIntent;

    @Inject public AvengersListPresenter (List<Character> avengers, Context context) {

        mAvengersList = avengers;
        mContext = context;
    }
```

[Dagger 2](http://google.github.io/dagger/) will resolve the presenter because has a `@Inject` annotation. The argument that the constructor receives is solved by [Dagger 2](http://google.github.io/dagger/), because it knows how to build it thanks to the `@Provides` method in the module.

<br>
## Conclusion

The power of a good use of an injector like [Dagger 2](http://google.github.io/dagger/) is indisputable, imagine that you have different strategies depending the Api Level of the framework, the possibilities are endless.

<br>
## Resources:

- **Chiu-Ki Chan** - [Dagger 2 + Espresso + Mockito](http://blog.sqisland.com/2015/04/dagger-2-espresso-2-mockito.html)

- **Fernando Cejas** - [Tasting Dagger 2 on Android](http://fernandocejas.com/2015/04/11/tasting-dagger-2-on-android/)

- **Google Developers** - [Dagger 2, A new type of dependency injection](https://www.youtube.com/watch?v=oK_XtfXPkqw)

- **Mike Gouline** - [Dagger 2, Even sharper, less square](http://blog.gouline.net/2015/05/04/dagger-2-even-sharper-less-square/)