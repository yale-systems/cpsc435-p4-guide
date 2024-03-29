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
    $ cp solution/* .
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


## Running Projects with p4app
You can use the following steps to utilize p4app inside the VM. However, please note that this may **not** be the most efficient or straightforward method to utilize p4app. Feel free to adjust the scripts as needed for your project directory structure.

First, begin by cloning the p4app repository:
```bash
$ cd ~/
$ git clone https://github.com/p4lang/p4app.git --branch rc-2.0.0 --recursive
$ cd p4app/docker/scripts
```

Next, create the `/p4app/` and `/tmp/p4app-logs` directories which will be utilized by p4app scripts.
```bash
$ sudo mkdir /p4app/
$ mkdir /tmp/p4app-logs
```
Note that `/tmp/p4app-logs` directory will be removed if you reboot the VM. You have to recreate it after each reboot.

Since p4app scripts require sudo permissions and p4setup.bash sourcing, we create a script to simplify this process. Remember to replace YOUR_USERNAME with your actual username!
```bash
$ touch run.sh
$ echo 'source /home/YOUR_USERNAME/p4-build/p4setup.bash
python3 p4apprunner.py' > run.sh 
$ chmod +x run.sh
```

Now, let's run an example. To execute a project, we need to copy all of its contents into the `/p4app` directory.
```bash
$ sudo cp ../../examples/basic.p4app/* /p4app/
```

Finally, we need to modify the `main.py` of the project and add the following lines to the **beginning** of the file:
```bash
import sys
sys.path.append("/home/YOUR_USERNAME/p4app/docker/scripts")
```

Your project is now set up and ready to run! You can execute it using the following command.
```bash
$ sudo -s ./run.sh 

> python /p4app/main.py 
> p4c-bm2-ss --std p4-16 "/p4app/basic.p4" -o "/tmp/p4app-logs/basic.json" --p4runtime-files "/tmp/p4app-logs/basic.p4info.txt"
[--Wwarn=deprecated] warning: .txt format is being deprecated; use .txtpb instead

----- Reading tables rules for s1 -----
MyIngress.ipv4_lpm:  hdr.ipv4.dstAddr (b'\n\x00\x00\x01', 32) -> MyIngress.ipv4_forward dstAddr b'>\x0eK\x8d\xb2\xff' port b'\x01' 
MyIngress.ipv4_lpm:  hdr.ipv4.dstAddr (b'\n\x00\x00\x02', 32) -> MyIngress.ipv4_forward dstAddr b'fhn.\xb2&' port b'\x02' 
*** Ping: testing ping reachability
h1 -> h2 
h2 -> h1 
*** Results: 0% dropped (2/2 received)

----- Reading tables rules for s1 -----
OK
```


## Running the starter project with P4
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
