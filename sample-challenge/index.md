---
kind: challenge

title: {{ix-title}}

description: |
  {{ix-description}}

categories:
{{ix-categories}}

tagz:
{{ix-tags}}

difficulty: {{ix-difficulty}}

createdAt: {{ix-created-at}}
updatedAt: {{ix-updated-at}}

cover: __static__/different-ways-to-start-containers.png

playground:
  # See https://labs.iximiuz.com/api/playgrounds for available playgrounds
  name: k3s
  # Protect the playground's registry (registry.iximiuz.com) with a username and password.
  # default: no authentication
  registryAuth: testuser:testpassword

  # List available tabs (by default IDE, kexp, and one terminal tab per machine are used)
  tabs:
  - kind: ide             # enable the online IDE (code-server)
  - kind: kexp            # enable the Kubernetes Visualizer (kexp)
  - kind: web-page        # embed an external web page
    name: Blog
    url: https://iximiuz.com
  - machine: dev-machine  # 'dev-machine' is a hostname in this case, implies 'kind: terminal'
  - machine: cplane-01    # another machine, with the hostname 'cplane-01'
  - number: 80            # expose port 80 on the 'dev-machine' machine as a tab
    machine: dev-machine  # implies 'kind: http-port'

  # List available machines (by default, all playground machines are available)
  machines:
  - name: dev-machine
    users:
    - name: laborant
      default: true
      welcome: 'Welcome to the jungle!'
      # Allow the following identities to log in to the machine as this user.
      # Defaults to ['conductor', '<current-user>']. The special 'conductor'
      # identity is used by the UI to provide access via the web terminal, and
      # the <current-user> is used to allow cross-machine access within the same
      # playground.
      # sshAuthorizedUsers:
      #   - conductor
      #   - laborant
    # Override default resources (cannot be above the limits of the playground)
    resources:
      cpuCount: 1
      ramSize: "1Gi"
  - name: cplane-01
    users:
    - name: root
      default: true
      welcome: '-'  # a hacky way to disable the welcome message
    # Disable SSH access to the machine (automatically disables the terminal tab)
    noSSH: true

tasks:
  # Tasks are executed until they complete (exit with code 0).
  init_task_1:
    # Init tasks are used to finalize the setup of the playground.
    # Until all init tasks are completed, the playground screen shows
    # a loading animation. Init tasks run in parallel but before all
    # regular tasks. If the sequence of the init tasks is important,
    # you can specify the `needs` property.
    init: true
    # By default, the task is executed on every machine of the playground.
    # If the task needs to be executed only on a specific machine, you
    # can specify it with the `machine` property. Beware, at the moment,
    # either none or all tasks must have the `machine` property (in the latter case,
    # different tasks may be executed on different machines).
    # machine: dev-machine

    # By default, tasks are executed as the `root` user.
    # If the task needs to be executed as a different user, you
    # can specify it with the `user` property.
    user: laborant

    run: |
      docker run hello-world

  # Regular tasks are used to check if the challenge is completed.
  # Regular tasks run in parallel but can be serialized with the
  # `needs` property.
  verify_container:
    hintcheck: |
      if [ $(docker_container_count_total) -gt 1 ]; then
        echo "Have you started more than one container?"
        echo "This challenge requires only one container to be running."
        exit 0
      fi

      if [ $(docker_container_count_total) -gt 0 ] && [ $(docker_container_count_running) -eq 0 ]; then
        echo "Seems like there is a container, but it's not running."
        echo "Did it crash, exited too quickly, or have you created but not started it?"
        exit 0
      fi
    run: |
      [ $(docker_container_count_running) -gt 1 ] && echo "Too many running containers" && exit 1
      [ $(docker_container_count_running) -lt 1 ] && echo "No running containers" && exit 1

      sleep 2  # making sure it's stable enough

      [ $(docker_container_count_running) -gt 1 ] && echo "Too many running containers" && exit 1
      [ $(docker_container_count_running) -lt 1 ] && echo "No running containers" && exit 1

      docker ps -q --no-trunc

  # Ready tasks are executed in a loop until they either complete (exit with 0)
  # or fail (exiting with a non-zero code doesn't mean failure, see failcheck for
  # details). There is a small delay between consecutive executions of the task
  # (currently 1 second, but it's subject to change).
  verify_container_id:
    needs:
      - verify_container
    env:
      - CONTAINER_ID=x(.needs.verify_container.stdout)
    # By default, the task will time out after 10 seconds. If you expect a particular
    # check to take longer (or it intentionally should time out faster), you can
    # configure the timeout with the `timeout_seconds` property.
    timeout_seconds: 5
    # `run` is a mandatory part of each task. The solution checker (examiner)
    # will keep executing the task's `run` script until it exits with 0.
    run: |
      PROVIDED_ID="$(cat /tmp/container-id.txt)"
      if [ "${PROVIDED_ID}" == "" ]; then
        echo "Provided container ID is empty"
        exit 1
      fi

      if [[ "${CONTAINER_ID}" != "${PROVIDED_ID}"* ]]; then
        echo "Container ID is not correct"
        exit 1
      fi
    # If defined, the failcheck is executed before the task's `run` instructions.
    # If the failcheck exits with a non-zero exit code, the task is considered failed,
    # and the whole challenge transitively is marked as failed, too (i.e., the user
    # will have to restart the challenge).
    failcheck: |
      if ! docker_container_is_running ${CONTAINER_ID}; then
        echo "The container isn't running anymore. Did it crash?"
        exit 1
      fi
    # Hintcheck is optional. If defined, the solution checker (examiner)
    # will run the hintcheck script after every task's `run` script.
    # The exit code of the hintcheck script has no effect on the task's outcome.
    # Any output of the hintcheck script (stdout and stderr) gets showed in the
    # task's UI element (so-called, dynamic hints).
    hintcheck: |
      if ! docker_container_is_running ${CONTAINER_ID}; then
        echo "To understand what happened, try running 'docker ps -a'."
        echo "It'll show all containers, including non-running ones."
      fi

  verify_container_pid:
    needs:
      - verify_container
    env:
      - CONTAINER_ID=x(.needs.verify_container.stdout)
    failcheck: |
      if ! docker_container_is_running ${CONTAINER_ID}; then
        echo "The container isn't running anymore. Did it crash?"
        exit 1
      fi
    hintcheck: |
      if ! docker_container_is_running ${CONTAINER_ID}; then
        echo "To understand what happened, try running 'docker ps -a'."
        echo "It'll show all containers, including non-running ones."
      fi
    run: |
      PROVIDED_PID="$(cat /tmp/container-pid.txt)"
      if [ "${PROVIDED_PID}" == "" ]; then
        echo "Provided container PID is empty"
        exit 1
      fi

      if [ "${PROVIDED_PID}" != "$(docker_container_pid ${CONTAINER_ID})" ]; then
        echo "Container PID is not correct"
        exit 1
      fi

  verify_container_ip:
    needs:
      - verify_container
    env:
      - CONTAINER_ID=x(.needs.verify_container.stdout)
    failcheck: |
      if ! docker_container_is_running ${CONTAINER_ID}; then
        echo "The container isn't running anymore. Did it crash?"
        exit 1
      fi
    hintcheck: |
      if ! docker_container_is_running ${CONTAINER_ID}; then
        echo "To understand what happened, try running 'docker ps -a'."
        echo "It'll show all containers, including non-running ones."
      fi
    run: |
      PROVIDED_IP="$(cat /tmp/container-ip.txt)"
      if [ "${PROVIDED_IP}" == "" ]; then
        echo "Provided container IP is empty"
        exit 1
      fi

      if [ "${PROVIDED_IP}" != "$(docker_container_ip ${CONTAINER_ID})" ]; then
        echo "Container IP is not correct"
        exit 1
      fi
