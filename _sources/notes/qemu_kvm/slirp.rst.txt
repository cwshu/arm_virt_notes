QEMU Slirp Network Backend
==========================

``Slirp`` APIs in ``libslirp.h``

- ``slirp_input()``: slirp 透過硬體來 send packet
- ``slirp_output()``: 把 packet 送回 guest, 透過 peer 的 ``receive()`` function (``qemu_send_packet()``)

::

  struct Slirp
  
  - void* opaque = (SlirpState*) s; // net_slirp_init()
  - libslirp.h
  
    - functions
      
      - slirp_init()
      - slirp_input()
      - slirp_pollfds_poll(), slirp_pollfds_fill()
      - slirp_add_hostfwd(): socreate() Slirp's socket
  
    - library user should provide some functions (like callback?)
  
      - slirp_output(): [net/slirp.c] qemu_send_packet(SlirpState->nc)
    

``SlirpState`` wraps ``Slirp`` API, and provide ``NetClientInfo`` interface for QEMU Network architecture.
::
  
  struct SlirpState
  
  - Slirp* slirp: slirp wrapper in qemu net framework
  
    - slirp_hostfwd() => Slirp::slirp_add_hostfwd()
    - slirp_guestfwd() => Slirp::slirp_add_guestfwd()
    - slirp_smb() => Slirp::slirp_add_exec()
  
  - NetClientState nc: interface NetClientState
  
    - nc = qemu_new_net_client(&net_slirp_info, ... )
  
      - nc->incoming_queue->deliver = qemu_deliver_packet_iov() [1.]
      - nc->info = net_slirp_info; [2.]
      - nc->peer = peer; // external parameter [3.]
  
    - nc source
  
      1. qemu_new_net_client() default 
      2. struct NetClientInfo net_slirp_info; (slirp provide NetClientInfo interface)
  
         - .receive = net_slirp_receive; net_slirp_receive() calls Slirp::slirp_input();
         - .cleanup = net_slirp_cleanup; net_slirp_cleanup() calls SlirpState::slirp_smb_cleanup();
  
      3. from net_slirp_init() function parameters: peer is customizable by function caller.
  
socket address mapping(NAPT) and host socket API abstraction
::

  struct socket (Slirp socket)
  
  - struct socket *so_next, *so_prev: linked list of sockets
  - int s: actual socket fd number
  - Slirp* slirp: managing Slirp instance
  - struct sbuf so_rcv, so_snd: Recv/Send buffer
  
  - socreate(Slirp* slirp): 
  
    - so->slirp = slirp
    - so->s = socket() syscall
  
  - callers of send() and recv() syscall
  
    - soread(), sowrite()
    - sosendoob(), sorecvoob()
    - sosendto(), sorecvfrom()
    - slirp_send(struct socket *so, const void *buf, size_t len, int flags): send(so->s, buf, len, flags);

.. _slirp_input:

slirp_input()
~~~~~~~~~~~~~

``slirp_input()`` use system call to send packet to remote OS. 
for loopback network, slirp_input() may send packet back to peer network.

at least 4 path of ``slirp_input()``::

    1. __libc_send()
    => slirp_input() => slirp_send() => __libc_send()

    2. sosendto(): call sendto() syscall, only udp.c & udp6.c use it.
    => slirp_input() => sosendto() => sendto() # syscall

    3. slirp_output()
    => slirp_input() => tcp_input() => tcp_output() => slirp_output() 
    => qemu_send_packet() => e1000_receive_iov()

    4. slirp_output() by arp?
    => slirp_input() => arp_input() => slirp_output() 

- slirp_send() callers

  - sosendoob()
  - sowrite()
  - sbappend()
  - ``slirp/socket.c`` and ``slirp/sbuf.c``

Misc
----

- struct: NICState, NICInfo, NICPeers
- caller of ``slirp_output()``

  - ``slirp_pollfds_poll() => tcp_output() => slirp_output()``
  - ``ra_timer_handler() => ndp_send_ra() => slirp_output()``
