# **GitLab CI/CD using Docker on Ubuntu**

CI/CD stands for Continuous Integration/Continuous Deployment

## Step 1 -\> Creation of Project and Dockerfile

1. Before using Ci/CD create a GitLab project first by following the steps
    present in "GitLab initialization" file.

2. Since we are using docker for CI/CD, let us install docker first on the
    server:

### **Install Docker on Ubuntu**

install a few prerequisite packages which let apt use packages over HTTPS:

> `sudo apt install apt-transport-https ca-certificates curl software-properties-common`

Then add the GPG key for the official Docker repository to your system:

> `curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add –`

Add the Docker repository to APT sources:

> `sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"`

Next, update the package database with the Docker packages from the newly added repo:

> `sudo apt update`

Make sure you are about to install from the Docker repo instead of the default Ubuntu repo:

> `apt-cache policy docker-ce`

You’ll see output like this, although the version number for Docker may be different:

```bash
Output of apt-cache policy docker-ce
docker-ce:
  Installed: (none)
  Candidate: 18.03.1~ce~3-0~ubuntu
  Version table:
     18.03.1~ce~3-0~ubuntu 500
        500 https://download.docker.com/linux/ubuntu bionic/stable amd64 Packages 
```

Notice that docker-ce is not installed, but the candidate for installation is from the Docker repository for Ubuntu 18.04 (bionic).

Finally, install Docker:

> `sudo apt install docker-ce`

Docker should now be installed, the daemon started, and the process enabled to start on boot. Check that it’s running:

> `sudo systemctl status docker`

The output should be similar to the following, showing that the service is active and running:

```bash
Output
● docker.service - Docker Application Container Engine
   Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2018-07-05 15:08:39 UTC; 2min 55s ago
     Docs: https://docs.docker.com
 Main PID: 10096 (dockerd)
    Tasks: 16
   CGroup: /system.slice/docker.service
           ├─10096 /usr/bin/dockerd -H fd://
           └─10113 docker-containerd --config /var/run/docker/containerd/containerd.toml
```

1. Create a _Dockerfile_ in the project by selecting the new file option in the
    plus icon present in the ‘Project Overview -\> Details’ screen. In the file
    type

```docker
# Set the base image to node:12-alpine
FROM node:12-alpine as build
# The FROM keyword represents a build stage
# Specify where our app will live in the container
WORKDIR /app

# Copy the React App to the container
COPY . /app/

# Prepare the container for building React
RUN npm install # RUN command is used to run linux/cmd lines.

# We want the production version
RUN npm run build

# Prepare nginx
FROM nginx:1.16.0-alpine # The –from option is used to link previous stage to the current stage.
COPY --from=build /app/build /usr/share/nginx/html
RUN rm /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/conf.d

# Fire up nginx
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"] //Daemon off is used to make sure that the container doesn’t exit on turning on nginx (Solve clash issues)
```

2. Click Commit changes at the bottom of the page to create the file. In the
    next step, you’ll configure a _GitLab runner_ to keep control of who gets to
    execute the deployment job.
3. Create new file by following ‘3.’ And name the file as nginx.conf and configure the file based on your preferences. A beginner’s guide would be found in <http://nginx.org/en/docs/beginners_guide.html>

## Step 2 -\> Configuring the GitLab Runner

> The System on which you want to perform CD/CI needs to be registered in
> GitLab as a _GitLab runner._ The runner is accessed from GitLab via SSH, so
> we should store SSH related data inside GitLab. Note that root permissions
> are required for the instructions to run, since it is automated. The
> following steps are needed to be done to fully configure a runner on a
> Ubuntu system.

1. Download the GitLab runner Package. For example, for Debian or Ubuntu

```bash
# Replace ${arch} with any of the supported architectures, e.g. amd64, arm, arm64
# A full list of architectures can be found here https://gitlab-runner-downloads.s3.amazonaws.com/latest/index.html
curl -LJO
```

2. Install the package for your system as follows. For example, for Debian or
    Ubuntu:

> `dpkg -i gitlab-runner_<arch>.deb`

3. Verify the installation by checking the service status:

> `systemctl status gitlab-runner`

You will have active (running) in the output:

```bash
Output
● gitlab-runner.service - GitLab Runner
   Loaded: loaded (/etc/systemd/system/gitlab-runner.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2020-06-01 09:01:49 UTC; 4s ago
 Main PID: 16653 (gitlab-runner)
    Tasks: 6 (limit: 1152)
   CGroup: /system.slice/gitlab-runner.service
           └─16653 /usr/lib/gitlab-runner/gitlab-runner run --working-directory /home/gitlab-runner --config /
```

4. `After installing GitLab runner on the system. We need to register it to the GitLab account. Follow the steps below.`

    To register the runner, you need to get the project token and the GitLab
    URL:

