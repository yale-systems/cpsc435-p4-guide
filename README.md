# P4 on a Virtual Machine on MacOS
If you want to run p4 programs on M1/M2 Macs and you have encountered errors such as interfaces not being created and dropped packets on p4app examples, you can follow the instructions below which will manually install all the required applications to run a p4 program. 

## Installation
- First, you should create an Ubuntu virtual machine using UTM. You can use the following tutorial which shows you how you can set up a virtual machine on Mac. I recommend allocating at least 8GB of Memory and 100GB of disk to the VM you create. 
If you encounter any problems here, try updating UTM and rebooting the computer.

    https://www.youtube.com/watch?v=FS-HJTM6Oec&ab_channel=TatOGTech

- Next, you should clone the p4-guide repo and start the installation process as below. Note that this part may take ~2 hours.
    ```bash
    $ mkdir ~/p4-build
    $ cd ~/p4-build
    $ sudo apt install git
    $ git clone https://github.com/jafingerhut/p4-guide
    $ ./p4-guide/bin/install-p4dev-v7.sh |& tee log.txt
    ```

- After the installation is done, you should execute the command `source p4setup.bash` in every bash shell where you wish to run the P4 development tools. You can also add this command to `$HOME/.bashrc`
    ```bash
    $ echo 'source $HOME/p4-build/p4setup.bash >> $HOME/.bashrc'
    ```

- You can check if your installation was successful by testing one of the examples in the p4-guide directory.
    ```bash
    $ cd tutorials/exercises/basic/
    $ make run
    ```
    Here you should see a Mininet terminal showing and you can test p4 using 
    ```bash
    ======================================================================
    Welcome to the BMV2 Mininet CLI!
    ======================================================================
    ...

    mininet> net
    h1 eth0:s1-eth1
    h2 eth0:s1-eth2
    h3 eth0:s2-eth1
    h4 eth0:s2-eth2
    s1 lo:  s1-eth1:eth0 s1-eth2:eth0 s1-eth3:s3-eth1 s1-eth4:s4-eth2
    s2 lo:  s2-eth1:eth0 s2-eth2:eth0 s2-eth3:s4-eth1 s2-eth4:s3-eth2
    s3 lo:  s3-eth1:s1-eth3 s3-eth2:s2-eth4
    s4 lo:  s4-eth1:s2-eth3 s4-eth2:s1-eth4

    mininet> pingall
    *** Ping: testing ping reachability
    h1 -> h2 h3 h4 
    h2 -> h1 h3 h4 
    h3 -> h1 h2 h4 
    h4 -> h1 h2 h3 
    *** Results: 0% dropped (12/12 received)
    mininet> 
    ```

## Running the starter project
- Now you can clone the switch-cache repo 
    ```bash
    $ cd ~/
    $ git clone https://github.com/yale-build-a-router/switch-cache.git
    $ cd switch-cache/cache.p4app/
    ```

- In order to run the applications, we need to define the network topology we need in JSON format. You only need to create a network with two hosts and one switch for the starter project (`h1 --- s1 --- h2`). For a working example, you may refer to `tutorials/exercises/basic/pod-topo/`. 

- Now we need to configure a Makefile for the project. First, execute the following command to bring the original Makefile to your directory and name it `common.mk`.
    ```bash
    $ cp ~/p4-build/tutorials/utils/Makefile common.mk
    ```
    *\*\*Make sure to modify the RUN_SCRIPT variable inside the common.mk file to point to `~/p4-build/tutorials/utils/run_exercise.py`*.

    Assuming you have the following directory structure, you can use a Makefile as below.
    ```bash
    ├── p4-build/
    ├── switch-cache
        ├── cache.p4app
            ├── topo/
                ├── topology.json
                ├── s1-runtime.json
            ├── cache.p4
            ├── client.py
            ├── common.mk
            ├── main.py         # We won't need this file anymore
            ├── Makefile
            ├── server.py
    ```
    `Makefile:`
    ```Makefile
    BMV2_SWITCH_EXE = simple_switch_grpc
    TOPO = topo/topology.json

    include ./common.mk
    ```

- Everything is ready now to run your program. First, you should complete the cache.p4 file. Then we should check if nodes can ping each other.
    ```bash
    $ make run
    ...
    ======================================================================
    Welcome to the BMV2 Mininet CLI!
    ======================================================================
    ...
    mininet> net
    h1 eth0:s1-eth1
    h2 eth0:s1-eth2
    s1 lo:  s1-eth1:eth0 s1-eth2:eth0
    mininet> pingall
    *** Ping: testing ping reachability
    h1 -> h2 
    h2 -> h1 
    *** Results: 0% dropped (2/2 received)
    ```

- Now, let's run the server on h1 and try to send some requests to it from h2. Here we assume the IP address of h1 is 10.0.1.1. You can set the nodes' IP addresses in `topo/topology.json`
    ```bash
    mininet> h1 ./server.py 1=123 2=345 &
    mininet> h2 ./client.py 10.0.1.1 1
    123
    mininet> h2 ./client.py 10.0.1.1 2
    345
    mininet> h2 ./client.py 10.0.1.1 11
    NOTFOUND
    ```

## Populating tables on the switches during runtime with P4Runtime
You may need to add entries to your P4 tables after running the network. For instance, in the cache project, you need to have a table that stores some cached key-value pairs and can be updated during runtime. To do this, you can use the `mycontroller.py` provided in this directory. 

**NOTE** You must fill the parts marked with 'TODO' in `mycontroller.py` (in lines 9,43,77) before using it! 

An example:
```bash
mininet> h1 ./server.py 1=123 &
mininet> h2 ./client.py 10.0.1.1 1
123
mininet> h2 ./client.py 10.0.1.1 100
NOTFOUND
```

Now open another terminal and run `mycontroller.py`:
```bash
$ ./mycontroller.py 
Installed P4 Program using SetForwardingPipelineConfig on s1
Installed cache rule on s1

----- Reading tables rules for s1 -----
table_id: 48142472
match {
  field_id: 1
  exact {
    value: "d"
  }
}
action {
  action {
    action_id: 22586559
    params {
      param_id: 1
      value: "\032"
    }
  }
}

-----
```

Go back to the miniet CLI and check for the new value again:
```bash
mininet> h2 ./client.py 10.0.1.1 100
26
```