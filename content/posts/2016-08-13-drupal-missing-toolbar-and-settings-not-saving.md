+++
author = "hannibal"
categories = ["Drupal", "PHP"]
date = "2016-08-13"
title = "Drupal missing ToolBar and settings not saving"
url = "/2016/08/13/drupal-missing-toolbar-and-settings-not-saving"

+++

Hi folks.

Quick gotcha, when working with Drupal. If you just freshly installed it, and everything seems to work fine, and yet you are experiencing things like, the admin toolbar is randomly disappearing, or configuration is not saved; than you might not have modrewrite enabled on your apache server.

Because, by default, Drupal has clean url enabled, that needs URL rewriting on apache.

So, step one.

Have this in your .htaccess file:
~~~bash
<IfModule mod_rewrite.c>
  RewriteEngine on
  ... # and than a bunch of rewrite rules according to your leisure
~~~

Than look up this line in your httpd.conf file and remove the prefix '#'.
~~~bash
#LoadModule rewrite_module libexec/apache2/mod_rewrite.so
~~~

That is all. From there on, everything should work. If, you don't want the clean url setting, yet you can't disable it, and don't want to restart the server and edit the settings.php file; use drush like this:

~~~bash
drush vset clean_url 0 --yes
~~~

This should disable it and bust the cache in the process so it's immediately visible.

That is all folks.

Cheers,
Gergely.
