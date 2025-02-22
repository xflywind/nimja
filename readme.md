Nimja Template Engine
=====================

<p align="center">
  <img width="460" src="logo/logojanina.png">
</p>


typed and compiled template engine inspired by [jinja2](https://jinja.palletsprojects.com/), [twig](https://twig.symfony.com/) and [onionhammer/nim-templates](https://github.com/onionhammer/nim-templates) for Nim.


FEATURES
========

- compiled
- statically typed
- extends (a master template)
- control structures (if elif else / for / while)
- import other templates
- most nim code is valid in the templates


MOTIVATING EXAMPLE
==================


server.nim

```nim
## compile this example with: --experimental:vmopsDanger or you will get
## Error: cannot 'importc' variable at compile time; getCurrentDirectoryW

import asynchttpserver, asyncdispatch
import ../src/parser
import os, random # os and random are later used in the templates, so imported here

type
  User = object
    name: string
    lastname: string
    age: int

proc renderIndex(title: string, users: seq[User]): string =
  ## the `index.nwt` template is transformed to nim code.
  ## so it can access all variables like `title` and `users`
  ## the return variable could be `string` or `Rope` or
  ## anything which has a `&=`(obj: YourObj, str: string) proc.
  compileTemplateFile(getCurrentDir() / "index.nwt")

proc main {.async.} =
  var server = newAsyncHttpServer()

  proc cb(req: Request) {.async.} =

    # in the templates we can later loop trough this sequence
    let users: seq[User] = @[
      User(name: "Katja", lastname: "Kopylevych", age: 32),
      User(name: "David", lastname: "Krause", age: 32),
    ]
    await req.respond(Http200, renderIndex("index", users))

  server.listen Port(8080)
  while true:
    if server.shouldAcceptRequest():
      await server.acceptRequest(cb)
    else:
      poll()

asyncCheck main()
runForever()
```

index.nwt:

```twig
{% extends master.nwt%}
{#
  extends uses the master.nwt template as the "base".
  All the `block`s that are defined in the master.nwt are filled
  with blocks from this template.

  If the templates extends another, all content HAVE TO be in a block.

  blocks can have arbitrary names

  currently the extends must be on the FIRST LINE!
#}


{% block content %}
  {# A random loop to show off. #}
  {# Data is defined here for demo purpose, but could come frome database etc.. #}
  <h1>Random links</h1>
  {% const links = [
    (title: "google", target: "https://google.de"),
    (title: "fefe", target: "https://blog.fefe.de")]
  %}
  {% for (ii, item) in links.pairs() %}
    {{ii}} <a href="{{item.target}}">This is a link to: {{item.title}}</a><br>
  {% endfor %}

  <h1>Members</h1>
    {# `users` was a param to the `renderIndex` proc #}
    {% for (idx, user) in users.pairs %}
        <a href="/users/{{idx}}">{% importnwt "./partials/_user.nwt" %}</a><br>
    {% endfor %}
{% endblock %}

{% block footer %}
  {#
    we can call arbitraty nim code in the templates.
    Here we pick a random user from users.
  #}
  {% var user = users.sample() %}

  {#
    imported templates have access to all variables declared in the parent.
    So `user` is usable in "./partials/user.nwt"
  #}
  This INDEX was presented by.... {% importnwt "./partials/_user.nwt" %}
{% endblock footer %} {# the 'footer' in endblock is completely optional #}
```

master.nwt
```twig
{#

  This template is later expanded from the index.nwt template.
  All blocks are filled by the blocks from index.nwt

  Variables are also useable.
 #}
<html>
<head>
  <title>{{title}}</title>
</head>
<body>

<style>
body {
  background-color: aqua;
  color: red;
}
</style>

{# The master can declare a variable that is later visible in the child template #}
{% var aVarFromMaster = "aVarFromMaster" %}

{# We import templates to keep the master small #}
{% importnwt "partials/_menu.nwt" %}

<h1>{{title}}</h1>

{# This block is filled from the child templates #}
{%block content%}{%endblock%}


{#
  If the block contains content and is NOT overwritten later.
  The content from the master is rendered
  (does not work in the alpha version..)
#}
{% block onlyMasterBlock %}Only Master Block (does it work yet?){% endblock %}

<footer>
  {% block footer %}{% endblock %}
</footer>

</body>
</html>
```

Basic Syntax
============

- `{{ myObj.myVar }}` --transformed-to--->  `$(myObj.myVar)`
- {% myExpression.inc() %} --transformed-to---> `myExpression.inc()`
- {# a comment #}


BUILD WITH:
--experimental:vmopsDanger


How?
====

nimja transforms templates to nim code on compilation,
so you can write arbitrary nim code.
```nim
proc foo(ss: string, ii: int): string =
  compileTemplateStr(
    """example{% if ii == 1%}{{ss}}{%endif%}{% var myvar = 1 %}{% myvar.inc %}"""
  )
```
is transformed to:

```nim
proc foo(ss: string; ii: int): string =
  result &= "example"
  if ii == 1:
    result &= "one"
  var myvar = 1
  inc(myvar, 1)
```

this means you have the full power of nim in your templates.


USAGE
=====

there are only two relevant procedures:

- `compileTemplateStr(str: string)`
  compiles a template string to nim ast (THIS ONE IS NOT FULLY USABLE IN THIS ALPHA VERSION)
- `compileTemplateFile(path: string)`
  compiles the content of a file to nim ast


if / elif / else
-----------------

```twig
{% if aa == 1 %}
  aa is: one
{% elif aa == 2 %}
  aa is: two
{% else %}
  aa is something else
{% endif %}
```

for
---

```twig
{% for (cnt, elem) in @["foo", "baa", "baz"].pairs() %}
  {{cnt}} -> {{elem}}
{% endfor %}
```

while
----

```twig
{% while isTrue() %}
  still true
{% endwhile %}
```

```twig
{% var idx = 0 %}
{% while idx < 10 %}
  still true
  {% idx.inc %}
{% endwhile %}
```

comments
-------

```twig
{# single line comment #}
{#
  multi
  line
  comment
#}
{# {% var idx = 0 %} #}
```

"to string" / output
--------------------

declare your own `$` before you call
`compileTemplateStr()` or `compileTemplateFile()`
for your custom objects.
```twig
{{myVar}}
{{someProc()}}
```

importnwt
---------

import the content of another template.
The imported template has access to the parents variables.
So it's a valid strategy to have a "partial" template that for example
can render an object or a defined type.
Then include the template wherever you need it:

best practice is to have a partials folder.
and every partial template begins with an underscore "_"
all templates are partial that do not extend another
template and therefore can be included.

partials/_user.nwt:
```twig
<div class="col-3">
  <h2>{{user.name}}</h2>
  <ul>
    <li>Age: {{user.age}}</li>
    <li>Lastname: {{user.lastname}}</li>
  </ul>
</div>
```

partials/_users.nwt:
```twig
<div class="row">
  {% for user in users: %}
    {% importnwt "partials/_user.nwt" %}
  {% endfor %}
</div>
```

extends
-------

a child template can extend a master template.
So that placeholder blocks in the master are filled
with content from the child.


partials/_master.nwt
```twig
<html>
<body>
A lot of boilerplate
{% block content %}{% endblock %}
<hr>
{% block footer %}{% endblock %}
</body>
</html>
```

child.nwt
```
{% extends "partials/_master.nwt" %}
{% block content %}I AM CONTENT{% endblock %}
{% block footer %}...The footer..{% endblock %}
```

if the child.nwt is compiled then rendered like so:

```nim
proc renderChild(): string =
  compileTemplateFile("child.nwt")

echo renderChild()
```

output:
```html
<html>
<body>
A lot of boilerplate
I AM CONTENT
<hr>
...The footer..
</body>
</html>
```

Compile / Use
=============

This is a COMPILED template engine.
This means you must _recompile_ your application
for every change you do in the templates!

```bash
nim c -r --experimental:vmopsDanger yourfile.nim
```

to avoid writing `--experimental:vmopsDanger` every time you compile, create a file:

yourfile.nims
```nim
switch("experimental", "vmopsDanger")
```

sometimes, nim does not catch changes to template files.
Then compile with "-f" (force)

```bash
nim c -f -r --experimental:vmopsDanger yourfile.nim
```
