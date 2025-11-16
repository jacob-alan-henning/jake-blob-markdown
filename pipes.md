# Pipelines are disposable, not infrastructure
![pipes](images/bad-pipes.jpg)

Teams that are afraid of their CI/CD pipelines treat them like infrastructure; something you carefully architect, document, and maintain. That's wrong.
Pipelines are feedback loops. The only way to make them reliable is to break them constantly and learn from it.

## You've got it twisted
When you treat a pipeline like infrastructure, you get

* One person who "owns" it and everyone else is afraid to touch it
* Elaborate planning sessions before changing anything
* Pipelines that work but nobody understands why
* Fear of breaking it, so it calcifies and gets worse over time

Pipeline reliability is not the same as deployment reliability or application reliability. Your pipeline is supposed to fail. That's its job; catch problems before they hit production. A pipeline that never fails either isn't being used or isn't actually testing anything.

## What actually works
* Find the commands to build your application.
* Find the commands to deploy it.
* Find the commands to verify the deploy isn't terrible.
* Pin every version you can.
* Glue it together.
* Now break it.
* Run it against dev in a loop. Let it fail. Fix what breaks. Do it again.
* Add the tests for your application ... you have those right?
* Make people actually use it.
* Find the pain points.
* Fix them.
* Break it again.
* Run as much in parallel as you can.
* Add more environments.
* Add more tests - integration, regression, smoke more than unit.
* Remove stuff nobody looks at.
* When it breaks, don't let it happen again.
* Make it faster.
* Break it some more.
* Add caching.
* Remove stuff that doesn't work anymore - what was good once can become bad.
* Make it cheaper.
* Try something new.
* Break something.
* Fix something.
* More yaml. Less yaml. More environments. Fewer environments.
* Break it again.

## The only real rule
You can't design your way to a good pipeline. You can only iterate your way there.

The tooling barely matters. Pick something standard (GitLab, GitHub Actions, even Jenkins if you're stuck with it) and just start. Laugh out anyone who says they have a better custom solution.

What matters is your willingness to fuck with it. To let it break. To learn from the breaks. To make it disposable enough that you're not afraid to change it.

The more disposable it is, the more fearless you can be with it.

Start gluing basics together. Add what seems useful. Remove what isn't. Repeat until it stops sucking.

The only way to make it reliable is to break it a lot.
