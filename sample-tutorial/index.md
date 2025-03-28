---
kind: tutorial

title: Sample Tutorial

description: |
  Learn how to use the iximiuz Labs platform to create tutorials that combine markdown,
  interactive playgrounds, dynamic tasks, and rich visual elements to provide a fun and
  effective learning experience.

categories:
  - linux
  - containers

tagz:
  - hello-world
  - example
  - tutorial-docs

createdAt: 2025-03-28
updatedAt: 2025-03-28

cover: __static__/cover.png

# PLAYGROUND CONFIGURATION
#
# Each tutorial can have an accompanying playground.
# The playground object below defines the base playground name
# and the machine, tab, and container registry customizations.
#
# See https://labs.iximiuz.com/api/playgrounds?filter=base for the
# full list of available base playgrounds.
playground:
  # Name of the base playground is the only required attribute.
  name: k3s

  # List available machines. By default, all playground machines are available,
  # but once you define the `machines` attribute, only the explicitly listed
  # subset of machines will be added to the playground.
  machines:
  - name: dev-machine
    # Override default resources (cannot be above the limits of the playground)
    resources:
      cpuCount: 1
      ramSize: "1Gi"
    # users: ...
    # noSSH: ...

  - name: cplane-01
    # Override the default login user and set a custom welcome message.
    users:
    - name: root
      default: true
      # Use a special value '-' to disable the welcome message completely.
      welcome: Welcome to the control plane node.

  - name: node-01
    # Include machine in the playground but disable SSH access to it
    # (automatically disables the terminal UI tab).
    noSSH: true

  # This machine is intentionally not listed, meaning that it won't be available
  # in the content's playground (even though it's part of the base playground).
  # - name: node-02

  # List available tabs. By default IDE, kexp if playground is about Kubernetes,
  # and one terminal tab per machine will be used. However, if the `tabs` attribute
  # is defined, the explicitly listed set of tabs will be used instead.
  tabs:
  - kind: ide            # enable the online IDE (code-server)
  - kind: kexp           # enable the Kubernetes Visualizer (kexp)

  - machine: dev-machine # 'dev-machine' is a hostname in this case, implies 'kind: terminal'
  - machine: cplane-01

  - kind: http-port      # expose a web app listening on machine's port as an iframe tab
    name: Nginx          # user-defined name of the tab
    number: 30080        # port number to expose
    machine: node-01     # machine to expose the port on
    # tls: true          # enable TLS if the target service uses HTTPS

  - kind: web-page       # embed an external web page as an iframe tab
    name: example.com    # user-defined name of the tab
    url: https://example.com

  # Protect the playground's registry (registry.iximiuz.com)
  # with a username and password. Default: no authentication,
  # but the registry is only reachable from the playground.
  registryAuth: user:password


