# Overview of Important Shell Commands

## Zombie Apocalypse

When logging into the server, I can occasionally see a message on the welcome screen like

```text
  => There is 1 zombie process.
```

We can find out what the process and its ID is:

```shell
marvey@achterhus:~$ ps aux | awk '$8=="Z"'
root       36048  0.0  0.0      0     0 ?        Z    Feb25   0:02 [python] <defunct>
marvey@achterhus:~$ ps -ef | grep '[d]efunct'
root       36048    1680  0 Feb25 ?        00:00:02 [python] <defunct>
```

It's apparently a Python process, so let's check who the parent is:

```shell
marvey@achterhus:~$ ps -p 1680 -o comm,args
COMMAND         COMMAND
esphome         /usr/local/bin/python /usr/local/bin/esphome dashboard /config
```

Since ESPHome is running in a Docker container using `network_mode: host`, we can simply restart the container itself, which should take care of any zombies:

```shell
marvey@achterhus:~$ docker restart esphome
esphome
marvey@achterhus:~$ ps aux | awk '$8=="Z"'
marvey@achterhus:~$ ps -ef | grep '[d]efunct'
marvey@achterhus:~$
```

If this were a regular process, we might have nudged the parent to collect its zombie children by calling `kill -s SIGCHLD <parent_pid>`.
