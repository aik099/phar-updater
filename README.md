PHAR Updater
============

[![Build Status](https://travis-ci.org/padraic/phar-updater.svg?branch=master)](https://travis-ci.org/padraic/phar-updater)

You have a phar file to distribute, and it's all the rage to include a self-update
command. Do you really need to write that? Here at Humbug Central, our army of
minions (all ten of them) have written one for you with the following features
(to which we'll add over time):

* Full support for SSL/TLS verification so your users will not get their downloads
replaced by cheeky people.
* Support for OpenSSL phar signatures.
* Version checking (currently to latest SHA-1).
* Simple API where it either updates or Exceptions will go wild.

Development continues so...give it a whirl and complain loudly in the issues
section if needed.

Installation
============

```json
require: {
   "padraic/phar-updater": "~1.0@dev"
}
```

The package utilises PHP Streams for remote requests so it will require the openssl
extension and the `allow_url_open` setting to both be enabled. Support for curl
will follow in time.

Usage
=====

The default update strategy uses an SHA-1 hash of the current remote phar in a
version file, and will update the local phar when this version is changed. There
is also a Github strategy which tracks Github Releases where you can upload a
new phar file for a release.

### Basic SHA-1 Strategy

Create your self-update command, or even an update command for some other phar
other than the current one, and include this.

```php
/**
 * The simplest usage assumes the currently running phar is to be updated and
 * that it has been signed with a private key (using OpenSSL).
 *
 * The first constructor parameter is the path to a phar if you are not updating
 * the currently running phar.
 */

use Humbug\SelfUpdate\Updater;

$updater = new Updater();
$updater->getStrategy()->setPharUrl('http://example.com/current.phar');
$updater->getStrategy()->setVersionUrl('http://example.com/current.version');
try {
    $result = $updater->update();
    $result ? exit('Updated!') : exit('No update needed!');
} catch (\Exception $e) {
    exit('Well, something happened! Either an oopsie or something involving hackers.');
}
```

If you are not signing the phar using OpenSSL:

```php
/**
 * The second parameter to the constructor must be false if your phars are
 * not signed using OpenSSL.
 */

use Humbug\SelfUpdate\Updater;

$updater = new Updater(null, false);
$updater->getStrategy()->setPharUrl('http://example.com/current.phar');
$updater->getStrategy()->setVersionUrl('http://example.com/current.version');
try {
    $result = $updater->update();
    $result ? exit('Updated!') : exit('No update needed!');
} catch (\Exception $e) {
    exit('Well, something happened! Either an oopsie or something involving hackers.');
}
```

If you need version information:

```php
use Humbug\SelfUpdate\Updater;

$updater = new Updater();
$updater->getStrategy()->setPharUrl('http://example.com/current.phar');
$updater->getStrategy()->setVersionUrl('http://example.com/current.version');
try {
    $result = $updater->update();
    if ($result) {
        $new = $updater->getNewVersion();
        $old = $updater->getOldVersion();
        exit(sprintf(
            'Updated from SHA-1 %s to SHA-1 %s', $old, $new
        ));
    } else {
        exit('No update needed!')
    }
} catch (\Exception $e) {
    exit('Well, something happened! Either an oopsie or something involving hackers.');
}
```

See the Update Strategies section for an overview of how to setup the SHA-1
strategy. It's a simple to maintain choice for development or nightly versions of
phars which are released to a specific numbered version.

### Github Release Strategy

Beyond devleopment or nightly phars, if you are released numbered versions on
Github (i.e. tags), you can upload additional files (such as phars) to include in
the Github Release.

```php
/**
 * Other than somewhat different setters for the strategy, all other operations
 * are identical.
 */

use Humbug\SelfUpdate\Updater;

$updater = new Updater();
$updater->setStrategy(Updater::STRATEGY_GITHUB);
$updater->getStrategy()->setPackageName('myvendor/myapp');
$updater->getStrategy()->setPharName('myapp.phar');
$updater->getStrategy()->setCurrentLocalVersion('v1.0.1');
try {
    $result = $updater->update();
    $result ? exit('Updated!') : exit('No update needed!');
} catch (\Exception $e) {
    exit('Well, something happened! Either an oopsie or something involving hackers.');
}
```

Package name refers to the name used by Packagist, and phar name is the phar's
file name assumed to be constant across versions.

It's left to the implementation to supply the current release version associated
with the local phar. This needs to be stored within the phar and should match
the version string used by Github. This can follow any standard practice with
recognisable pre- and postfixes, e.g.
`v1.0.3`, `1.0.3`, `1.1`, `1.3rc`, `1.3.2pl2`.

If you wish to update to a non-stable version, for example where users want to
update according to a development track, you can set the stability flag for the
Github strategy. By default this is set to `stable` or, in constant form,
`\Humbug\SelfUpdate\Strategy\GithubStrategy::STABLE`:

```php
$updater->getStrategy()->setStability('unstable');
```

### Rollback Support

The Updater automatically copies a backup of the original phar to myname-old.phar.
You can trigger a rollback quite easily using this convention:

```php
use Humbug\SelfUpdate\Updater;

/**
 * Same constructor parameters as you would use for updating. Here, just defaults.
 */
$updater = new Updater();
try {
    $result = $updater->rollback();
    $result ? exit('Success!') : exit('Failure!');
} catch (\Exception $e) {
    exit('Well, something happened! Either an oopsie or something involving hackers.');
}
```

As users may have diverse requirements in naming and locating backups, you can
explicitly manage the precise path to where a backup should be written, or read
from using the `setBackupPath()` function when updating a current phar or the
`setRestorePath()` prior to triggering a rollback. These will be used instead
of the simple built in convention.

### Constructor Parameters

The Updater constructor is fairly simple. The three basic variations are:

```php
/**
 * Default: Update currently running phar which has been signed.
 */
$updater = new Updater;
```

```php
/**
 * Update currently running phar which has NOT been signed.
 */
$updater = new Updater(null, false);
```

```php
/**
 * Use a strategy other than the default SHA Hash.
 */
$updater = new Updater(null, false, Updater::STRATEGY_GITHUB);
```

```php
/**
 * Update a different phar which has NOT been signed.
 */
$updater = new Updater('/path/to/impersonatephil.phar', false);
```

### Avoid Post Update File Includes

Updating a currently running phar is made trickier since, once replaced, attempts
to load files from it within a process originating from an older phar is likely
to create an `internal corruption of phar` error. For example, if you're using
Symfony Console and have created an event dispatcher for your commands, the lazy
loading of some event classes will have this impact.

The solution is to disable or remove the dispatcher for your self-update command.

In general, when writing your self-update CLI commands, either pre-load any classes
likely needed prior to updating, or disable their loading if not essential.

### Custom Update Strategies

All update strategies revolve around checking for updates, and downloading updates.
The actual work behind replacing local files and backups is handled separate.
To create a custom strategy, you can implement `Humbug\SelfUpdate\Strategy\StrategyInterface`
and pass a new instance of your implementation post-construction.

```php
$updater = new Updater(null, false);
$updater->setStrategyObject(new MyStrategy);
```

The similar `setStrategy()` method is solely used to pass flags matching internal
strategies.

Update Strategies
=================

SHA-1 Hash Synchronisation
--------------------------

The phar-updater package only (that will change!) supports an update strategy
where phars are updated according to the SHA-1 hash of the current phar file
available remotely. This assumes the existence of only two to three remote files:

* myname.phar
* myname.version
* myname.phar.pubkey (optional)

The `myname.phar` is the most recently built phar.

The `myname.version` contains the SHA-1 hash of the most recently built phar where
the hash is the very first string (if not the only string). You can generate this
quite easily from bash using:

```sh
sha1sum myname.phar > myname.version
```

Remember to regenerate the version file for each new phar build you want to distribute.
Using `sha1sum` adds additional data after the hash, but it's fine since the hash is
the first string in the file which is the only requirement.

If using OpenSSL signing, which is very much recommended, you can also put the
public key online as `myname.phar.pubkey`, for the initial installation of your
phar. However, please note that phar-updater itself will never download this key, will
never replace this key on your filesystem, and will never install a phar whose
signature cannot be verified by the locally cached public key.

If you need to switch keys for any reason whatsoever, users will need to manually
download a new phar along with the new key. While that sounds extreme, it's generally
not a good idea to allow for arbitrary key changes that occur without user knowledge.
The openssl signing has no mechanism such as a central authority or a browser's trusted
certificate stash with which to automate such key changes in a safe manner.


Github Releases
---------------

When tagging new versions on Github, these are created and hosted as `Github Releases`
which allow you to attach a changelog and additional file downloads. Using this
Github feature allows you to attach new phars to releases, associating them with
a version string that is published on Packagist.

Taking advantage of this architecture, the Github Strategy for updating phars can
compare the existing local phar's version against remote release versions and
update to the most recent stable (or unstable) version from Github.

At present, it's assume that phar files all bear the same name across releases,
i.e. just a plain name like `myapp.phar` without versioning information in the
file name. You can also upload your phar's public key in the same way. Using
the established convention of being the phar name with `.pubkey` appended, e.g.
`myapp.phar` would be matched with `myapp.phar.pubkey`.

You can read more about Github releases [here](https://help.github.com/articles/creating-releases/).

While you can draft a release, Github releases are created automatically whenever
you create a new git tag. If you use git tagging, you can go to the matching
release on Github, click the `Edit` button and attach files. It's recommended to
do this as soon as possible after tagging to limit the window whereby a new
release exists without an updated phar attached.