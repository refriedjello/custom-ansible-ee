# custom-ansible-ee

How to build a custom Ansible Execution Environment without using ansible-builder.

This repo is just a simple README discussing how to build a custom Ansible EE without relying on RH supplied upstream images or `ansible-builder` for the final image build. No code here.

# pre-reqs

This work was done in the following environment:

- RHEL8.8 (Ootpa)
- AAP 2.2 (Ansible Automation Platform)
- podman version 4.4.1
- RedHat UBI 8.6 image
- the podman image must contain the following software at a min:
  - Python3.8 - Python3.10 (I use python3.9)
  - Ansible 2.9 or Ansible Core 2.11 - 2.13 (I use ansible-core 2.12)
  - Ansible Runner (I use ansible-runner 2.1.4)
  - see the sorta, but not super, helpful document: https://access.redhat.com/documentation/en-us/red_hat_ansible_automation_platform/2.3/html-single/creating_and_consuming_execution_environments/index#con-about-ee
- your own stuff, like Collections or Python libraries, once you get it working

# why?

Maybe you don't want to use `ansible-builder` on philosphical or emotional grounds. Or maybe you maintain numerous image types, and like me, it bugs you the canon method requires you use a differnet method for building/maintaining Ansible EEs.

# referencing current RH published EE image

This is an act of mimcry, what did RH peeps do to get things to work, so we can do the same thing.

The latest-supported supplied RH image to base your own modified EEs on is below:

https://quay.io/repository/ansible/awx-ee

Download that to your local namespace:

```
podman pull quay.io/ansible/awx-ee
```

Inspect that image to see how it was built using the following command.

```
podman image inspect quay.io/ansible/awx-ee
```

I won't past the entire output here for brevity. Study the output a bit and see how it was built. There are some obvious things you can skip.

As far as what to pay attention to, some of the output spans multiple lines and is subject to change in the future, but here are the keys that I found to be relevant:

- Env:Path
- Config:Entrypoint
- Cmd
- Config:WorkingDir
- History

In the next section I will show specifically what I did to get a completely custom EE image working on AAP.

# prep work

Ensure you meet the standards defined in the section "pre-reqs" and then create a sub-dir to store your work.

I will create a new sub-dir under my home directory named `my-new-ee` and then enter into it:

```
cd
mkdir my-new-ee
cd my-new-ee
```

# extract the entrypoint scripts

If you are perspicacous (like I pretend to be) you will have noticed there is a custom `ENTRYPOINT` line that calls on two scripts:

```
/opt/builder/bin/entrypoint
dumb-init
```

Note: in earlier versions of the RH EE image named `ansible-runner` the entrypoint script called `dumb-init`.

The second one is actually a binary file and so the simplest way to extract them both is to launch an instance of the RH "official" image and run `podman cp` against that container instance and those files, as shown below:

Note: Be sure to do this in the build diretory you are using to keep your files nice and sorted
Note: the full-path to the script `dumb-init` is not listed (sadly) but you can always enter the container and search for it if needed. The full path is revealed below.

```
podman run -d quay.io/ansible/awx-ee
```

That command will launch a container instance, using an interactive shell and in detached mode (to keep it from auto-stopping) from the RH EE image. You will get the container ID in your output, but in case you missed it you can find it via this (your output will vary):

```
podman container list --all | grep awx
8b90b31b1d4f  quay.io/ansible/awx-ee:latest                                      bash                  54 seconds ago  Up 54 seconds                          agitated_banzai
```

Note: the 2nd to last column reads "Up" which is what we need; we need the container to be running to execute the `podman cp` command.

Now lets extract those scripts. You will need the container ID of the launched container from the `awx-ee` image above. I am using the container ID running on my machine, be sure to replace with the exact container ID running on yours:

```
podman cp 8b90b31b1d4f:/opt/builder/bin/entrypoint entrypoint
podman cp 8b90b31b1d4f:/usr/local/bin/dumb-init dumb-init
```

Now run `ls` and you should see that both files exist within your EE directory.

# special note - entering container instance of the RH EE image

If you need/want to poke around in a launched instance of that image you can use the command below:

```
podman run -it quay.io/ansible/awx-ee /bin/bash
```

# writing a quick test playbook

We need something to test against to make sure Ansible actually works.

Create a file named `test.yml` with the following contents:

```
---
- hosts: localhost
  gather_facts: no

  tasks:

  - name: test ansible command
    ansible.builtin.ping:
```

That just runs a simple Ansible task against the localhost, which will be our container. It just tests that all of the Ansible bits and bobs can be called successfully and doesn't do anything else.

# create Containerfile

Create a file named `Containerfile` and add the following lines:

```
# the base image to use
FROM registry.access.redhat.com/ubi8/ubi:8.6-903

# mimic baked in settings from the RH EE image
WORKDIR /runner
ENTRYPOINT ["/opt/builder/bin/entrypoint"]
USER root
ENV HOME=/home/runner
COPY entrypoint /opt/builder/bin/entrypoint
COPY dumb-init /usr/local/bin/dumb-init

# prep our custom image to be an EE
RUN yum update -y; true
RUN yum install -y python39 python39-pip
RUN python3.9 -m pip install ansible-core==2.12.5 ansible-runner==2.1.4
COPY test.yml /home/runner/test.yml

# your special stuff below
```

Let's run through this real quick:

- `FROM` says which source imagem to base off. As mentioned, I am using the RH UBI image. The main thing here is that this image is not built to function as an Ansible EE, we made it so.
- The next block is from the legwork we did when we ran `podman inspect` against the official EE image. The output from that command has been translated into docker `Containerfile` lines as appropriate. Note the two `COPY` lines which re-inject the two `ENTRYPOINT` scripts we saw in that output.
- The block after to prep our image is where we start to turn this new image into an Ansible playbook running machine. I install Python 3.9 here because that's what I use, but ultimately it comes down to what Ansible is compatible with. I also install `pip` because I like a soft life.
- In that section I also install the sofware requisites to run Ansible on this image. The necessary Ansible software. I have pinned the versions here based on what I run in prod currently
- Lastly I copy the `test.yml` playbook we made above so we have a way show our work

# building your new custom EE

Okay! We have exfiltrated the files we need from the supported EE image, wrote a quick test playbook, and crafted a `Containerfile` that contains the instructions for `podman` to use to build a new image.

Run the command below to do the thing that will build a new image:

```
podman build -t my-new-ee -f Containerfile
```

Note: the argument to `-t` is the image name we have been using here.

You will see a ton of output and if you are good at following instructions and a bit lucky, the process will have completed successfully.

At the very end of the output you are looking for something like this:

```
COMMIT my-new-ee
--> f44dac649af
Successfully tagged localhost/my-new-ee:latest
```

Pretty obvious that's good stuff and we can proceed.

# testing your new shiny EE locally

When using Ansible Tower/AAP/AWX/whatever it will launch the command `ansible-runner` to actually instantiate your image and run the playbook specified by the Template definition you have. We can test some of that here, and make sure `ansible-runner` actually works. Because if it doesn't work here, it won't work in AAP.

Use the command below to call `ansible-runner` against your shiny EE image and call the test playbook. The argument `somedir` doesn't really matter, it's the local directory on the host it will use to store any stuffs.

```
ansible-runner run --process-isolation --process-isolation-executable podman --container-image my-new-ee somedir -p /home/runner/test.yml
```

Here is the full output I get from running that (I trimmed bullshit out):

```
$ ansible-runner run --process-isolation --process-isolation-executable podman --container-image my-new-ee somedir -p /home/runner/test.yml

PLAY [localhost] ***************************************************************

TASK [test ansible command] ****************************************************
ok: [localhost]

PLAY RECAP *********************************************************************
localhost                  : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

That is what we want to see. It was able to run our little test playbook successfully and do its little thing.

# next steps

The next steps of the entire work-flow are not covered here. But you will need to get this EE uploaded to a container registry, defined on AAP/Tower, and have a Project and Template setup on AAP/Tower in order to test it fully. And of course, depending on your circumstances, you will need to tweak your `Containerfile` and re-build and re-publish.

Don't forget to test locally where possible to save time.

But the fact that you can run `ansible-runner` locally against it is very promising.