5. In your GitLab project, navigate to **Settings** \> **CI/CD** \>
    **Runners**.

6. In the **Set up a specific Runner manually** section, you’ll find the
    **registration token** and the GitLab URL. Copy both to a text editor;
    you’ll need them for the next command. They will be referred to as
    <https://your_gitlab.com> and project_token.

7. In the terminal write the following command.

> `sudo gitlab-runner register -n --url https://your_gitlab.com \--registration-token project_token --executor docker --description "Deployment Runner" --docker-image "docker:stable" --tag-list tag_name \--docker-privileged`

The command options can be interpreted as follows:

- \-n executes the register command non-interactively (we specify all
  parameters as command options).

- \--url -\> is the GitLab URL you copied from the runners page in GitLab.

- \--registration-token -\> is the token you copied from the runners page in
  GitLab.

- \--executor -\> is the executor type. docker executes each CI/CD job in a
  Docker container (see [GitLab’s documentation on
  executors](https://docs.gitlab.com/runner/executors/)).

- \--description -\> is the runner’s description, which will show up in GitLab.

- \--docker-image -\> is the default Docker image to use in CI/CD jobs, if not
  explicitly specified.

- \--tag-list -\> is a list of tags assigned to the runner. Tags can be used in
  a pipeline configuration to select specific runners for a CI/CD job. The
  deployment tag will allow you to refer to this specific runner to execute
  the deployment job.

- \--docker-privileged -\> executes the Docker container created for each CI/CD
  job in privileged mode. A privileged container has access to all devices on
  the host machine and has nearly the same access to the host as processes
  running outside containers (see Docker’s documentation about [runtime
  privilege and Linux
  capabilities](https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities)).
  The reason for running in privileged mode is so you can use Docker-in-Docker
  (dind) to build a Docker image in your CI/CD pipeline. It is good practice
  to give a container the minimum requirements it needs. For you it is a
  requirement to run in privileged mode in order to use Docker-in-Docker. Be
  aware, you registered the runner for this specific project only, where you
  are in control of the commands being executed in the privileged container.

  After executing the gitlab-runner register command, you will receive the
  following output:

  Output:

  Runner registered successfully. Feel free to start it, but if it's running
  already the config should be automatically reloaded!

8. Verify the registration process by going to **Settings** \> **CI/CD** \>
    **Runners** in GitLab, where the registered runner will show up.

9. Do another checkup on the GitLab runner config file, by reading and changing
    it using `vi`, which is present at `/etc/gitlab-runner/config.toml. check whether privileged is set to true.`

## Step 3 -\> Configure user for deployment

1. On your server, create a new user which is responsible for handling CD/CI
    request using command :

> `sudo adduser user_name`  
You’ll be guided through the user creation process. Enter a strong password and optionally any further user information you want to specify. Finally confirm the user creation with Y.

2. Add the user to the Docker group:

> `sudo usermod -aG docker user_name`  
This permits **user** to execute the docker command, which is required to perform the deployment.  
**Warning:** Adding a user to the Docker group, [grants privileges equivalent to the root user](https://docs.docker.com/engine/install/linux-postinstall/). For more details on how this has an impact on security in your system, see [Docker Daemon Attack Surface](https://docs.docker.com/engine/security/#docker-daemon-attack-surface).

## Step 4 -\> Set up SSH Key for User

1\. switching to the newly created **user_name** user for whom you’ll generate
the SSH key. You’ll be prompted for the **user_name** password to complete the
user switch.

> `su user_name`

2\. Next, generate a 4096-bit SSH key. It is important to answer the questions of
the ssh-keygen command correctly:

- First question: answer it with ENTER, which stores the key in the default
    location (the rest of this tutorial assumes the key is stored in the default
    location).

- Second question: configures a password to protect the SSH private key (the
    key used for authentication). If you specify a passphrase, you’ll have to
    enter it each time the private key is used. In general, a passphrase adds
    another security layer to SSH keys, which is good practice. Somebody in
    possession of the private key would also require the passphrase to use the
    key. For the purposes of this tutorial, it is important that you have an
    empty passphrase, because the CI/CD pipeline will execute non-interactively
    and therefore does not allow to enter a passphrase.

To summarize, run the following command and confirm both questions with ENTER to
create a 4096-bit SSH key and store it in the default location with an empty
passphrase:

> `ssh-keygen -b 4096`

To authorize the SSH key for the **user_name** user, you need to append the
public key to the authorized_keys file:

> `cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys`  

\~ is short for the user home in Linux. The cat program will print the contents
of a file; here you use the \>\> operator to redirect the output of cat and append it to the authorized_keys file.

## Step 5 -\> Storing private key data in GitLab CD/CI

You are going to store the SSH private key in a GitLab CI/CD file variable, so that the pipeline can make use of the key to log in to the server.

When GitLab creates a CI/CD pipeline, it will send all variables to the corresponding runner and the variables will be set as environment variables for the duration of the job. In particular, the values of **file** variables are stored in a file and the environment variable will contain the path to this
file.

While you’re in the variables section, you’ll also add a variable for the server
IP and the server user, which will inform the pipeline about the destination server and user to log in.

1. Start by showing the SSH private key:

> `cat ~/.ssh/id_rsa`

2. Copy the output to your clipboard. Make sure to add a linebreak after
    \-----END RSA PRIVATE KEY-----:

```bash
~/.ssh/id_rsa
-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY----- 
```

3. Now navigate to **Settings** \> **CI / CD** \> **Variables** in your GitLab
    project and click **Add Variable**. Fill out the form as follows:

- Key: PRIVATE_KEY
- Value: Paste your SSH private key from your clipboard (including a line
  break at the end).
- Type: **File**
- Environment Scope: **All (default)**
- Protect variable: **Checked**
- Mask variable: **Unchecked**

> **Note:** The variable can’t be masked because it does not meet the regular
> expression requirements (see [GitLab’s documentation about masked
> variables](https://gitlab.com/help/ci/variables/README#masked-variables)).
> However, the private key will never appear in the console log, which makes
> masking it obsolete.

4. A file containing the private key will be created on the runner for each
    CI/CD job and its path will be stored in the \$PRIVATE_KEY environment
    variable.

5. Create another variable with your server IP. Click **Add Variable** and fill
    out the form as follows:

- Key: SERVER_IP
- Value: your_server_IP
- Type: **Variable**
- Environment scope: **All (default)**
- Protect variable: **Checked**
- Mask variable: **Checked**

6. Finally, create a variable with the login user. Click **Add Variable** and
    fill out the form as follows:

- Key: SERVER_USER
- Value: deployer
- Type: **Variable**
- Environment scope: **All (default)**
- Protect variable: **Checked**
- Mask variable: **Checked**

## Step 6 -\> Configuring the .gitlab-ci.yml file

**YAML/YML** is a [human-readable](https://en.wikipedia.org/wiki/Human-readable) [data-serialization language](https://en.wikipedia.org/wiki/Serialization). It is commonly used for [configuration files](https://en.wikipedia.org/wiki/Configuration_file) and in applications where data is being stored or transmitted similar to XML.

You are going to configure the GitLab CI/CD pipeline. The pipeline will build a Docker image and push it to the container registry. GitLab provides a container registry for each project. You can explore the container registry by going to **Packages & Registries** \> **Container Registry** in your GitLab project (read more in [GitLab’s container registry documentation](https://docs.gitlab.com/ee/user/packages/container_registry/).) The final step in your pipeline is to log in to your server, pull the latest Docker image, remove the old container, and start a new container.

1. Go to the **Project overview** page, click the **+** button and select **New
    file**. Then set the **File name** to .gitlab-ci.yml or (Alternatively you
    can clone the repository and make all following changes to .gitlab-ci.yml on
    your local machine, then commit and push to the remote repository.).
    **Note:** We follow the latter.

2. Paste the following code into the file.

```yaml
stages:
  - publish
  - deploy

variables:
  TAG_LATEST: $CI_REGISTRY_IMAGE/$CI_COMMIT_REF_NAME:latest
  TAG_COMMIT: $CI_REGISTRY_IMAGE/$CI_COMMIT_REF_NAME:$CI_COMMIT_SHORT_SHA

publish:
  image: docker:18.09
  stage: publish
  services:
    - name: docker:18.09-dind

  script:
    - docker build -t $TAG_COMMIT -t $TAG_LATEST .
    - docker login -u gitlab-ci-token -p $CI_BUILD_TOKEN $CI_REGISTRY
    - docker push $TAG_COMMIT
    - docker push $TAG_LATEST

deploy:
  image: alpine:latest
  stage: deploy
  script:
    - chmod og= $PRIVATE_KEY
    - apk update && apk add openssh-client
    - ssh -i $PRIVATE_KEY -o StrictHostKeyChecking=no $SERVER_USER@$SERVER_IP "docker login -u gitlab-ci-token -p $CI_BUILD_TOKEN $CI_REGISTRY"
    - ssh -i $PRIVATE_KEY -o StrictHostKeyChecking=no $SERVER_USER@$SERVER_IP "docker pull $TAG_COMMIT"
    - ssh -i $PRIVATE_KEY -o StrictHostKeyChecking=no $SERVER_USER@$SERVER_IP "docker container rm -f cbme-website || true"
    - ssh -i $PRIVATE_KEY -o StrictHostKeyChecking=no $SERVER_USER@$SERVER_IP "docker run -d -p 3002:80 --name cbme-website $TAG_COMMIT"
  environment:
    name: production
    url: http://your_website
  only:
    - master
```

3. I will explain only a part of the code present here. A detailed explaination
    of the entire process is provided in the link below:

    <https://www.digitalocean.com/community/tutorials/how-to-set-up-a-continuous-deployment-pipeline-with-gitlab-ci-cd-on-ubuntu-18-04>

4. The explaination is as follows:

    - The _stages_ field show the build stages, each runs separately one after
        another, the _stage’s_ running status can be found in **CD/CI -\>
        Pipeline.**

    - The _stage_ field inside a title field will be equal to one of the
        stages named in the 4.a field.

    - The stage _publish_ does:

        1. Build a new docker image

        2. To store the newly built image into the GitLab private repository
            using push, we login to docker using GitLab provided data.

        3. We push the recent update into Container Registry as well as commit
            update for restore purposes.

    - The stage _deploy_ does:

        1. Make the private key accessible

        2. Update the openssh-client present in the docker

        3. Access the GitLab-runner machine as the **user_name** using SSH via
            the command.

> `ssh -i $PRIVATE_KEY -o StrictHostKeyChecking=no $SERVER_USER@$SERVER_IP “command”`
>
> where:
> **-i identity_file** A file from which the [**identity
> key**](https://www.ssh.com/ssh/identity-key) (private key) for [**public key authentication**]
> (<https://www.ssh.com/ssh/public-key-authentication>) is read.
> **-o options** > **StrictHostKeyChecking** Specifies if ssh should
> automatically add host keys to the \~/.ssh/known_hosts file, and
> refuses to connect to hosts whose host key has changed.

The below is used to run each of these commands

```bash
docker login -u gitlab-ci-token -p $CI_BUILD_TOKEN $CI_REGISTRY
docker pull $TAG_COMMIT
docker container rm -f my-app || true
docker run -d -p 80:80 --name my-app $TAG_COMMIT
```

5. Finally click Commit changes at the bottom of the page in GitLab to create the .gitlab-ci.yml file. Alternatively, when you have cloned the Git repository locally, commit and push the file to the remote.

6. You’ve created a GitLab CI/CD configuration for building a Docker image and
    deploying it to your server. You can check whether the deployment went
    successfully by looking at the debug log inside each job and check running
    docker container by using `docker ps`

## Step 7 -\> Validating Deployment

Now you’ll validate the deployment in various places of GitLab as well as on your server and in a browser.

When a .gitlab-ci.yml file is pushed to the repository, GitLab will automatically detect it and start a CI/CD pipeline. At the time you created the .gitlab-ci.yml file, GitLab started the first pipeline.

1. Go to **CI/CD** -\> **Pipelines** in your GitLab project to see the
    pipeline’s status. If the jobs are still running/pending, wait until they
    are complete. You will see a **Passed** pipeline with two green checkmarks,
    denoting that the publish and deploy job ran successfully.

2. Go to **CI/CD -\> Jobs** in your GitLab project to see the job status of
    each stage of pipeline. Click on _Status_ or the _jobId_ to view the job
    debug log.

> **Note:** Your GitLab container registry may have an [expiration
> policy](https://gitlab.com/help/user/packages/container_registry/index#expiration-policy).
> The expiration policy regularly removes older images and tags from the
> container registry. As a consequence, a deployment that is older than the
> expiration policy would fail to re-deploy, because the Docker image for this
> commit will have been removed from the registry. You can manage the
> expiration policy in **Settings \> CI/CD \> Container Registry tag
> expiration policy**. The expiration interval is usually set to something
> high, like 90 days. But when you run into the case of trying to deploy an
> image that has been removed from the registry due to the expiration policy,
> you can solve the problem by re-running the **publish** job of that
> particular pipeline as well, which will re-create and push the image for the
> given commit to registry.

## Step 8 -\> Rolling back a deployment

1. Move over to **Operations \> Environments \> production** you will see the
    newly created deployment.

2. Now click the **re-deploy** button of the initial, older deployment.

3. Confirm the popup by clicking the **Rollback** button.

> The deploy job of that older pipeline will be restarted and you will be
> redirected to the job’s overview page. Wait for the job to finish, then open
> <http://your_website> in a browser, where you’ll see the older revision coming up
> again.

## References

1. [How To Set Up a Continuous Deployment Pipeline with GitLab CI/CD on Ubuntu 18.04 \| DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-continuous-deployment-pipeline-with-gitlab-ci-cd-on-ubuntu-18-04) – entire instructions present here

2. [GitLab CI/CD \| GitLab](https://docs.gitlab.com/ee/ci/README.html) – GitLab CD/CI Manual
