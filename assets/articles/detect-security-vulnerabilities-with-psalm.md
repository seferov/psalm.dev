<!--
  title: Detect PHP security vulnerabilities with Psalm
  date: 2020-06-23 07:20:00
  author: Matt Brown
  author_link: https://twitter.com/mattbrowndev
-->

Security vulnerabilities are often pretty hard to spot manually. While a null-pointer error can make itself known very quickly, you can execute code for a decade without noticing it has a serious vulnerability.

But just as static type-checking helps developers find bugs in their code, a lot of security vulnerabilities can also be discovered statically through a technique called _taint analysis_. Taint analysis attempts to find connections between user-controlled input (like `$_GET['name']`) and places that we don’t want unescaped user-controlled input to end up (like `<h1><?=$name?></h1>`) by examining how data flows through your application.

There are a couple of commercial tools that perform taint analysis for PHP. We tried one at Vimeo a couple of years ago but the results were disappointing, as none of the reported issues were actually exploitable. While the tool was looking for the right sorts of things (SQL injection, cross-site-scripting vulnerabilities etc.) a lot of the false-positives were the result of poor type inference — something that Psalm is pretty good at.

I started work on Psalm’s taint analysis engine last year, trialling the feature on Vimeo’s codebase (where it discovered an embarassingly-large number of exploitable vulnerabilities) and now it’s ready for everyone else to use.

Here are two example vulnerabilities that it spots:

**Cross-site-scripting (XSS)**

```php
<?php // --taint-analysis

function getName() : string {
    return $_GET['name'] ?? 'unknown';
}

function sayHello() : string {
    return 'Hello ' . getName();
}
?>
<!-- wrap call in htmlentities() to fix -->
<h1><?= sayHello() ?></h1>
```

**SQL injection**

```php
<?php // --taint-analysis

/** @psalm-immutable */
class User {
    public $id;
    public function __construct($userId) {
        $this->id = $userId;
    }
}

class UserUpdater {
    public static function deleteUser(PDO $pdo, User $user) : void {
        $pdo->exec("delete from users where user_id = " . $user->id);
    }
}

$userObj = new User($_GET["user_id"]);

// remove the next line to fix issue
UserUpdater::deleteUser(new PDO(), $userObj);
```

## How does it work?

When you run Psalm with `--taint-analysis` it slowly builds up a graph of data flows in your application due to assignments, property/array access, function calls and more.

Once it has mapped out all potential connections, Psalm attempts to identify problematic paths between user-controlled input and statements like `echo`.

### Taint sources

In taint analysis, user-controlled input is called a _taint source_.

Example sources:
 - `$_GET`
 - `$_POST`
 - `$_COOKIE`

You can also [define your own taint sources](https://psalm.dev/docs/security_analysis/custom_taint_sources) with annotations and/or Psalm plugins.

### Taint sinks

There are some places we don’t want that tainted data to end up – we call those places _taint sinks_.

Example sinks:
 - `<div id="section_<?= $id ?>">`
 - `$pdo->exec("select * from users where name='" . $name . "'");`
 - `header('Location: ' . $location);`

As above, [you can define your own taint sinks](https://psalm.dev/docs/security_analysis/custom_taint_sinks) with Psalm annotations.

## Configuration is key

Psalm’s regular bug detection mode is akin to an unflashy car with a full tank of petrol – you can rely on it to get you from A to B, as long as the roads are well-paved.

Psalm’s security analysis, on the other hand, is like an off-road vehicle with half a tank of petrol: it’ll get you to places regular Psalm can‘t reach, but to make the most of it you’ll need to give it quite a bit of extra fuel (in the form of annotations and, if necessary, plugin code). If you put in the effort, you’ll hopefully get some really useful results.

If you want to find out more about Psalm’s taint analysis, including how to avoid false-positives, [read more in the docs](/docs/security_analysis)!
