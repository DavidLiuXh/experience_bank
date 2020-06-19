* Librdkafka要和kakfa集群通讯, 网络操作肯定是少不了的,这就需要封装transport数据传输层;
* Librdkafka毕竟是SDK, 作为访问kafka集群的客户端,不需要支持大并发, 在网络IO模型 上选用了 [poll](http://man7.org/linux/man-pages/man2/poll.2.html);
* IO模型确定后, 发送和接收数据必不可少的缓冲区buffer, 我们前面已经介绍过, 请参考[Librdkafka的基础数据结构 3 -- Buffer相关 ](https://www.jianshu.com/p/30cd98f04891) ;
* 以上介绍的librdkafka中的poll模型和buffer, 完全可以独立出来, 用在其他项目上, 作者封装得很好;
* Librdkafka与kafka broker间是tcp连接, 在接收数据后就涉及到一个拆包组包的问题, 这个就和kafka的协议有关了, kafka的二进制协议:
    1. 前4个字节是payload长度;
    2. 后面紧接着是payload具体内容,这部分又分为header和body两部分;

* 收包时就可以先收4字节,拿到payload长度, 再根据这个长度收够payload内容, 这样一个完整的response就接收到了;
* 下面我们来结合代码分析一下,  其中有一部分是windows下的实现代码,我们将忽略掉
---
###### rd_kafka_transport_s
*  所在文件: src/rdkafka_transport_int.h srd/rdkafka_transport.c(.h)
* 定义:
```
struct rd_kafka_transport_s {	
	int rktrans_s; // 与broker通讯的socket
	
	rd_kafka_broker_t *rktrans_rkb; // 与之通讯的broker, 一个broker一个tcp连接,也就对应着一个transport对象

#if WITH_SSL
	SSL *rktrans_ssl; //ssl连接句柄
#endif

        // sasl权限验证相关
	struct {
                void *state;               /* SASL implementation
                                            * state handle */

                int           complete;    /* Auth was completed early
					    * from the client's perspective
					    * (but we might still have to
                                            *  wait for server reply). */

                /* SASL framing buffers */
		struct msghdr msg;
		struct iovec  iov[2];

		char          *recv_buf;
		int            recv_of;    /* Received byte count */
		int            recv_len;   /* Expected receive length for
					    * current frame. */
	} rktrans_sasl;

        //  response接收buffer
	rd_kafka_buf_t *rktrans_recv_buf;  /* Used with framed_recvmsg */

        // poll模型使用
        /* Two pollable fds:
         * - TCP socket
         * - wake-up fd
         */
        struct pollfd rktrans_pfd[2];
        int rktrans_pfd_cnt;

        size_t rktrans_rcvbuf_size;    /**< Socket receive buffer size */
        size_t rktrans_sndbuf_size;    /**< Socket send buffer size */
};

typedef struct rd_kafka_transport_s rd_kafka_transport_t
```
* 异步连接到broker `rd_kafka_transport_connect`:
```
rd_kafka_transport_t *rd_kafka_transport_connect (rd_kafka_broker_t *rkb,
						  const rd_sockaddr_inx_t *sinx,
						  char *errstr,
						  size_t errstr_size) {
	rd_kafka_transport_t *rktrans;
	int s = -1;
	int on = 1;
        int r;

        rkb->rkb_addr_last = sinx;

        // 建socket, 可以调用socket_cb, 用用户自定义的方式来得到socket, 默认用socket函数
	s = rkb->rkb_rk->rk_conf.socket_cb(sinx->in.sin_family,
					   SOCK_STREAM, IPPROTO_TCP,
					   rkb->rkb_rk->rk_conf.opaque);
	if (s == -1) {
		rd_snprintf(errstr, errstr_size, "Failed to create socket: %s",
			    socket_strerror(socket_errno));
		return NULL;
	}


#ifdef SO_NOSIGPIPE
	/* Disable SIGPIPE signalling for this socket on OSX */
	if (setsockopt(s, SOL_SOCKET, SO_NOSIGPIPE, &on, sizeof(on)) == -1) 
		rd_rkb_dbg(rkb, BROKER, "SOCKET",
			   "Failed to set SO_NOSIGPIPE: %s",
			   socket_strerror(socket_errno));
#endif

	/* Enable TCP keep-alives, if configured. */
	if (rkb->rkb_rk->rk_conf.socket_keepalive) {
#ifdef SO_KEEPALIVE
		if (setsockopt(s, SOL_SOCKET, SO_KEEPALIVE,
			       (void *)&on, sizeof(on)) == SOCKET_ERROR)
			rd_rkb_dbg(rkb, BROKER, "SOCKET",
				   "Failed to set SO_KEEPALIVE: %s",
				   socket_strerror(socket_errno));
#else
		rd_rkb_dbg(rkb, BROKER, "SOCKET",
			   "System does not support "
			   "socket.keepalive.enable (SO_KEEPALIVE)");
#endif
	}

        /* Set the socket to non-blocking */
        if ((r = rd_fd_set_nonblocking(s))) {
                rd_snprintf(errstr, errstr_size,
                            "Failed to set socket non-blocking: %s",
                            socket_strerror(r));
                goto err;
        }

	/* Connect to broker */
       // 可以用用户自定义的连接方式connect, 回调没set的话调用connect
        if (rkb->rkb_rk->rk_conf.connect_cb) {
                r = rkb->rkb_rk->rk_conf.connect_cb(
                        s, (struct sockaddr *)sinx, RD_SOCKADDR_INX_LEN(sinx),
                        rkb->rkb_name, rkb->rkb_rk->rk_conf.opaque);
        } else {
                if (connect(s, (struct sockaddr *)sinx,
                            RD_SOCKADDR_INX_LEN(sinx)) == SOCKET_ERROR &&
                    (socket_errno != EINPROGRESS
                            ))
                        r = socket_errno;
                else
                        r = 0;
        }

        if (r != 0) {
		rd_snprintf(errstr, errstr_size,
			    "Failed to connect to broker at %s: %s",
			    rd_sockaddr2str(sinx, RD_SOCKADDR2STR_F_NICE),
			    socket_strerror(r));
		goto err;
	}

	/* Create transport handle */
        // 创建并初始化rd_kafka_transport_s
	rktrans = rd_calloc(1, sizeof(*rktrans));
	rktrans->rktrans_rkb = rkb;
	rktrans->rktrans_s = s;
	rktrans->rktrans_pfd[rktrans->rktrans_pfd_cnt++].fd = s;

        // 添加poll事件
        if (rkb->rkb_wakeup_fd[0] != -1) {
                rktrans->rktrans_pfd[rktrans->rktrans_pfd_cnt].events = POLLIN;
                rktrans->rktrans_pfd[rktrans->rktrans_pfd_cnt++].fd = rkb->rkb_wakeup_fd[0];
        }

	/* Poll writability to trigger on connection success/failure. */
	rd_kafka_transport_poll_set(rktrans, POLLOUT);

	return rktrans;

 err:
	if (s != -1)
                rd_kafka_transport_close0(rkb->rkb_rk, s);

	return NULL;
}
```
* 对 poll方法的封装 `rd_kafka_transport_poll`:
```
int rd_kafka_transport_poll(rd_kafka_transport_t *rktrans, int tmout) {
        int r;
        // 调用poll, 失败直接return
	r = poll(rktrans->rktrans_pfd, rktrans->rktrans_pfd_cnt, tmout);
	if (r <= 0)
		return r;

        rd_atomic64_add(&rktrans->rktrans_rkb->rkb_c.wakeups, 1);
        // 没什么实际用处
        if (rktrans->rktrans_pfd[1].revents & POLLIN) {
                /* Read wake-up fd data and throw away, just used for wake-ups*/
                char buf[512];
                if (rd_read((int)rktrans->rktrans_pfd[1].fd,
                            buf, sizeof(buf)) == -1) {
                        /* Ignore warning */
                }
        }
         // 返回poll到的有效事件
         return rktrans->rktrans_pfd[0].revents;
}
```
* transport io 事件循环`rd_kafka_transport_io_serve`:
```
void rd_kafka_transport_io_serve (rd_kafka_transport_t *rktrans,
                                  int timeout_ms) {
	rd_kafka_broker_t *rkb = rktrans->rktrans_rkb;
	int events;

        // 如果没收到response的请求个数没超过最大限制, 并且有需要发送的buf, 就把POLLOUT事件加入
	if (rd_kafka_bufq_cnt(&rkb->rkb_waitresps) < rkb->rkb_max_inflight &&
	    rd_kafka_bufq_cnt(&rkb->rkb_outbufs) > 0)
		rd_kafka_transport_poll_set(rkb->rkb_transport, POLLOUT);

        // poll啊poll
	if ((events = rd_kafka_transport_poll(rktrans, timeout_ms)) <= 0)
                return;
        // 暂时去掉POLLOUT
        rd_kafka_transport_poll_clear(rktrans, POLLOUT);

         //  处理poll到的io events
	rd_kafka_transport_io_event(rktrans, events);
}
```
* IO事件处理 `rd_kafka_transport_io_event`:
broker有一个简单的状态机转换,有如下几个状态:
```
enum {
		RD_KAFKA_BROKER_STATE_INIT,
		RD_KAFKA_BROKER_STATE_DOWN,
		RD_KAFKA_BROKER_STATE_CONNECT,
		RD_KAFKA_BROKER_STATE_AUTH,

		/* Any state >= STATE_UP means the Kafka protocol layer
		 * is operational (to some degree). */
		RD_KAFKA_BROKER_STATE_UP,
                RD_KAFKA_BROKER_STATE_UPDATE,
		RD_KAFKA_BROKER_STATE_APIVERSION_QUERY,
		RD_KAFKA_BROKER_STATE_AUTH_HANDSHAKE
	} rkb_state;
```
根据这个broker当前的状态,有不同的操作和状态转换
```
static void rd_kafka_transport_io_event (rd_kafka_transport_t *rktrans,
					 int events) {
	char errstr[512];
	int r;
	rd_kafka_broker_t *rkb = rktrans->rktrans_rkb;

	switch (rkb->rkb_state)
	{
	case RD_KAFKA_BROKER_STATE_CONNECT:
#if WITH_SSL
		if (rktrans->rktrans_ssl) {
			/* Currently setting up SSL connection:
			 * perform handshake. */
			rd_kafka_transport_ssl_handshake(rktrans);
			return;
		}
#endif

		/* Asynchronous connect finished, read status. */
		if (!(events & (POLLOUT|POLLERR|POLLHUP)))
			return;

		if (rd_kafka_transport_get_socket_error(rktrans, &r) == -1) {
			rd_kafka_broker_fail(
                                rkb, LOG_ERR, RD_KAFKA_RESP_ERR__TRANSPORT,
                                "Connect to %s failed: "
                                "unable to get status from "
                                "socket %d: %s",
                                rd_sockaddr2str(rkb->rkb_addr_last,
                                                RD_SOCKADDR2STR_F_PORT |
                                                RD_SOCKADDR2STR_F_FAMILY),
                                rktrans->rktrans_s,
                                rd_strerror(socket_errno));
		} else if (r != 0) {
			/* Connect failed */
                        errno = r;
			rd_snprintf(errstr, sizeof(errstr),
				    "Connect to %s failed: %s",
                                    rd_sockaddr2str(rkb->rkb_addr_last,
                                                    RD_SOCKADDR2STR_F_PORT |
                                                    RD_SOCKADDR2STR_F_FAMILY),
                                    rd_strerror(r));

			rd_kafka_transport_connect_done(rktrans, errstr);
		} else {
			/* Connect succeeded */
			rd_kafka_transport_connected(rktrans);
		}
		break;

	case RD_KAFKA_BROKER_STATE_AUTH:
		/* SASL handshake */
		if (rd_kafka_sasl_io_event(rktrans, events,
					   errstr, sizeof(errstr)) == -1) {
			errno = EINVAL;
			rd_kafka_broker_fail(rkb, LOG_ERR,
					     RD_KAFKA_RESP_ERR__AUTHENTICATION,
					     "SASL authentication failure: %s",
					     errstr);
			return;
		}
		break;

	case RD_KAFKA_BROKER_STATE_APIVERSION_QUERY:
	case RD_KAFKA_BROKER_STATE_AUTH_HANDSHAKE:
	case RD_KAFKA_BROKER_STATE_UP:
	case RD_KAFKA_BROKER_STATE_UPDATE:

		if (events & POLLIN) {
			while (rkb->rkb_state >= RD_KAFKA_BROKER_STATE_UP &&
			       rd_kafka_recv(rkb) > 0)
				;
		}

		if (events & POLLHUP) {
			rd_kafka_broker_fail(rkb,
                                             rkb->rkb_rk->rk_conf.
                                             log_connection_close ?
                                             LOG_NOTICE : LOG_DEBUG,
                                             RD_KAFKA_RESP_ERR__TRANSPORT,
					     "Connection closed");
			return;
		}

		if (events & POLLOUT) {
			while (rd_kafka_send(rkb) > 0)
				;
		}
		break;

	case RD_KAFKA_BROKER_STATE_INIT:
	case RD_KAFKA_BROKER_STATE_DOWN:
		rd_kafka_assert(rkb->rkb_rk, !*"bad state");
	}
}
```
最主要的是以下两点:
 1. 有可读事件:
```
循环读, 读到不能读为止
if (events & POLLIN) {
			while (rkb->rkb_state >= RD_KAFKA_BROKER_STATE_UP &&
			       rd_kafka_recv(rkb) > 0)
				;
		}
```
`rd_kafka_recv`按kafka的协议来收包, 先收4字节,拿到payload长度, 再根据这个长度收够payload内容, 这样一个完整的response就接收到了
 2.  有可写事件:
```
if (events & POLLOUT) {
			while (rd_kafka_send(rkb) > 0)
				;
		}
```
循环写, 写到不能写为止
* 关闭transport 
```
void rd_kafka_transport_close (rd_kafka_transport_t *rktrans) {
#if WITH_SSL
	if (rktrans->rktrans_ssl) {
		SSL_shutdown(rktrans->rktrans_ssl);
		SSL_free(rktrans->rktrans_ssl);
	}
#endif

        rd_kafka_sasl_close(rktrans);

	if (rktrans->rktrans_recv_buf)
		rd_kafka_buf_destroy(rktrans->rktrans_recv_buf);

	if (rktrans->rktrans_s != -1)
                rd_kafka_transport_close0(rktrans->rktrans_rkb->rkb_rk,
                                          rktrans->rktrans_s);

	rd_free(rktrans);
}
```
* 封装sendmsg方法
```
ssize_t rd_kafka_transport_socket_sendmsg (rd_kafka_transport_t *rktrans,
                                           rd_slice_t *slice,
                                           char *errstr, size_t errstr_size) {
        struct iovec iov[IOV_MAX];
        struct msghdr msg = { .msg_iov = iov };
        size_t iovlen;
        ssize_t r;

       // 从slice产生iovec数组, 然后再发送
        rd_slice_get_iov(slice, msg.msg_iov, &iovlen, IOV_MAX,
                         /* FIXME: Measure the effects of this */
                         rktrans->rktrans_sndbuf_size);
        msg.msg_iovlen = (typeof(msg.msg_iovlen))iovlen;

#ifdef sun
        /* See recvmsg() comment. Setting it here to be safe. */
        socket_errno = EAGAIN;
#endif

        r = sendmsg(rktrans->rktrans_s, &msg, MSG_DONTWAIT
#ifdef MSG_NOSIGNAL
                    | MSG_NOSIGNAL
#endif
                );

        if (r == -1) {
                if (socket_errno == EAGAIN)
                        return 0;
                rd_snprintf(errstr, errstr_size, "%s", rd_strerror(errno));
        }

        /* Update buffer read position */
        rd_slice_read(slice, NULL, (size_t)r);

        return r;
}
```
* 封装 send方法:
```
rd_kafka_transport_socket_send0 (rd_kafka_transport_t *rktrans,
                                 rd_slice_t *slice,
                                 char *errstr, size_t errstr_size) {
        ssize_t sum = 0;
        const void *p;
        size_t rlen;

       // rd_slice_peeker相当于一个iterator接口,每次返回一个slice中有效的segment
        while ((rlen = rd_slice_peeker(slice, &p))) {
                ssize_t r;
                //实际的发送
                r = send(rktrans->rktrans_s, p, rlen, 0
                );

                // 如果非eagain错误的话,是真的有错误发生了,return -1
                if (unlikely(r <= 0)) {
                        if (r == 0 || errno == EAGAIN)
                                return 0;
                        rd_snprintf(errstr, errstr_size, "%s",
                                    socket_strerror(socket_errno));
                        return -1;
                }

                /* Update buffer read position */
               //更新slice的读位置 
                rd_slice_read(slice, NULL, (size_t)r);

                sum += r;

                /* FIXME: remove this and try again immediately and let
                 *        the next write() call fail instead? */
                if ((size_t)r < rlen)
                        break;
        }

        return sum;
}
```
* 封装 recv方法:
```
rd_kafka_transport_socket_recv0 (rd_kafka_transport_t *rktrans,
                                 rd_buf_t *rbuf,
                                 char *errstr, size_t errstr_size) {
        ssize_t sum = 0;
        void *p;
        size_t len;
        
       // rbuf可写就一直写
        while ((len = rd_buf_get_writable(rbuf, &p))) {
                ssize_t r;

                r = recv(rktrans->rktrans_s, p,
                         len,
                         0);

                if (unlikely(r <= 0)) {
                        if (r == -1 && socket_errno == EAGAIN)
                                return 0;
                        else if (r == 0) {
                                /* Receive 0 after POLLIN event means
                                 * connection closed. */
                                rd_snprintf(errstr, errstr_size,
                                            "Disconnected");
                                return -1;
                        } else if (r == -1) {
                                rd_snprintf(errstr, errstr_size, "%s",
                                            rd_strerror(errno));
                                return -1;
                        }
                }

                /* Update buffer write position */
                // 更新buf的写位置
                rd_buf_write(rbuf, NULL, (size_t)r);

                sum += r;

                /* FIXME: remove this and try again immediately and let
                 *        the next recv() call fail instead? */
                if ((size_t)r < len)
                        break;
        }
        return sum;
}
```
---
### [Librdkafka源码分析-Content Table](https://www.jianshu.com/p/1a94bb09a6e6)