# PLAYGROUND TASKS
#
# The `tasks` attribute is optional. If defined, the listed
# set of tasks will be executed on the tutorial's playground.
#
# There types of tasks currently supported:
# - init: executed once on the playground initialization
# - regular: executed in a loop until they complete (exit with code 0).
# - helper: executed in a loop indefinitely, performing miscellaneous
#           background jobs.
tasks:
  init_deploy_nginx:
    # Init tasks are used to finalize the setup of the playground.
    # Until all init tasks are completed, the playground screen shows
    # a loading animation.
    init: true

    # By default, the task is executed on every machine of the playground.
    # If the task needs to be executed only on a specific machine, you
    # can specify it with the `machine` property. Beware, at the moment,
    # either none or all tasks must have the `machine` property (in the latter case,
    # different tasks may be executed on different machines).
    machine: dev-machine

    # By default, tasks are executed as the `root` user.
    # If the task needs to be executed as a different user, you
    # can specify it with the `user` property.
    user: laborant

    run: |
      kubectl run nginx-01 --image=ghcr.io/iximiuz/labs/nginx:alpine --port=80

  init_expose_nginx:
    init: true
    machine: dev-machine
    # Init tasks run in parallel but before all regular tasks.
    # If the sequence of the init tasks is important,
    # you can specify the `needs` property.
    needs:
      - init_deploy_nginx
    run: |
      kubectl wait --for=condition=ready pod nginx-01

      kubectl expose pod nginx-01 --port=80 --type=NodePort \
        --overrides '{ "apiVersion": "v1","spec":{"ports": [{"port":80,"protocol":"TCP","targetPort":80,"nodePort":30080}]}}'

    # By default, the task will time out after 10 seconds. If you expect a particular
    # check to take longer (or it intentionally should time out faster), you can
    # configure the timeout with the `timeout_seconds` property.
    timeout_seconds: 90

  # Regular tasks are used to check if certain system conditions are met
  # and/or requested user actions are completed. Each regular task should
  # have a visual representation in the content's body, and the total/done
  # number of regular tasks is displayed in the content's header, serving
  # as a main content completion indicator.
  verify_file_exists:
    machine: dev-machine
    # The only mandatory property of a regular task is the `run` script.
    # The task will be executed in a loop until it exits with 0.
    run: |
      if [ ! -f /tmp/some/file.txt ]; then
        echo "This is a diagnostic message. The user won't see it, but it's helpful for debugging."
        exit 1
      fi

  # Tasks can also be used to collect user input. While it's up to the task's
  # author to decide what to do with the input, the input is usually done via
  # some well-known file (the task's UI element and the task's `run` script
  # should agree on the file name beforehand).
  # The task's `run` script can do anything with the provided user input.
  input_container_name:
    machine: dev-machine
    # This task just echoes the user input (but usually, you'd want to assert
    # something about the input).
    run: |
      cat /tmp/container-name.txt

  verify_container_is_running:
    machine: dev-machine
    # Regular tasks run only after all init tasks are completed.
    # By default, regular tasks run in parallel but the execution order
    # can be controlled with the `needs` property.
    needs:
      - input_container_name
    env:
      # A handy way to pass the output of a previous task to the current one
      # is to use the a template variable x(.needs.task_name.stdout|stderr).
      # This example illustrates how to pass the output of the `input_container_name`
      # task to the `verify_container_is_running` task.
      - CONTAINER_NAME=x(.needs.input_container_name.stdout)
    # The task engine also defines a number of higher-level helper functions
    # that can be used in the `run` script. For example, `docker_container_is_running`
    # is a helper function that checks if a container is running.
    run: |
      if ! docker_container_is_running ${CONTAINER_NAME}; then
        echo "The container isn't running."
        exit 1
      fi

  # Ready tasks are executed in a loop until they either complete (exit with 0)
  # or fail (exiting with a non-zero code doesn't mean failure, see failcheck for
  # details). There is a small delay between consecutive executions of the task
  # (currently 1 second, but it's subject to change).
  verify_container_is_stopped:
    machine: dev-machine
    needs:
      - input_container_name
      - verify_container_is_running
    env:
      - CONTAINER_NAME=x(.needs.input_container_name.stdout)
    run: |
      if docker_container_is_running ${CONTAINER_NAME}; then
        echo "The container is still running."
        exit 1
      fi

      if ! docker ps -a | grep -q ${CONTAINER_NAME}; then
        echo "The container is completely gone."
        exit 1
      fi

    # Hintcheck is an optional run-like script that is executed after
    # the task's `run` script. The exit code of the hintcheck script
    # has no effect on the task's outcome. Any output of the hintcheck
    # script (stdout and stderr) gets showed in the task's UI element as
    # a dynamic hint.
    hintcheck: |
      if docker_container_is_running ${CONTAINER_NAME}; then
        echo "The container is still running."
        echo "Run 'docker stop ${CONTAINER_NAME}' to stop it."
      fi

    # Failcheck is an optional run-like script that is executed before
    # the task's `run` instructions to validate that some critical
    # (playground-wide) conditions are still met. If the failcheck exits
    # with a non-zero exit code, the task is considered failed, and the whole
    # playground transitively is marked as failed, too (i.e., the user will
    # have to restart the current content attempt).
    failcheck: |
      if ! docker ps -a | grep -q ${CONTAINER_NAME}; then
        echo "The container has been removed. You shouldn't have done that!"
        echo "Please, restart the tutorial and try again."
        exit 1
      fi

  helper_send_some_traffic:
    machine: node-01
    # Helper tasks are executed in a loop indefinitely, performing miscellaneous
    # background jobs. They are not reflected in the UI and do not block the
    # playground boot, and have no effect on the content's completion.
    # However, they are handy for restoring certain conditions, or creating
    # a continuous load (or trouble) for the system under test.
    helper: true
    run: |
      curl node-01:30080
      sleep 5


# EMBEDDED CHALLENGES
#
# Challenges to "embed" into the tutorial's text using ::card-challenge:: component.
# See ::card-challenge:: components in the markdown below for usage examples.
# Note: every challenge identifier here is exactly the same as
# in the challenge's URL but with - replaced with _ (underscore).
# You can embed any challenges from the https://labs.iximiuz.com/challenges catalog.
challenges:
  docker_101_container_run: {}
  kubernetes_pod_with_faulty_init_sequence: {}
---

