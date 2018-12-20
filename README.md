# UDP
![](https://github.com/zcq-zq/UDP/blob/master/pic/1.png)<br>

		Linux内核中协议栈中，UDP报文发送的主要流程如下：
    inet_sendmsg()-->
          udp_sendmsg() -->
    udp_sendmsg()主要流程如下：
    1）前期处理。包括，对数据长度合法性判断、pending数据的判断、目的地址的处理和获取、控制信息的处理、组播处理、connected信息处理、MSG_CONFIRM标志的处理等。
    2）调用ip_append_data()接口将其添加到传输控制块(sock)的发送队列中(利用发送队列中的现有skb，或者新创建skb，详细原理和流程请参见ip_append_data()接口的分析)。
    3）判断是否有cork标记(MSG_MORE)，如果没有，则说明需要立即发送，则调用udp_push_pending_frames()接口发送报文，实际是将包提交至IP层；如果设置了cork，则说明需要阻塞等待直到数据达到MTU大小，则完成本次的udp_sendmsg()处理。
    
    UDP报文发送的关键处理函数: __skb_recv_datagram() 
```
    do {
		
		int _off = *off;

		last = (struct sk_buff *)queue;
		spin_lock_irqsave(&queue->lock, cpu_flags);
		skb_queue_walk(queue, skb) {
			last = skb;
			*peeked = skb->peeked;
			if (flags & MSG_PEEK) {
				if (_off >= skb->len && (skb->len || _off ||
							 skb->peeked)) {
					_off -= skb->len;
					continue;
				}

				skb = skb_set_peeked(skb);
				error = PTR_ERR(skb);
				if (IS_ERR(skb))
					goto unlock_err;

				atomic_inc(&skb->users);
			} else
				__skb_unlink(skb, queue);

			spin_unlock_irqrestore(&queue->lock, cpu_flags);
			*off = _off;
			return skb;
		}
		spin_unlock_irqrestore(&queue->lock, cpu_flags);

		if (sk_can_busy_loop(sk) &&
		    sk_busy_loop(sk, flags & MSG_DONTWAIT))
			continue;		

		error = -EAGAIN;
		if (!timeo)		//非阻塞模式 			goto no_packet;

	} while (!wait_for_more_packets(sk, err, &timeo, last));

	return NULL;
```
