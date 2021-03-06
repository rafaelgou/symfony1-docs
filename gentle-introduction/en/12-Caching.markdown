Chapter 12 - Caching
====================

One of the ways to speed up an application is to store chunks of generated HTML code, or even full pages, for future requests. This technique is known as caching, and it can be managed on the server side and on the client side.

Symfony offers a flexible server-caching system. It allows saving the full response, the result of an action, a partial, or a template fragment into a file, through a very intuitive setup based on YAML files. When the underlying data changes, you can easily clear selective parts of the cache with the command line or special action methods. Symfony also provides an easy way to control the client-side cache through HTTP 1.1 headers. This chapter deals with all these subjects, and gives you a few tips on monitoring the improvements that caching can bring to your applications.

Caching the Response
--------------------

The principle of HTML caching is simple: Part or all of the HTML code that is sent to a user upon a request can be reused for a similar request. This HTML code is stored in a special place (the `cache/` folder in symfony), where the front controller will look for it before executing an action. If a cached version is found, it is sent without executing the action, thus greatly speeding up the process. If no cached version is found, the action is executed, and its result (the view) is stored in the cache folder for future requests.

As all the pages may contain dynamic information, the HTML cache is disabled by default. It is up to the site administrator to enable it in order to improve performance.

Symfony handles three different types of HTML caching:

  * Cache of an action (with or without the layout)
  * Cache of a partial or a component
  * Cache of a template fragment

The first two types are handled with YAML configuration files. Template fragment caching is managed by calls to helper functions in the template.

### Global Cache Settings

For each application of a project, the HTML cache mechanism can be enabled or disabled (the default), per environment, in the `cache` setting of the `settings.yml` file. Listing 12-1 demonstrates enabling the cache.

Listing 12-1 - Activating the Cache, in `frontend/config/settings.yml`

    dev:
      .settings:
        cache: true

### Caching an Action

Actions displaying static information (not depending on database or session-dependent data) or actions reading information from a database but without modifying it (typically, GET requests) are often ideal for caching. Figure 12-1 shows which elements of the page are cached in this case: either the action result (its template) or the action result together with the layout.

Figure 12-1 - Caching an action

