---
kind: lesson

title: 'Lesson 1: Docker intro'
description: 'The very first lesson of the course.'

createdAt: 2024-01-01
updatedAt: 2024-01-01

name: green
slug: lesson-1

cover: __static__/cover.png

playground:
  name: docker

tasks:
  run_container:
    run: |
      [[ $(docker ps -aq --filter "ancestor=nginx") ]] && docker ps -aq --filter "ancestor=nginx"

  stop_container:
    needs:
    - run_container
    run: |
      [[ "exited" = "$(docker container inspect --format='{{.State.Status}}' x(.needs.run_container.stdout))" ]]

  remove_container:
    needs:
    - run_container
    - stop_container
    run: |
      [[ "" = "$(docker ps -aq --filter 'id=x(.needs.run_container.stdout)')" ]]
---
