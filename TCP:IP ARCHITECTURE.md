## TCP/IP ARCHITECTURE IN LINUX

### KEEPALIVE TIMER

The keepalive timer is used by TCP to probe the peer when there is no activity over the connection for a long time. This timer is used by interactive TCP connections where the connection may be in an idle state for a long time — for example, telnet,rlogin, and so on. Connections need to probe their peers by sending a TCP segment.The segment is sent with sequence number 1 less than the the highest acknowledged sequence number. When this segment reaches the other end, it should generate an ACK immediately thinking that it was retransmission Once the ACK to the keepalive probe is received, we are sure that the peer is alive; otherwise we know that there is a problem. Let ’ s see how this timer is implemented in Linux. 

#### When Is Keepalive Timer Activated? 

On Linux, the keepalive timer implements both a SYN ACK timer and a keepalive
timer. This means that for any of these timers, we reset the same timer, that is, tp →timer . In this section we will only focus on the keepalive timer. The timer is started when a new connection is established in tcp_create_openreq_child() , only if the KEEP ALIVE option ( tp → keepopen ) is enabled for the socket. This is done when an application issues the SO_KEEPALIVE socket option on the socket. This option is not enabled by default, which also means that the keepalive timer is not enabled for all the TCP connections by default. 

####  How Is the Timer Reset? 

The timer is reset by calling tcp_reset_keepalive_timer() , which kicks off the keepalive timer registered as tp → timer for the TCP connection. This timer is initialized as tcp_keepalive_timer in tcp_init_xmit_timers() at the time of opening a socket. 

###  tcp _ keepalive _ timer ()

Let ’ s see how the keepalive timer functions. It fi rst looks for the user of the socket.If so, we need to let the user of the socket complete its task and defer execution of the timer at some later time. We reset keepalive timer by calling tcp_reset_keepalive_timer() to expire after HZ/20 ticks at line 584, release socket hold and leave (cs 10.29a ). The keepalive callback routine can act as a SYN - ACK timer by calling tcp_synack_timer() at line 589 to manage incoming connection request (discussed in Section 10.6.3 ), in case it is a listening socket. Next we check if the socket is in the FIN_WAIT2 state, and the socket is already closed at line 593. If that is the case, we call tcp_time_wait() in case we have not expired TCP_TIMEWAIT_LEN number of ticks.Otherwise if we have expired, we send out reset on the connection and remove the connection from our end. TIME_WAIT timer will be discussed in Section 10.7.2 .

Next we check if the keepalive connection is not enabled (tp → keepalive) or
the connection is in the closed state at line 606. If any of the conditions is TRUE, we release socket lock and return. We send the keepalive probe only if the segment has been idle for some time. So, next we check if any data segment was transmitted which is still unacknowledged ( tp → packets_out is nonzero) or if there is anything in the send queue that needs to be sent next ( tp → send_head != NULL ) at line 612. If any of these conditions is TRUE, we reset the keepalive timer by calling tcp_reset_keepalive_timer() at line 642,release the socket lock, and leave (cs 10.29b ).

If we are here, we are eligible for sending out the keepalive probe if the time
has actually expired. First we calculate the time elapsed since the last segment was received at line 615. Next we compare if the time since last segment was received has exceeded the probe time interval at line 617.keepalive_time_when() gets us probe time interval. The keepalive probe time interval is tp → keepalive_time in case it is set using socket options; otherwise it is sysctl_tcp_keepalive_time . If the timer has not expired, we calculate the next expiry as the time left for the keepalive timer to expire at line 635 and would reset the probe timer to expire in the near future.
Otherwise, if the time has actually expired, the next check would be to see if the number of unacknowledged probes has exceeded the limit at lines 618 – 619. We increment tp → probes_out whenever the probe is sent out (is discussed ahead), and the counter is reset when we get an ACK when no outstanding unacknowledged data are there in the queue (see Section 10.4 ). If we have exceeded probe limits,the reset segment is sent out by calling tcp_send_active_reset() and the connection is closed, lines 620 – 621. In this case, we release the socket lock and leave.

If we have not exceeded the limit on the number of unacknowledged probes,
we call tcp_write_wakeup() to send out a probe (see Section 10.3.7 ). If the probe segment is transmitted successfully, we increment the probe counter by 1 at line 625. 