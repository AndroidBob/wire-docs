# How to install wire

We have a pipeline in  `wire-server-deploy` producing container images, static
binaries, ansible playbooks, debian package sources and everything required to
install Wire.

On your machine (we call this the "admin host"), you need to have `docker`
installed (or any other compatible container runtime really, even though
instructions may need to be modified). See [how to install
docker](https://docker.com) for instructions.

Create a fresh workspace to download the artifacts:

```
cd ...  # you pick a good location!
```

Copy over ssh keys, or generate a key pair exclusively for this installation:

```
mkdir -p ./dot_ssh
cp ~/.ssh/id_rsa ./dot_ssh/
```

vs.

```
ssh-keygen -t ed25519 -a 100 -f ./dot_ssh/id_ed25519
ssh-add ./dot_ssh/id_ed25519
````

Ensure the nodes your deploying to, as well as any bastion host between you and them accept this ssh key for the user you are performing the install as:

```
# make sure the server accepts your ssh key for user root
# TODO: replace with ansible oneliner making use of the inventory
ssh-copy-id -i ./dot_ssh/id_ed25519.pub <username>@<server>
```

Obtain the latest artifacts from wire-server-deploy. Once
https://github.com/wireapp/wire-server-deploy/pull/363 is finished, these will
be in the "Releases" tab. Until then, go to the PRs "Checks" tab, and look for
the URL of the "assets" tarball linked from CI.

Extract the above listed artifacts into your workspace:

```
wget https://path.of.the/artifact.tgz
tar xvzf path-to-file.tgz
```

There's also a docker image containing the tooling inside this repo.

If you don't intend to develop *on wire-server-deploy itself*, you should load
this and register an alias:

```
docker load -i assets/wire-server-deploy-*.tar
alias d="docker run -it --network=host -v $PWD:/wire-server-deploy quay.io/wire/wire-server-deploy:$wire_server_deploy_version"
alias dapi="d ansible-playbook -i ansible/inventory/offline/hosts.ini"
```

The following artifacts are provided:

 - `wire-server-deploy-*.tar`
   A container image containing ansible, helm, and other tools and their
   dependencies in versions verified to be compatible with the current wire
   stack. Published to `quay.io/wire/wire-server-deploy` as well, but shipped
   in the artifacts tarball for convenience.
 - `ansible`
   These contain all the ansible playbooks the rest of the guide refers to, as
   well as an example inventory, which should be configured according to the
   environment this is installed into.
 - `binaries`
   This contains static binaries, both used during the kubespray-based
   kubernetes bootstrapping, as well as to provide some binaries that are
   installed during other ansible playbook runs.
 - `charts`
   The charts themselves, as tarballs. We don't use an external helm
   repository, every helm chart dependency is resolved already.
 - `containers-system`
   These are the container images needed to bootstrap kubernetes itself
   (currently using kubespray)
 - `containers-helm`
   These are the container images our charts (and charts we depend on) refer to.
   Also come as tarballs, and are seeded like the system containers.
 - *containers-other*
   These are other container images, not deployed inside k8s. Currently only
   contains restund.
 - `debs`
   This acts as a self-contained dump of all packages required to install
   kubespray, as well as all other packages that are installed by ansible
   playbooks on nodes that don't run kubernetes.
   There's an ansible playbook copying these assets to an "assethost", starting
   a little webserver there serving it, and configuring all nodes to use it as
   a package repo.
 - `values`
   Contains helm chart values and secrets. Needs to be tweaked to the
   environment.

Provide a `ansible/inventory/offline/hosts.ini` configured to your environment.
Copy over `hosts.ini.example`  to `hosts.ini`, and edit it, following the instructions in that file.

Also, take a look at `ansible/inventory/offline/group_vars/all/offline.yml` to
see if it matches your expectations.


Copy over binaries and debs, serves assets from the asset host, and configures
other hosts to fetch debs from it:


```
dapi ansible/setup-offline-sources.yml
```

Run kubespray until docker is installed and runs:

```
dapi ansible/roles-external/kubespray/cluster.yml --tags bastion,bootstrap-os,preinstall,container-engine
```

With docker being installed, seed all container images:

```
dapi ansible/seed-offline-docker.yml
```

Run the rest of kubespray:

```
dapi ansible/kubernetes.yml --skip-tags bootstrap-os,preinstall,container-engine
```

TODO:

 - run other playbooks for other pets.
 - run `helm_external.yml` to render helm values from the ansible inventory
 - run helm to install our charts
 - add zauth tool to our docker container