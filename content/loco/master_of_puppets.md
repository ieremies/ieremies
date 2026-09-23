---
title: Master of Puppets
draft: false
type: "docs"
toc: true
---

{{< callout type="important" >}} 
This a rather long tutorial. Some parts of it you might already know, so
feel free to use the table of contents to skip a head.
{{< /callout >}}

## Access (`ssh`)

Lets first access the login node:

1.  If you don\'t have an ssh key, generate one:

    ``` bash
    ssh-keygen -t ed25519 -a 100 -C "name@laptop" -f ~/.ssh/id_ed25519
    ```

2.  Send the admin your public key. It is one line, e.g.
    `ssh-ed25519 AAC3Nz... name@laptop`. Also, indicate your expected
    username, something in the lines of `john.doe`.

    ``` bash
    cat ~/.ssh/id_ed25519.pub
    ```

3.  After you received confirmation that everything is configured, you
    can loggin with:

    ``` bash
    ssh -p 2222 USERNAME@MACHINE.loco.ic.unicamp.br 
    ```

    Change `USERNAME` and `MACHINE` to what the admin indicates. When
    connecting to `ic.unicamp.br` via ssh, you need use the port `2222`
    and be connected in UNICAMP\'s [VPN](https://www.detic.unicamp.br/servicos/acesso-remoto-seguro-vpn/).

### Admin side (for reference)

1.  Create the new user

    ``` bash
    useradd -m -s /bin/bash USERNAME
    ```

2.  Add the credentials

    ``` bash
    install -d -m 700 -o USERNAME -g USERNAME /home/USERNAME/.ssh
    echo 'ssh-ed25519 AAAAC3... name@laptop' >> /home/USERNAME/.ssh/authorized_keys
    chown USERNAME:USERNAME /home/USERNAME/.ssh/authorized_keys
    chmod 600 /home/USERNAME/.ssh/authorized_keys
    ```

Debug: check the permissions, it should be 600 on `authorized_keys`, 700
on .ssh.

``` bash
stat -c '%a %U %n' /home/USERNAME /home/USERNAME/.ssh /home/USERNAME/.ssh/authorized_keys
```

### Sending something to the cluster{#send}

The fastest way is to use `rsync`. The basic command is:

``` bash
#                     source       destination
rsync -avz --progress /local/path/ user@server:/remote/path/
```

In general, I use:

``` bash
rsync -avz --info=progress2 --partial \
  -e "ssh -i ~/.ssh/id_ed25519 -p 2222" \
  --exclude='.git' \
  --filter=":e- .gitignore" \
  /local/path/ user@server:/remote/path/
```

Flag description:

- `-a` archive mode --- recursive + preserve perms, times, symlinks,
  ownership always
- `-v/-z` verbose / compress during transfer always
- `--info=progress2` show overall progress always
- `--partial` keep half-transferred file on interruption big files,
  flaky link
- `-e \"ssh ...\"` explicit ssh command non-default port or key
- `--exclude` skip dirs/files, e.g. --exclude=\'.git\'
  --exclude=\'\*.tmp\' don\'t copy junk

Other flags to consider:

- `--delete` remove remote files not in source only if mirroring ---
  dangerous, read warning below
- `-n` (dry run) preview without sending first run, always
- `--checksum` compare by checksum, not size+mtime mtimes unreliable
  across systems

Since I usually keep things at my home directory on the server, I have a
quick bash function as follows:

``` bash
# in ~/.bashrc or ~/.zshrc
beam() {
    local dir="$1"
    rsync -avz --info=progress2 --partial \
      -e "ssh -i ~/.ssh/id_ed25519 -p 2222" \
      --exclude='.git' \
      --filter=":e- .gitignore" \
      "$dir" USERNAME@pricer.loco.ic.unicamp.br:/home/USERNAME
}
```