This is a self-documenting sample that serves as a guide on how to author **tutorials** on iximiuz Labs.
You can find the source of this document on [GitHub](https://github.com/iximiuz/labs-content-samples).
Feel free to use it as a starting point for your own tutorials.

## What is a Tutorial?

Tutorials are deep dives into DevOps and server-side topics where theory blends with hands-on examples.
You can think of them as a combination of a blog post and a playground.
A tutorial can be just a few paragraphs long or as large as a book (but in the latter case,
please consider splitting it into multiple tutorials or turning it into a [course](/new/course)):

- Tutorials can (and often should) contain code snippets, images, and _interactive tasks_ (see below).
- Authenticated users can start and complete tutorials, with the corresponding progress reflected in their personal dashboard.

## How to Edit the Tutorial

At the moment, iximiuz Labs does not offer a WYSIWYG editor.
Instead, content editing is expected to be done on your local machine,
while the changes are real-time streamed to the page by [`labctl`](https://github.com/iximiuz/labctl),
providing immediate feedback via the traditional _hot-reloading_ experience.

This approach allows you to write content in your favorite markdown editor or even a full-fledged IDE,
keeping the full control over the raw content source and staying out of the way of any AI-powered features of your editor.

::remark-box
---
kind: warning
---
We encourage you to store the source of your tutorials in a git repository,
similarly to how you manage code projects.
This way, you can benefit from the versioning and backup features of git.
::

To get the up-to-date instructions on how to install and use `labctl`,
click "Open in Editor" button in the tutorial's menu:

![Open in Editor](__static__/open-in-editor.png)

## Tutorial Metadata and Front Matter

Every tutorial is represented by a markdown file that begins with a YAML _front matter_ block.
The smallest possible front matter block for a tutorial should contain the following fields:

```yaml
---
kind: tutorial
title: My Awesome Linux Containers Tutorial
description: |
  A deep dive into managing containers with practical, hands-on examples.

categories:
  - linux
  - containers

tagz:
  - example
  - tutorial-docs

createdAt: 2025-03-26

cover: __stаtic__/cover.png
---
```

The above fields are essential because they feed the platform with information used for indexing, navigation, and displaying your tutorial.
However, iximiuz Labs also supports a number of additional fields that can be used to:

- Specify the tutorial's playground and its customization options.
- Define _init_, _helper_, and _regular_ tasks to be executed on the playground.
- Define accompanying the tutorial _challenges_ to later embed them with the `content-card` component.

::details-box
---
:summary: Click here to see the full list of supported front matter fields
---

```yaml
---
kind: tutorial         # fixed

title: <string>        # required
description: <string>  # required

# Up to 2 categories
categories:            # required
  - category-1
  - category-2

# Up to 5 tags
tagz:                  # required
  - tag-1
  - tag-2

createdAt: <string>    # required
updatedAt: <string>    # optional

cover: __stаtic__/path/to/cover.png  # required

playground:            # optional
  name: <string>
  machines: ...
  tabs: ...

tasks:                 # optional
  task_name_1:
    ...
  task_name_2:
    ...

challenges:            # optional
  challenge_name_1: {}
  challenge_name_2: {}
---
```
::

## Tutorial Markdown

TODO:

- Show how to embed images (`![](./path/to/image.png)`, `::image-box` and `::slide-show` components).
- Show how to embed code snippets (including line highlighting and file names).
- Show how to embed a remark box.
- Show how to embed a details box.
- Show how to embed a hint box.
- Show how to embed a challenge using the `::card-challenge` component.

### Additional Rich Markdown Elements

iximiuz Labs extends standard markdown with additional visual and interactive elements:

#### Image Embedding

- Image Box: Use the ::image-box component to embed images with optional captions and defined maximum widths.

::image-box
---
:src: __static__/docker-run.png
:alt: 'Docker run command under the hood: pulling the image, creating a container, attaching to the container, connecting to the network, and finally starting it.'
:max-width: 1000px
---
::

- Slide Show: Use the ::slide-show component to display a series of images (useful for diagrams or multi-step illustrations).

::slide-show
---
slides:
  - image: __static__/builder-pattern-go.png
    alt: "Builder Pattern for a Go application."
  - image: __static__/builder-pattern-nodejs.png
    alt: "Builder Pattern for a Node.js application."
---
::

#### Visual Boxes for Notes and Hints

- Remark Box: Used to display warnings or important notes. It supports different kinds (e.g., warning or error).

::remark-box
---
kind: warning
---
💡 Remember: Always attach a playground if your tutorial involves interactive commands.
::

- Details Box: A collapsible section for additional details, code snippets, or extended explanations.

::details-box
---
:summary: More about container image layers
---
The image layers are used for efficient caching and storage. Each layer represents a set of file changes.
::

-	Hint Box: Provides in-context hints for tasks without revealing too much. These are particularly useful for guiding users through troubleshooting.

::hint-box
---
:summary: Hint 1
---
Try running `docker run --help` to see available flags for running containers in detached mode.
::


## Playground Configuration

With every tutorial, you have the option to attach a remote playground -
allowing users to run commands in a browser-based or SSH-accessible environment.

::remark-box
---
kind: warning
---
Tutorials can be created without a playground attached, but it is recommended to include one to enhance the learning experience.
**Only tutorials with playgrounds can be started and completed by iximiuz Labs users.**
::

At the moment, you can augment a tutorial only with an "official" iximiuz Labs playground.
Click [here](/playgrounds?filter=base) to see a full list of available playgrounds.

```sh
curl -s https://labs.iximiuz.com/api/playgrounds?filter=base | jq -r '.[] | .name'
```

While a tutorial can be authored without any playground configuration, it is highly recommended to attach a playground.
This gives users an interactive environment to execute commands and follow along with the tutorial's practical exercises.

### Minimal Playground Configuration

The most basic playground configuration requires only a name:

```yaml
playground:
  name: docker
```

This minimal configuration creates a default playground with a set of standard behaviors.

### Customizing Machines

If you need to alter the set of machines or customize their behavior, you can define a machines attribute within the playground section. This lets you specify details such as:

- Default User Override: Each machine can have a default user defined. For instance, you might set a specific user who will automatically be logged in:

```yaml
machines:
  - name: docker
    users:
      - name: laborant
        default: true
        welcome: 'Welcome to the jungle!'
```

- Welcome Message: The welcome attribute lets you customize a greeting message that appears when a user logs into that machine.
- noSSH Option: Setting noSSH: true on a machine keeps the machine active in the playground, but prevents users from accessing it via SSH. This is useful when you want to provide a read-only or restricted experience (e.g., for a Kubernetes node where only visual or HTTP interactions are needed).

```yaml
- name: kubernetes
  users:
    - name: root
      default: true
      welcome: '-'
  noSSH: true
```

### Controlling UI Tabs

By default, every machine in the playground is allocated a terminal tab. However, you as the author can customize which tabs are visible. Beyond the terminal, the following types of tabs are available:

- HTTP Port Tab: Expose a service running on a machine by mapping an HTTP port.
- Embedded Web Page Tab: Use an iframe-like component to embed an external webpage directly within the tutorial.
- IDE Tab: Leverage a built-in online IDE (powered by code-server) for a coding environment.
- Kubernetes Visual Explorer Tab: For Kubernetes-specific tutorials, a visual explorer tab (kexp) is available.

For example, a custom playground configuration with tabs might look like this:

```yaml
...
```

### Playground Container Registry

Each playground has a private container registry that is used to store images for the tutorial.
It's a handy way to transfer images between playground VMs, or from a playground VM into a playground's Kubernetes cluster.

To protect the registry, you can specify a username and password:

```yaml
registryAuth: testuser:testpassword
```

## Task Types and Examples

Tasks in iximiuz Labs are used to automate checks and provide dynamic feedback as users follow along.
Here are three key types of tasks you can include in your tutorials:

### Initialization Task

An `init` task runs during playground initialization (before the UI is fully available).
It's vital for setting up the environment but is not reflected in the user interface.
For example:

```yaml
...
```

### Regular Verification Task

A regular task verifies a specific system condition.
It may depend on other tasks and is shown in the UI to provide real-time feedback.

```yaml
...
```

Each regular task should have a corresponding `::simple-task` component in the markdown to provide a visual representation of the task's progress and _dynamic hints_:

```markdown
::simple-task
---
:tasks: tasks
:name: <task-name>
---
#active
<what needs to be done / what system condition needs to be met>

#completed
<what has been done or any other confirmation of completion>
::
```

Here's an example of a simple task UI element:

::simple-task
---
:tasks: tasks
:name: verify_file_exists
---
#active
Waiting for `/tmp/some/file.txt` to be created...

#completed
Nailed it! The file has appeared 🎉
::

### User-Input Task

A _user-input_ task requires the user to submit specific data (for example, reading a value from a log).
The input is validated, and if correct, it's passed to the corresponding task for further processing:

```markdown
::user-input-task
---
:tasks: tasks
:name: <task-name>
:validate: <simple shell command or pipeline>
:destination: <path to a file to store the input>
---
#active
<what needs to be entered by the user>

#completed
<confirmation of completion>
::
```

Here's an example of a user-input task UI element:

::user-input-task
---
:tasks: tasks
:name: input_container_name
:validate: 'echo "x(.input)" | grep "[0-9a-f-]\{2,16\}"'
:destination: /tmp/container-name.txt
---
#active
Enter the name of the future container:

#completed
Nice one! You certainly have a good taste in names 🎉
::

### Helper Task

TODO: ...

```yaml
...
```

## Instead of Conclusion

Happy authoring on iximiuz Labs - make your tutorials engaging, interactive, and deeply informative!
