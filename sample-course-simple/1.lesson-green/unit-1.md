---
kind: unit

title: Let's play with containers

name: docker-intro
---

Hey hey! Welcome to the very first Docker lesson.

How about running a good old nginx container? Execute the following command in the
terminal tab on the right:

```sh
docker run --name cont-1 nginx
```

::simple-task
---
:tasks: tasks
:name: run_container
---
#active
Waiting for nginx container to start...

#completed
The nginx container is up and running. Well done!
::

Ok, great! Let's see how we can stop the container we just started. Open another
terminal tab and run:

```sh
docker stop cont-1
```

::simple-task
---
:tasks: tasks
:name: stop_container
---
#active
Waiting for nginx container to stop...

#completed
Congrats! You successfully stopped the nginx container.
::

Nice! But stopping container doesn't necessary mean its resources were freed. Let's
do a little cleanup:

```sh
docker rm cont-1
```

::simple-task
---
:tasks: tasks
:name: remove_container
---
#active
Waiting for nginx container to be removed...

#completed
Yay! No more nginx containers!
::

Great job!