Just remember to change your `USERNAME`. Them, when I want to send
(\"beam\") something to the machine:

``` bash
beam my_code 
```

and it will arrive at `~/my_code`.

I also have another function to \"beam\" things from the server. In this
case, you just need to swap the source and destination on the rsync
command.

## Slurm

The official slurm documentation does a great job of explaining the
concepts. Althought not everything is applicable in our context (such as
GPUs and network constraints), I do recomend a quick look around:

- [Slurm Workload Manager - Quick Start User
  Guide](https://slurm.schedmd.com/quickstart.html)
- [Slurm Workload Manager - Job Array
  Support](https://slurm.schedmd.com/job_array.html)

Here is a quick rundown of the main terms slurm uses:

- A **node** is a machine, a single computer. (ps: a single machine can
  be split into multiple nodes, but we will not do this)
- A **partition** is a set of nodes.

Slurm encapsulates resources using the idea of jobs and tasks.

- A **job** can span multiple compute nodes and is the sum of all task
  resources.
- **Tasks** are a subset of resources in a job and can only exists on a
  single compute node.
- In practice you build a job based on the number of tasks you want to
  run, in parallel, and how many resources you want for each task.

In terms of computational power, slurm defines:

- Thread: one or more hardware contexts withing a single core
- Core: a complete private set of registers, physical unit
- CPU: this can mean different things, but, since we configured
  `SelectTypeParameters=CR_Core_Memory`, a CPU is a single core

### Quick tour: the queue

Before anything, look at what is running on the cluster:

``` bash
squeue
```

An empty queue looks like this:
```
    JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
```

Submit a quick job with `srun`. It runs on a compute node but attaches
to your terminal, so you see the output directly (and you are blocked
until it finishes):

``` bash
srun --ntasks=1 --cpus-per-task=1 --hint=nomultithread \
     --cpu-bind=cores --mem-bind=local --mem=1G \
     hostname
```

No script file needed for a one-liner: `sbatch --wrap` wraps the command
in a small shell script for you. See [complicated](#complicated) after.

``` bash
sbatch --job-name=demo --output=demo_%j.log \
       --ntasks=1 --cpus-per-task=1 \
       --hint=nomultithread --cpu-bind=cores --mem-bind=local \
       --mem=1G --time=60 \
       --wrap "sleep 20 && echo \"I am done\""
```

`sbatch` returns immediately and prints the new job id:

```
    Submitted batch job 42
```

Check the queue again --- the job shows up there (`ST` column: `PD` =
pending, `R` = running, `CG` = completing):

``` bash
squeue
```
```
    JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
       42    main      demo    USER1  R       0:04      1 pricer
```

While it runs, its output goes to `demo_42.log` (the `%j` in `--output`
is replaced by the job id). Because of the sleep, the job lives `20s`.

If you want it gone before it finishes, cancel by job id:

``` bash
scancel 42
```

Remember: once a job is gone from `squeue`, it leaves no trace there.

### Running more complicated tasks{#complicated}

Every project might have some difference to how it runs. For this
tutorial, I will assume something like this:

``` bash
./binary_or_script INSTANCE --param 0
```

In this scenario, a command is given an instance and perhaps a set of
parameters. It does its thing and reports by either printing to `stdout`
or writing a log file. I will also assume, as it is common, that you are
not using multi-threading or paralelism. Note: I am running ONE instance
against ONE set of parameters, not a class of instances, not a
combination of parameters.

As we have done above, we can submit this task as

``` bash
srun "./binary_or_script INSTANCE --param 0"
```

`srun` attaches the process to your terminal, so it is only really
useful for debuging or interative process, but you are still subject to
the queue.

A better approach is to create a *batch script* file, I will call it
`job.sh`.

``` bash
#!/bin/bash
#SBATCH --job-name=my_test
#SBATCH --output=output_%j.log
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=1
#SBATCH --hint=nomultithread
#SBATCH --cpu-bind=cores
#SBATCH --mem-bind=local
#SBATCH --mem=4G
#SBATCH --time=01:00:00

echo "Running my task on SLURM"
./binary_or_script INSTANCE --param 0
```

Let me explain each parameter (if not specified, the example values are
also the default): 
| Flag                   | Meaning                                                                               |
|------------------------|---------------------------------------------------------------------------------------|
| `--output=out_%j.log`  | File for stdout+stderr logs; `%j` → job ID                                            |
| `--ntasks=1`           | Number of tasks/processes (default 1 in this case)                                    |
| `--cpus-per-task=1`    | CPU cores per task                                                                    |
| `--hint=nomultithread` | Disable hyperthreading, use physical cores. Does nothing if process already single-threaded |
| `--cpu-bind=cores`     | Pin tasks to assigned cores only                                                      |
| `--mem-bind=local`     | Allocate memory from local NUMA node nearest assigned cores. Default: `none`          |
| `--mem=4G`             | RAM for job. Requests 4 GB                                                            |
| `--time=60`            | Max wall-clock 60 minutes; SLURM kills job if exceeded. Default: `120`                |


If you want a deeper explanation, feel free to chat with me. If you
don\'t care and just want to run your experiment, the only parameter I
need you to take care is the `--time`. I recomend you use it as low as
you can, so your job can be queued faster and the system will run
better. This way, if you know your experiment will last for only a
minute, maybe use `--time=5` instead of `60`.

### Best practices

This is a set of rules we ask you to follow, but will not be enforced:

- If you will be running a multi hour experiment that you only need for
  tomorrow, we ask you to use the `--begin=19:00`;
- If your experiment is expected to take days, please use the `--begin`
  argument to start it on friday night.
- If possible, prefer to split each instance into one job. This
  facilitates the scheduler job.
- Indicate the `--time` as close to your expected running time as
  possible. The same is true for memory usage.
- If your instances are expected to run really quick (less than a second
  in total, for example), consider packing them into one single job.
  Otherwise, most of the time will be wasted setting up and shutting
  down the container.
- Clean up your log files regularly. Each user has (by default) 10GB of
  home folder storage.

## Apptainer

Before we continue, some quick explanations:

- apptainer will mount your home to the container, so anything inside it
  will also be available in the container by default;
- apptainer will NOT use any program from the host system, that is why
  we use it;
- apptainer will NOT see anything outside the home if not \"binded\"
  explicitly.

The core idea:

- **Image** = one read-only `.sif` file (or a `--sandbox` directory
  during development). Your container programs live in it; your **data
  lives outside** and is mounted in.
- **Definition file** (`.def`) = recipe: base image + `%post` install
  steps + environment + default command. Build once, run many times.
- **Three entry points**: `apptainer run img.sif` (the image\'s
  `%runscript`), `apptainer exec img.sif <cmd>` (run any command),
  `apptainer shell img.sif` (interactive shell).
- **Binds (mounts)**, automatic: your `$HOME`, the current working
  directory, and `/tmp` are always mounted **read-write**. Everything
  else on the host is invisible until you `--bind` it.
- **The image is read-only.** Container processes can only write where a
  bind points (your home, /tmp, or an explicit `--bind`). Files they
  create on a bind are owned by **you**, not root.
- **Host env vars pass through** into the container automatically
  (proxies, `OMP_NUM_THREADS`, ...). Set per-run extras with
  `--env VAR=val` or `--env-file file`, or permanently in the image\'s
  `%environment`.

If you don\'t need to recompile your code very often or your
experiements are really quick (\~1ms), you can compile your code as a
build step in the container and cut its size down using tools like
[slimtoolkit/slim](https://github.com/slimtoolkit/slim).

### Defining your container

If you have used containers, such as Docker, it is very similar. If not,
don\'t worry, follow this steps:

1.  Define your container in `.def` file. Here is an example:
    ```
    Bootstrap: docker
    From: ubuntu:24.04

    %post
        apt-get update
        DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
            build-essential make \
            && rm -rf /var/lib/apt/lists/*    # reduce container size
    ```
2.  Write this into a file such as `~/example.def` in your home folder.

3.  Them, you need to `build` it using
    `apptainer build ~/example.sif ~/example.def`. The first parameter
    is the output and should be a `.sif`, while the second is the
    definition we just wrote.

4.  After building (it can take 1 or 2 minutes), you should have an
    `~/example.sif` in your home.

To execute command inside the container is a matter of
`apptainer exec example.sif bash -c 'ls'`.

- `example.sif` is the image\'s name
- `bash` is the interpreter
- `-c ''` indicates it must run the string as a command

For example, if you want to compile your code which is in `~/my_code`,
you can

``` bash
apptainer exec example.sif bash -c 'cd ~/my_code && make'
```

The resulting binary will be in its traditional place, but the
compilation (and linkage) was done inside the container, therefore, it
must also be ran inside it.

If you need to use gurobi, check out [gurobi](#gurobi).

### Compiling and running with gurobi{#gurobi}

Now that everything is setup, we can compile it. I will assume you can
already compile with gurobi in your machine, where it is probably
installed under `/opt/gurobi/`.

We use the following command to compile:

``` bash
apptainer exec \
    --bind "$HOME/gurobi1303:/opt/gurobi1303" \
    --bind "$HOME/gurobi.lic:/root/gurobi.lic" \
    example.sif bash -c 'cd ~/my_code && make'
```

As we said previously, the container does not *contain* anything from
the host machine, and only the things explicitly added in its `.def`. We
can \"mount\" the gurobi in our home to the container `/opt/gurobi`
using `--bind "$HOME/gurobi1303:/opt/gurobi"`. Inside our container,
gurobi lives in `/opt/gurobi/`, so make sure your linkage step and
header includes knows this. The third line ensures the license is where
gurobi expect it, since inside apptainer you will be running things as
`root`. Them, the rest of the command follows what we have done
previously.

On success, your compilation will produce a binary where you expect it.
Note that, since it was compiled inside the container, it will only work
inside it. So let\'s run it!

``` bash
apptainer exec \
    --bind "$HOME/gurobi1303:/opt/gurobi1303" \
    --bind "$HOME/gurobi.lic:/root/gurobi.lic" \
    example.sif bash -c 'cd ~/my_code && ./a.out'
```

Similarly to what we\'ve done to compile, we must also bind gurobi in
order to run it.

### \"Complete\" example of a `.def` file

Here is a `.def` that can run C/C++ with the boost library, python
(using [uv](https://docs.astral.sh/uv/)) and gurobi.

```
Bootstrap: docker
From: ubuntu:24.04

%post
    apt-get update
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
        build-essential \
        cmake make g++ \
        libboost-all-dev libabsl-dev \
        curl ca-certificates \
        git \
        python3-dev \
        && rm -rf /var/lib/apt/lists/*


    # Gurobi validates WLS licenses via a token POST to token.gurobi.com and
    # hardcodes the RHEL CA bundle path. Ubuntu keeps the bundle under
    # /etc/ssl/certs — expose it at the path Gurobi expects.
    mkdir -p /etc/pki/tls/certs
    ln -sf /etc/ssl/certs/ca-certificates.crt /etc/pki/tls/certs/ca-bundle.crt

    # Install uv
    curl -LsSf https://astral.sh/uv/install.sh | env UV_INSTALL_DIR=/opt/uv sh

%environment
    export GUROBI_HOME=/opt/gurobi
    export GRB_LICENSE_FILE="${GRB_LICENSE_FILE:-$HOME/gurobi.lic}"
    export LD_LIBRARY_PATH="$GUROBI_HOME/linux64/lib:${LD_LIBRARY_PATH:-}"

    export PATH="/opt/uv:$PATH"
    export UV_CACHE_DIR="$HOME/.cache/uv"
```