![Caching an action](http://www.symfony-project.org/images/book/1_4/F1201.png "Caching an action")

For instance, consider a `user/list` action that returns the list of all users of a website. Unless a user is modified, added, or removed (and this matter will be discussed later in the "Removing Items from the Cache" section), this list always displays the same information, so it is a good candidate for caching.

Cache activation and settings, action by action, are defined in a `cache.yml` file located in the module `config/` directory. See Listing 12-2 for an example.

Listing 12-2 - Activating the Cache for an Action, in `frontend/modules/user/config/cache.yml`

    list:
      enabled:     true
      with_layout: false   # Default value
      lifetime:    86400   # Default value

This configuration stipulates that the cache is on for the list action, and that the layout will not be cached with the action (which is the default behavior). It means that even if a cached version of the action exists, the layout (together with its partials and components) is still executed. If the `with_layout` setting is set to `true`, the layout is cached with the action and not executed again.

To test the cache settings, call the action in the development environment from your browser.

    http://myapp.example.com/frontend_dev.php/user/list

You will notice a border around the action area in the page. The first time, the area has a blue header, showing that it did not come from the cache. Refresh the page, and the action area will have a yellow header, showing that it did come from the cache (with a notable boost in response time). You will learn more about the ways to test and monitor caching later in this chapter.

>**NOTE**
>Slots are part of the template, and caching an action will also store the value of the slots defined in this action's template. So the cache works natively for slots.

The caching system also works for pages with arguments. The `user` module may have, for instance, a `show` action that expects an `id` argument to display the details of a user. Modify the `cache.yml` file to enable the cache for this action as well, as shown in Listing 12-3.

In order to organize your `cache.yml`, you can regroup the settings for all the actions of a module under the `all:` key, also shown in Listing 12-3.

Listing 12-3 - A Full `cache.yml` Example, in `frontend/modules/user/config/cache.yml`

    list:
      enabled:    true
    show:
      enabled:    true

    all:
      with_layout: false   # Default value
      lifetime:    86400   # Default value

Now, every call to the `user/show` action with a different `id` argument creates a new record in the cache. So the cache for this:

    http://myapp.example.com/user/show/id/12

will be different than the cache for this:

    http://myapp.example.com/user/show/id/25

>**CAUTION**
>Actions called with a POST method or with GET parameters are not cached.

The `with_layout` setting deserves a few more words. It actually determines what kind of data is stored in the cache. For the cache without layout, only the result of the template execution and the action variables are stored in the cache. For the cache with layout, the whole response object is stored. This means that the cache with layout is much faster than the cache without it.

If you can functionally afford it (that is, if the layout doesn't rely on session-dependent data), you should opt for the cache with layout. Unfortunately, the layout often contains some dynamic elements (for instance, the name of the user who is connected), so action cache without layout is the most common configuration. However, RSS feeds, pop-ups, and pages that don't depend on cookies can be cached with their layout.

### Caching a Partial or Component

Chapter 7 explained how to reuse code fragments across several templates, using the `include_partial()` helper. A partial is as easy to cache as an action, and its cache activation follows the same rules, as shown in Figure 12-2.

Figure 12-2 - Caching a partial or component

![Caching a partial or component](http://www.symfony-project.org/images/book/1_4/F1202.png "Caching a partial or component")

For instance, Listing 12-4 shows how to edit the `cache.yml` file to enable the cache on a `_my_partial.php` partial located in the `user` module. Note that the `with_layout` setting doesn't make sense in this case.

Listing 12-4 - Caching a Partial, in `frontend/modules/user/config/cache.yml`

    _my_partial:
      enabled:    true
    list:
      enabled:    true
    ...

Now all the templates using this partial won't actually execute the PHP code of the partial, but will use the cached version instead.

    [php]
    <?php include_partial('user/my_partial') ?>

Just as for actions, partial caching is also relevant when the result of the partial depends on parameters. The cache system will store as many versions of a template as there are different values of parameters.

    [php]
    <?php include_partial('user/my_other_partial', array('foo' => 'bar')) ?>

>**TIP**
>The action cache is more powerful than the partial cache, since when an action is cached, the template is not even executed; if the template contains calls to partials, these calls are not performed. Therefore, partial caching is useful only if you don't use action caching in the calling action or for partials included in the layout.

A little reminder from Chapter 7: A component is a light action put on top of a partial. This inclusion type is very similar to partials, and support caching in the same way. For instance, if your global layout includes a component called `day` with `include_component('general/day')` in order to show the current date, set the `cache.yml` file of the `general` module as follows to enable the cache on this component:

    _day:
      enabled: true

When caching a component or a partial, you must decide whether to store a single version for all calling templates or a version for each template. By default, a component is stored independently of the template that calls it. But contextual components, such as a component that displays a different sidebar with each action, should be stored as many times as there are templates calling it. The caching system can handle this case, provided that you set the `contextual` parameter to `true`, as follows:

    _day:
      contextual: true
      enabled:    true

>**NOTE**
>Global components (the ones located in the application `templates/` directory) can be cached, provided that you declare their cache settings in the application `cache.yml`.

### Caching a Template Fragment

Action caching applies to only a subset of actions. For the other actions--those that update data or display session-dependent information in the template--there is still room for cache improvement but in a different way. Symfony provides a third cache type, which is dedicated to template fragments and enabled directly inside the template. In this mode, the action is always executed, and the template is split into executed fragments and fragments in the cache, as illustrated in Figure 12-3.

Figure 12-3 - Caching a template fragment

![Caching a template fragment](http://www.symfony-project.org/images/book/1_4/F1203.png "Caching a template fragment")

For instance, you may have a list of users that shows a link of the last-accessed user, and this information is dynamic. The `cache()` helper defines the parts of a template that are to be put in the cache. See Listing 12-5 for details on the syntax.

Listing 12-5 - Using the `cache()` Helper, in `frontend/modules/user/templates/listSuccess.php`

    [php]
    <!-- Code executed each time -->
    <?php echo link_to('last accessed user', 'user/show?id='.$last_accessed_user_id) ?>

    <!-- Cached code -->
    <?php if (!cache('users')): ?>
      <?php foreach ($users as $user): ?>
        <?php echo $user->getName() ?>
      <?php endforeach; ?>
      <?php cache_save() ?>
    <?php endif; ?>

Here's how it works:

  * If a cached version of the fragment named 'users' is found, it is used to replace the code between the `<?php if (!cache($unique_fragment_name)): ?>` and the `<?php endif; ?>` lines.
  * If not, the code between these lines is processed and saved in the cache, identified with the unique fragment name.

The code not included between such lines is always processed and not cached.

>**CAUTION**
>The action (`list` in the example) must not have caching enabled, since this would bypass the whole template execution and ignore the fragment cache declaration.

The speed boost of using the template fragment cache is not as significant as with the action cache, since the action is always executed, the template is partially processed, and the layout is always used for decoration.

You can declare additional fragments in the same template; however, you need to give each of them a unique name so that the symfony cache system can find them afterwards.

As with actions and components, cached fragments can take a lifetime in seconds as a second argument of the call to the `cache()` helper.

    [php]
    <?php if (!cache('users', 43200)): ?>

The default cache lifetime (86400 seconds, or one day) is used if no parameter is given to the helper.

>**TIP**
>Another way to make an action cacheable is to insert the variables that make it vary into the action's routing pattern. For instance, if a home page displays the name of the connected user, it cannot be cached unless the URL contains the user nickname. Another example is for internationalized applications: If you want to enable caching on a page that has several translations, the language code must somehow be included in the URL pattern. This trick will multiply the number of pages in the cache, but it can be of great help to speed up heavily interactive applications.

### Configuring the Cache Dynamically

The `cache.yml` file is one way to define cache settings, but it has the inconvenience of being invariant. However, as usual in symfony, you can use plain PHP rather than YAML, and that allows you to configure the cache dynamically.

Why would you want to change the cache settings dynamically? A good example is a page that is different for authenticated users and for anonymous ones, but the URL remains the same. Imagine an `article/show` page with a rating system for articles. The rating feature is disabled for anonymous users. For those users, rating links trigger the display of a login form. This version of the page can be cached. On the other hand, for authenticated users, clicking a rating link makes a POST request and creates a new rating. This time, the cache must be disabled for the page so that symfony builds it dynamically.

The right place to define dynamic cache settings is in a filter executed before the `sfCacheFilter`. Indeed, the cache is a filter in symfony, just like the the security features. In order to enable the cache for the `article/show` page only if the user is not authenticated, create a `conditionalCacheFilter` in the application `lib/` directory, as shown in Listing 12-6.

Listing 12-6 - Configuring the Cache in PHP, in `frontend/lib/conditionalCacheFilter.class.php`

    [php]
    class conditionalCacheFilter extends sfFilter
    {
      public function execute($filterChain)
      {
        $context = $this->getContext();
        if (!$context->getUser()->isAuthenticated())
        {
          foreach ($this->getParameter('pages') as $page)
          {
            $context->getViewCacheManager()->addCache($page['module'], $page['action'], array('lifeTime' => 86400));
          }
        }

        // Execute next filter
        $filterChain->execute();
      }
    }

You must register this filter in the `filters.yml` file before the `sfCacheFilter`, as shown in Listing 12-7.

Listing 12-7 - Registering Your Custom Filter, in `frontend/config/filters.yml`

    ...
    security: ~

    conditionalCache:
      class: conditionalCacheFilter
      param:
        pages:
          - { module: article, action: show }

    cache: ~
    ...

Clear the cache (to autoload the new filter class), and the conditional cache is ready. It will enable the cache of the pages defined in the pages parameter only for users who are not authenticated.

The `addCache()` method of the `sfViewCacheManager` object expects a module name, an action name, and an associative array with the same parameters as the ones you would define in a `cache.yml` file. For instance, if you want to define that the `article/show` action must be cached with the layout and with a lifetime of 3600 seconds, then write the following:

    [php]
    $context->getViewCacheManager()->addCache('article', 'show', array(
      'withLayout' => true,
      'lifeTime'   => 3600,
    ));

>**SIDEBAR**
>Alternative Caching storage
>
>By default, the symfony cache system stores data in files on the web server hard disk. You may want to store cache in memory (for instance, via [memcached](http://www.danga.com/memcached/)) or in a database (notably if you want to share your cache among several servers or speed up cache removal). You can easily alter symfony's default cache storage system because the cache class used by the symfony view cache manager is defined in `factories.yml`.
>
>The default view cache storage factory is the `sfFileCache` class:
>
>     view_cache:
>         class: sfFileCache
>         param:
>           automaticCleaningFactor: 0
>           cacheDir:                %SF_TEMPLATE_CACHE_DIR%
>
>You can replace the `class` with your own cache storage class or with one of the symfony alternative classes (including `sfAPCCache`, `sfEAcceleratorCache`, `sfMemcacheCache`, `sfSQLiteCache`, and `sfXCacheCache`). The parameters defined under the `param` key are passed to the constructor of the cache class as an associative array. Any view cache storage class must implement all methods found in the abstract `sfCache` class. Refer to the Chapter 19 for more information on this subject.

### Using the Super Fast Cache

Even a cached page involves some PHP code execution. For such a page, symfony still loads the configuration, builds the response, and so on. If you are really sure that a page is not going to change for a while, you can bypass symfony completely by putting the resulting HTML code directly into the `web/` folder. This works thanks to the Apache `mod_rewrite` settings, provided that your routing rule specifies a pattern ending without a suffix or with `.html`.

You can do this by hand, page by page, with a simple command-line call:

    $ curl http://myapp.example.com/user/list.html > web/user/list.html

After that, every time that the `user/list` action is requested, Apache finds the corresponding `list.html` page and bypasses symfony completely. The trade-off is that you can't control the page cache with symfony anymore (lifetime, automatic deletion, and so on), but the speed gain is very impressive.

Alternatively, you can use the `sfSuperCache` symfony plug-in, which automates the process and supports lifetime and cache clearing. Refer to Chapter 17 for more information about plug-ins.

>**SIDEBAR**
>Other Speedup tactics
>
>In addition to the HTML cache, symfony has two other cache mechanisms, which are completely automated and transparent to the developer. In the production environment, the configuration and the template translations are cached in files stored in the `myproject/cache/config/` and `myproject/cache/i18n/` directories without any intervention.
>
>PHP accelerators (eAccelerator, APC, XCache, and so on), also called opcode caching modules, increase performance of PHP scripts by caching them in a compiled state, so that the overhead of code parsing and compiling is almost completely eliminated. This is particularly effective for the Propel classes, which contain a great amount of code. These accelerators are compatible with symfony and can easily triple the speed of an application. They are recommended in production environments for any symfony application with a large audience.
>
>With a PHP accelerator, you can manually store persistent data in memory, to avoid doing the same processing for each request, with the `sfAPCCache` class if use APC for instance. And if you want to cache the result of a CPU-intensive function, you will probably use the `sfFunctionCache` object. Refer to Chapter 18 for more information about these mechanisms.

Removing Items from the Cache
-----------------------------

If the scripts or the data of your application change, the cache will contain outdated information. To avoid incoherence and bugs, you can remove parts of the cache in many different ways, according to your needs.

### Clearing the Entire Cache

The `cache:clear` task of the symfony command line erases the cache (HTML, configuration, routing, and i18n cache). You can pass it arguments to erase only a subset of the cache, as shown in Listing 12-8. Remember to call it only from the root of a symfony project.

Listing 12-8 - Clearing the Cache

    // Erase the whole cache
    $ php symfony cache:clear

    // Short syntax
    $ php symfony cc

    // Erase only the cache of the frontend application
    $ php symfony cache:clear --app=frontend

    // Erase only the HTML cache of the frontend application
    $ php symfony cache:clear --app=frontend --type=template

    // Erase only the configuration cache of the frontend application
    // The built-types are config, i18n, routing, and template.
    $ php symfony cache:clear --app=frontend --type=config

    // Erase only the configuration cache of the frontend application and the prod environment
    $ php symfony cache:clear --app=frontend --type=config --env=prod

### Clearing Selective Parts of the Cache

When the database is updated, the cache of the actions related to the modified data must be cleared. You could clear the whole cache, but that would be a waste for all the existing cached actions that are unrelated to the model change. This is where the `remove()` method of the `sfViewCacheManager` object applies. It expects an internal URI as argument (the same kind of argument you would provide to a `link_to()`), and removes the related action cache.

For instance, imagine that the `update` action of the `user` module modifies the columns of a `User` object. The cached versions of the `list` and `show` actions need to be cleared, or else the old versions, which contain erroneous data, are displayed. To handle this, use the `remove()` method, as shown in Listing 12-9.

Listing 12-9 - Clearing the Cache for a Given Action, in `modules/user/actions/actions.class.php`

    [php]
    public function executeUpdate($request)
    {
      // Update a user
      $user_id = $request->getParameter('id');
      $user = UserPeer::retrieveByPk($user_id);
      $this->forward404Unless($user);
      $user->setName($request->getParameter('name'));
      ...
      $user->save();

      // Clear the cache for actions related to this user
      $cacheManager = $this->getContext()->getViewCacheManager();
      $cacheManager->remove('user/list');
      $cacheManager->remove('user/show?id='.$user_id);
      ...
    }

Removing cached partials and components is a little trickier. As you can pass them any type of parameter (including objects), it is almost impossible to identify their cached version afterwards. Let's focus on partials, as the explanation is the same for the other template components. Symfony identifies a cached partial with a special prefix (`sf_cache_partial`), the name of the module, and the name of the partial, plus a hash of all the parameters used to call it, as follows:

    [php]
    // A partial called by
    <?php include_partial('user/my_partial', array('user' => $user) ?>

    // Is identified in the cache as
    @sf_cache_partial?module=user&action=_my_partial&sf_cache_key=bf41dd9c84d59f3574a5da244626dcc8

In theory, you could remove a cached partial with the `remove()` method if you knew the value of the parameters hash used to identify it, but this is very impracticable. Fortunately, if you add a `sf_cache_key` parameter to the `include_partial()` helper call, you can identify the partial in the cache with something that you know. As you can see in Listing 12-10, clearing a single cached partial --for instance, to clean up the cache from the partial based on a modified `User`-- becomes easy.

Listing 12-10 - Clearing Partials from the Cache

    [php]
    <?php include_partial('user/my_partial', array(
      'user'         => $user,
      'sf_cache_key' => $user->getId()
    ) ?>

    // Is identified in the cache as
    @sf_cache_partial?module=user&action=_my_partial&sf_cache_key=12

    // Clear _my_partial for a specific user in the cache with
    $cacheManager->remove('@sf_cache_partial?module=user&action=_my_partial&sf_cache_key='.$user->getId());

To clear template fragments, use the same `remove()` method. The key identifying the fragment in the cache is composed of the same `sf_cache_partial` prefix, the module name, the action name, and the `sf_cache_key` (the unique name of the cache fragment included by the `cache()` helper). Listing 12-11 shows an example.

Listing 12-11 - Clearing Template Fragments from the Cache

    [php]
    <!-- Cached code -->
    <?php if (!cache('users')): ?>
      ... // Whatever
      <?php cache_save() ?>
    <?php endif; ?>

    // Is identified in the cache as
    @sf_cache_partial?module=user&action=list&sf_cache_key=users

    // Clear it with
    $cacheManager->remove('@sf_cache_partial?module=user&action=list&sf_cache_key=users');

>**SIDEBAR**
>Selective Cache Clearing Can damage your Brain
>
>The trickiest part of the cache-clearing job is to determine which actions are influenced by a data update.
>
>For instance, imagine that the current application has a `publication` module where publications are listed (`list` action) and described (`show` action), along with the details of their author (an instance of the `User` class). Modifying one User record will affect all the descriptions of the user's publications and the list of publications. This means that you need to add to the `update` action of the `user` module, something like this:
>
>     [php]
>     $c = new Criteria();
>     $c->add(PublicationPeer::AUTHOR_ID, $request->getParameter('id'));
>     $publications = PublicationPeer::doSelect($c);
>
>     $cacheManager = sfContext::getInstance()->getViewCacheManager();
>     foreach ($publications as $publication)
>     {
>       $cacheManager->remove('publication/show?id='.$publication->getId());
>     }
>     $cacheManager->remove('publication/list');
>
>When you start using the HTML cache, you need to keep a clear view of the dependencies between the model and the actions, so that new errors don't appear because of a misunderstood relationship. Keep in mind that all the actions that modify the model should probably contain a bunch of calls to the `remove()` method if the HTML cache is used somewhere in the application.
>
>And, if you don't want to damage your brain with too difficult an analysis, you can always clear the whole cache each time you update the database...

### Clearing several cache parts at once

The `remove()` method accepts keys with wildcards. It allows you to remove several cache parts with a single call. You can do for instance:

    [php]
    $cacheManager->remove('user/show?id=*');    // Remove for all user records

Another good example is with applications handling several languages, where the language code appears in all URLs. The URL to a user profile page should look like this:

    http://www.myapp.com/en/user/show/id/12

To remove the cached profile of the user having an `id` of `12` in all languages, you can simply call:

    [php]
    $cache->remove('user/show?sf_culture=*&id=12');

This also works for partials:

    [php]
    $cacheManager->remove('@sf_cache_partial?module=user&action=_my_partial&sf_cache_key=*');    // Remove for all keys

The `remove()` method accepts two additional parameters, allowing you to define which hosts and vary headers you want to clear the cache for. This is because symfony keeps one cache version for each host and vary headers, so that two applications sharing the same code base but not the same hostname use different caches. This can be of great use, for instance, when an application interprets the subdomain as a request parameter (like `http://php.askeet.com` and `http://life.askeet.com`). If you don't set the last two parameters, symfony will remove the cache for the current host and for the `all` vary header. Alternatively, if you want to remove the cache for another host, call `remove()` as follows:

    [php]
    $cacheManager->remove('user/show?id=*');                     // Remove records for the current host and all users
    $cacheManager->remove('user/show?id=*', 'life.askeet.com');  // Remove records for the host life.askeet.com and all users
    $cacheManager->remove('user/show?id=*', '*');                // Remove records for every host and all users

The `remove()` method works in all the caching strategies that you can define in the `factories.yml` (not only `sfFileCache`, but also `sfAPCCache`, `sfEAcceleratorCache`, `sfMemcacheCache`, `sfSQLiteCache`, and `sfXCacheCache`).

### Clearing cache across applications

Clearing the cache across applications can be a problem. For instance, if an administrator modifies a record in the `user` table in a `backend` application, all the actions depending on this user in the `frontend` application need to be cleared from the cache. But the view cache manager available in the `backend` application doesn't know the `frontend` application routing rules (applications are isolated from each other). So you can't write this code in the backend:

    [php]
    $cacheManager = sfContext::getInstance()->getViewCacheManager(); // Retrieves the view cache manager of the backend
    $cacheManager->remove('user/show?id=12');                        // The pattern is not found, since the template is cached in the frontend

The solution is to initialize a `sfCache` object by hand, with the same settings as the frontend cache manager. Fortunately, all cache classes in symfony provide a `removePattern` method providing the same service as the view cache manager's `remove`.

For instance, if the `backend` application needs to clear the cache of the `user/show` action in the `frontend` application for the user of `id` `12`, it can use the following:

    [php]
    $frontend_cache_dir = sfConfig::get('sf_cache_dir').DIRECTORY_SEPARATOR.'frontend'.DIRECTORY_SEPARATOR.sfConfig::get('sf_environment').DIRECTORY_SEPARATOR.'template';
    $cache = new sfFileCache(array('cache_dir' => $frontend_cache_dir)); // Use the same settings as the ones defined in the frontend factories.yml
    $cache->removePattern('user/show?id=12');

For different caching strategies, you just need to change the cache object initialization, but the cache removal process remains the same:

    [php]
    $cache = new sfMemcacheCache(array('prefix' => 'frontend'));
    $cache->removePattern('user/show?id=12');

Testing and Monitoring Caching
------------------------------

HTML caching, if not properly handled, can create incoherence in displayed data. Each time you disable the cache for an element, you should test it thoroughly and monitor the execution boost to tweak it.

### Building a Staging Environment

The caching system is prone to new errors in the production environment that can't be detected in the development environment, since the HTML cache is disabled by default in development. If you enable the HTML cache for some actions, you should add a new environment, called staging in this section, with the same settings as the `prod` environment (thus, with cache enabled) but with `web_debug` set to `true`.

To set it up, edit the `settings.yml` file of your application and add the lines shown in Listing 12-12 at the top.

Listing 12-12 - Settings for a `staging` Environment, in `frontend/config/settings.yml`

    staging:
      .settings:
        web_debug:  true
        cache:      true

In addition, create a new front controller by copying the production one (probably `myproject/web/index.php`) to a new `frontend_staging.php`. Edit it to change the arguments passed to the `getApplicationConfiguration()` method, as follows:

    [php]
    $configuration = ProjectConfiguration::getApplicationConfiguration('frontend', 'staging', true);

That's it--you have a new environment. Use it by adding the front controller name after the domain name:

    http://myapp.example.com/frontend_staging.php/user/list

### Monitoring Performance

Chapter 16 will explore the web debug toolbar and its contents. However, as this toolbar offers valuable information about cached elements, here is a brief description of its cache features.

When you browse to a page that contains cacheable elements (action, partials, fragments, and so on), the web debug toolbar (in the top-right corner of the window) shows an ignore cache button (a green, rounded arrow), as shown in Figure 12-4. This button reloads the page and forces the processing of cached elements. Be aware that it does not clear the cache.

The last number on the right side of the debug toolbar is the duration of the request execution. If you enable cache on a page, this number should decrease the second time you load the page, since symfony uses the data from the cache instead of reprocessing the scripts. You can easily monitor the cache improvements with this indicator.

Figure 12-4 - Web debug toolbar for pages using caching

![Web debug toolbar for pages using caching](http://www.symfony-project.org/images/book/1_4/F1204.png "Web debug toolbar for pages using caching")

The debug toolbar also shows the number of database queries executed during the processing of the request, and the detail of the durations per category (click the total duration to display the detail). Monitoring this data, in conjunction with the total duration, will help you do fine measures of the performance improvements brought by the cache.

### Benchmarking

The debug mode greatly decreases the speed of your application, since a lot of information is logged and made available to the web debug toolbar. So the processed time displayed when you browse in the `staging` environment is not representative of what it will be in production, where the debug mode is turned off.

To get a better view of the process time of each request, you should use benchmarking tools, like Apache Bench or JMeter. These tools allow load testing and provide two important pieces of information: the average loading time of a specific page and the maximum capacity of your server. The average loading time data is very useful for monitoring performance improvements due to cache activation.

### Identifying Cache Parts

When the web debug toolbar is enabled, the cached elements are identified in a page with a red frame, each having a cache information box on the top left, as shown in Figure 12-5. The box has a blue background if the element has been executed, or a yellow background if it comes from the cache. Clicking the cache information link displays the identifier of the cache element, its lifetime, and the elapsed time since its last modification. This will help you identify problems when dealing with out-of-context elements, to see when the element was created and which parts of a template you can actually cache.

Figure 12-5 - Identification for cached elements in a page

![Identification for cached elements in a page](http://www.symfony-project.org/images/book/1_4/F1205.png "Identification for cached elements in a page")

HTTP 1.1 and Client-Side Caching
--------------------------------

The HTTP 1.1 protocol defines a bunch of headers that can be of great use to further speed up an application by controlling the browser's cache system.

The HTTP 1.1 specifications of the World Wide Web Consortium (W3C, [http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html)) describe these headers in detail. If an action has caching enabled, and it uses the `with_layout` option, it can use one or more of the mechanisms described in the following sections.

Even if some of the browsers of your website's users may not support HTTP 1.1, there is no risk in using the HTTP 1.1 cache features. A browser receiving headers that it doesn't understand simply ignores them, so you are advised to set up the HTTP 1.1 cache mechanisms.

In addition, HTTP 1.1 headers are also understood by proxies and caching servers. Even if a user's browser doesn't understand HTTP 1.1, there can be a proxy in the route of the request to take advantage of it.

### Adding an ETag Header to Avoid Sending Unchanged Content

When the ETag feature is enabled, the web server adds to the response a special header containing a signature of the response itself.

    ETag: "1A2Z3E4R5T6Y7U"

The user's browser will store this signature, and send it again together with the request the next time it needs the same page. If the new signature shows that the page didn't change since the first request, the server doesn't send the response back. Instead, it just sends a `304: Not modified` header. It saves CPU time (if gzipping is enabled for example) and bandwidth (page transfer) for the server, and time (page transfer) for the client. Overall, pages in a cache with an ETag are even faster to load than pages in a cache without an ETag.

In symfony, you enable the ETag feature for the whole application in `settings.yml`. Here is the default ETag setting:

    all:
      .settings:
        etag: true

For actions in a cache with layout, the response is taken directly from the `cache/` directory, so the process is even faster.

### Adding a Last-Modified Header to Avoid Sending Still Valid Content

When the server sends the response to the browser, it can add a special header to specify when the data contained in the page was last changed:

    Last-Modified: Sat, 23 Nov 2010 13:27:31 GMT

Browsers can understand this header and, when requesting the page again, add an `If-Modified` header accordingly:

    If-Modified-Since: Sat, 23 Nov 2010   13:27:31 GMT

The server can then compare the value kept by the client and the one returned by its application. If they match, the server returns a `304: Not modified` header, saving bandwidth and CPU time, just as with ETags.

In symfony, you can set the `Last-Modified` response header just as you would for another header. For instance, you can use it like this in an action:

    [php]
    $this->getResponse()->setHttpHeader('Last-Modified', $this->getResponse()->getDate($timestamp));

This date can be the actual date of the last update of the data used in the page, given from your database or your file system. The `getDate()` method of the `sfResponse` object converts a timestamp to a formatted date in the format needed for the `Last-Modified` header (RFC1123).

### Adding Vary Headers to Allow Several Cached Versions of a Page

Another HTTP 1.1 header is `Vary`. It defines which parameters a page depends on, and is used by browsers and proxies to build cache keys. For example, if the content of a page depends on cookies, you can set its `Vary` header as follows:

    Vary: Cookie

Most often, it is difficult to enable caching on actions because the page may vary according to the cookie, the user language, or something else. If you don't mind expanding the size of your cache, set the `Vary` header of the response properly. This can be done for the whole application or on a per-action basis, using the `cache.yml` configuration file or the `sfResponse` related method as follows:

    [php]
    $this->getResponse()->addVaryHttpHeader('Cookie');
    $this->getResponse()->addVaryHttpHeader('User-Agent');
    $this->getResponse()->addVaryHttpHeader('Accept-Language');

Symfony will store a different version of the page in the cache for each value of these parameters. This will increase the size of the cache, but whenever the server receives a request matching these headers, the response is taken from the cache instead of being processed. This is a great performance tool for pages that vary only according to request headers.

### Adding a Cache-Control Header to Allow Client-Side Caching

Up to now, even by adding headers, the browser keeps sending requests to the server even if it holds a cached version of the page. You can avoid that by adding `Cache-Control` and `Expires` headers to the response. These headers are disabled by default in PHP, but symfony can override this behavior to avoid unnecessary requests to your server.

As usual, you trigger this behavior by calling a method of the `sfResponse` object. In an action, define the maximum time a page should be cached (in seconds):

    [php]
    $this->getResponse()->addCacheControlHttpHeader('max_age=60');

You can also specify under which conditions a page may be cached, so that the provider's cache does not keep a copy of private data (like bank account numbers):

    [php]
    $this->getResponse()->addCacheControlHttpHeader('private=True');

Using `Cache-Control` HTTP directives, you get the ability to fine-tune the various cache mechanisms between your server and the client's browser. For a detailed review of these directives, see the W3C `Cache-Control` specifications.

One last header can be set through symfony: the `Expires` header:

    [php]
    $this->getResponse()->setHttpHeader('Expires', $this->getResponse()->getDate($timestamp));

>**CAUTION**
>The major consequence of turning on the `Cache-Control` mechanism is that your server logs won't show all the requests issued by the users, but only the ones actually received. If the performance gets better, the apparent popularity of the site may decrease in the statistics.

Summary
-------

The cache system provides variable performance boosts according to the cache type selected. From the best gain to the least, the cache types are as follows:

  * Super cache
  * Action cache with layout
  * Action cache without layout
  * Fragment cache in the template

In addition, partials and components can be cached as well.

If changing data in the model or in the session forces you to erase the cache for the sake of coherence, you can do it with a fine granularity for optimum performance--erase only the elements that have changed, and keep the others.

Remember to test all the pages where caching is enabled with extra care, as new bugs may appear if you cache the wrong elements or if you forget to clear the cache when you update the underlying data. A staging environment, dedicated to cache testing, is of great use for that purpose.

Finally, make the best of the HTTP 1.1 protocol with symfony's advanced cache-tweaking features, which will involve the client in the caching task and provide even more performance gains.