---

In this challenge, you will need to perform the most fundamental Docker operation - start a container.

::image-box
---
src: __static__/different-ways-to-start-containers.png
alt: 'Different ways to start containers (Docker, Podman, nerdctl, ctr.)'
max-width: 600px
---

<i>Different ways to start containers (Docker, Podman, nerdctl, ctr.)</i>
::

You can use any container image you like,
but we recommend choosing a long-running container (e.g., `nginx`)
because to complete this challenge you will also need to inspect the running container and answer a few questions about it.

::simple-task
---
:tasks: tasks
:name: verify_container
---
#active
Waiting for the container to start...

#completed
Yay! The container is running 🎉
::

::hint-box
---
:summary: Hint 1
---

It's an easy one - `docker --help` is your friend.
::

To keep track of containers, Docker assigns a unique ID to each of them.
Can you find the ID of the container that you've just started?

::user-input-task
---
:tasks: tasks
:name: verify_container_id
:validate: '[[ "x(.needs.verify_container.stdout)" = "x(.input)"* ]]'
:destination: /tmp/container-id.txt
---
#active
Waiting for the container ID to be identified...

#completed
Yay! You've found the running container ID 🎉
::

::hint-box
---
:summary: Hint 2
---

Same advice as in the previous hint - `docker --help` is your friend 😉
::

Now, when you have a running container, let's try to understand what it actually is.

Did you know that [Docker containers are "just" Linux processes](https://iximiuz.com/en/posts/oci-containers/)?
Can you locate the main container's process? What is its PID?

::user-input-task
---
:tasks: tasks
:name: verify_container_pid
:validate: '[ "$(docker_container_pid x(.needs.verify_container.stdout))" = "x(.input)" ]'
:destination: /tmp/container-pid.txt
---
#active
Waiting for the container PID to be identified...

#completed
Yay! You've found the running container PID 🎉
::

::hint-box
---
:summary: Hint 3
---

If containers are _regular Linux processes_, will they show up in `ps` or `top` output? 🤔
::

::hint-box
---
:summary: Hint 4
---

Entered a PID of a process that definitely belongs to the container, but the solution checker doesn't accept it?
One of the key Docker design principles is to run **one service per container**.
However, it doesn't mean that every container will have only one process inside.
Actually, more often than not, you'll find a whole process tree inside a container.
Try identifying the root process of that tree.
That's what the checker expects.
::

::hint-box
---
:summary: Hint 5
---

Still having trouble?
You can always fall back to `docker inspect` to cross-check your findings.
::

Approximating containers to _regular Linux processes_ is helpful, but it's not very accurate.
Thinking of containers as of _boxes for processes_ might be even more helpful at times.

From inside the box, it may look like the containerized app is running on its own machine.
In particular, such a virtualized environment will have its own network interface and an IP address.
Can you find the IP address of the container that you've just started?

::user-input-task
---
:tasks: tasks
:name: verify_container_ip
:validate: '[ "$(docker_container_ip x(.needs.verify_container.stdout))" = "x(.input)" ]'
:destination: /tmp/container-ip.txt
---
#active

Waiting for the container IP address to be identified...

#completed
Yay! You've found the running container IP 🎉
::

::hint-box
---
:summary: Hint 6
---

There are many ways to find the container IP address.
If you're a Linux guru, you can try your luck with `ip netns` and `ip addr`.
::

::hint-box
---
:summary: Hint 7
---

Don't feel comfortable messing with Linux network namespaces?
You can always use the `docker inspect` command to find the container IP address.
::
