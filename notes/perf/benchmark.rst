Benchmark Tools
===============

Current Tools
-------------
- netperf/iperf
- ab
- speedtest-cli
- send file by nc

high level wrapper

- daynix/NetMeter: https://github.com/daynix/NetMeter

netperf
-------

installation::

    apt install netperf
    pacman -S netperf
    
    # server side: run server at 12865 port
    netserver
    # client side
    netperf -t TCP_STREAM -H <server_ip>

running::

    netperf -t TCP_STREAM -H <server_ip>

    netperf <global> -- <test-specific>
    # global options:
    #   -t <type>
    #     TCP_STREAM: bandwidth, Mbit/sec
    #     TCP_RR: tcp request/sec
    #     TCP_CRR: TCP_RR, but reconnect tcp connection each request.
    #   -l <time>
    # TCP_RR test specific option
    #   -r <req_size>,<resp_size>
    #      -r 32,1024

- Netperf manual <https://hewlettpackard.github.io/netperf/doc/netperf.html>
- netperf 与网络性能测量 <https://www.ibm.com/developerworks/cn/linux/l-netperf/>

iperf
-----

running::
  
    # server
    iperf -s
    # client
    iperf -c <server_ip>

- iperf tutorial <http://cnds.eecs.jacobs-university.de/courses/anl-2010/Iperf%20Tutorial.pdf>
- https://www.es.net/assets/Uploads/201007-JTIperf.pdf

NetMeter
--------

prepare
~~~~~~~
host::

    apt-get install python3 python3-numpy iperf gnuplot
    // winexe for windows guest

guest::
    
    // ssh server
    apt-get install iperf

sshkey::

    # generate host key
    host$ ssh-keygen -t rsa -f ~/.ssh/net_meter
    # set key to client
    host$ cat ~/.ssh/net_meter.pub | ssh -p <ssh_port> <ssh_user>@<ssh_host> "cat >> ~/.ssh/authorized_keys"
    # test if ssh-key is success
    host$ ssh -p <ssh_port> <ssh_user>@<ssh_host> -i ~/.ssh/net_meter

    # also set host key to client2

``NetMeterConfig.py`` and ``creds.dat``

  - ``shutdown=False``: Will ``NetMeter`` close client at the end of test?
  - key in ``creds.dat`` could't use ``~`` in path.

note
~~~~
- running

  - different buffer size in ``iperf``: ``iperf -l``
  - cpu usage?
  - time control: wait 30 sec
  - TCP/UDP protocol
  - multi-thread

- render plot and html
