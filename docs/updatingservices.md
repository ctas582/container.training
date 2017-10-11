# Updating services

- We want to make changes to the web UI

- The process is as follows:

  - edit code

  - build new image

  - ship new image

  - run new image

---

class: extra-details

## But first...

- Restart the workers

.exercise[

- Just scale back to 10 replicas:
  ```bash
  docker service update dockercoins_worker --replicas 10
  ```

- Check that they're running:
  ```bash
  docker service ps dockercoins_worker
  ```

]

---

## Updating a single service the hard way

- To update a single service, we could do the following:
  ```bash
  REGISTRY=localhost:5000 TAG=v0.2
  IMAGE=$REGISTRY/dockercoins_webui:$TAG
  docker build -t $IMAGE webui/
  docker push $IMAGE
  docker service update dockercoins_webui --image $IMAGE
  ```

- Make sure to tag properly your images: update the `TAG` at each iteration

  (When you check which images are running, you want these tags to be uniquely identifiable)

---

## Updating services the easy way

- With the Compose inbtegration, all we have to do is:
  ```bash
  export TAG=v0.2
  docker-compose -f composefile.yml build
  docker-compose -f composefile.yml push
  docker stack deploy -c composefile.yml nameofstack
  ```

--

- That's exactly what we used earlier to deploy the app

- We don't need to learn new commands!

---

## Updating the web UI

- Let's make the numbers on the Y axis bigger!

.exercise[

- Edit the file `webui/files/index.html`

- Locate the `font-size` CSS attribute and increase it (at least double it)

- Save and exit

- Build, ship, and run:
  ```bash
  export TAG=v0.2
  docker-compose -f dockercoins.yml build
  docker-compose -f dockercoins.yml push
  docker stack deploy -c dockercoins.yml dockercoins
  ```

]

---

## Viewing our changes

- Wait at least 10 seconds (for the new version to be deployed)

- Then reload the web UI

- Or just mash "reload" frantically

- ... Eventually the legend on the left will be bigger!

---

## Making changes

.exercise[

- Edit `~/orchestration-workshop/dockercoins/worker/worker.py`

- Locate the line that has a `sleep` instruction

- Increase the `sleep` from `0.1` to `1.0`

- Save your changes and exit

]

---

## Rolling updates

- Let's change a scaled service: `worker`

.exercise[

- Edit `worker/worker.py`

- Locate the `sleep` instruction and change the delay

- Build, ship, and run our changes:
  ```bash
  export TAG=v0.3
  docker-compose -f dockercoins.yml build
  docker-compose -f dockercoins.yml push
  docker stack deploy -c dockercoins.yml dockercoins
  ```

]

---

## Viewing our update as it rolls out

.exercise[

- Check the status of the `dockercoins_worker` service:
  ```bash
  watch docker service ps dockercoins_worker
  ```

- Hide the tasks that are shutdown:
  ```bash
  watch -n1 "docker service ps dockercoins_worker | grep -v Shutdown.*Shutdown"
  ```

]

If you had stopped the workers earlier, this will automatically restart them.

By default, SwarmKit does a rolling upgrade, one instance at a time.

We should therefore see the workers being updated one my one.

---

## Changing the upgrade policy

- We can set upgrade parallelism (how many instances to update at the same time)

- And upgrade delay (how long to wait between two batches of instances)

.exercise[

- Change the parallelism to 2 and the delay to 5 seconds:
  ```bash
    docker service update dockercoins_worker \
      --update-parallelism 2 --update-delay 5s
  ```

]

The current upgrade will continue at a faster pace.

---

## Changing the policy in the Compose file

- The policy can also be updated in the Compose file

- This is done by adding an `update_config` key under the `deploy` key:

  ```yaml
    deploy:
      replicas: 10
      update_config:
        parallelism: 2
        delay: 10s
  ```

---

## Rolling back

- At any time (e.g. before the upgrade is complete), we can rollback:

  - by editing the Compose file and redeploying;

  - or with the special `--rollback` flag

.exercise[

- Try to rollback the service:
  ```bash
  docker service update dockercoins_worker --rollback
  ```

]

What happens with the web UI graph?

---

## The fine print with rollback

- Rollback reverts to the previous service definition

- If we visualize successive updates as a stack:

  - it doesn't "pop" the latest update

  - it "pushes" a copy of the previous update on top

  - ergo, rolling back twice does nothing

- "Service definition" includes rollout cadence

- Each `docker service update` command = a new service definition

---

class: extra-details

## Timeline of an upgrade

- SwarmKit will upgrade N instances at a time
  <br/>(following the `update-parallelism` parameter)

- New tasks are created, and their desired state is set to `Ready`
  <br/>.small[(this pulls the image if necessary, ensures resource availability, creates the container ... without starting it)]

- If the new tasks fail to get to `Ready` state, go back to the previous step
  <br/>.small[(SwarmKit will try again and again, until the situation is addressed or desired state is updated)]

- When the new tasks are `Ready`, it sets the old tasks desired state to `Shutdown`

- When the old tasks are `Shutdown`, it starts the new tasks

- Then it waits for the `update-delay`, and continues with the next batch of instances