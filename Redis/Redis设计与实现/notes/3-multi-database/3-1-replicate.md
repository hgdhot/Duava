### 复制

1. redis的复制功能分为同步和命令传播。

   - 同步

     当客户端向服务器发送slaveof时要求从服务器复制主服务器时，从服务器首先需要同步，将从服务器的数据库状态更新至主服务器当前状态。

     具体步骤：

     （1）从服务器向主服务器发送sync命令。

     （2）主服务器收到sync后执行bgsave，生成RDB文件，并使用一个缓冲区记录从现在开始执行的所有写命令。

     （3）bdsave执行完毕后，主服务器将生成的RDB文件发送给从服务器，从服务器接收并载入该文件，并将自己的数据库状态更新至主服务器执行bgsave时的状态。

     （4）主服务器将缓冲区中所有写命令发送给从服务器，从服务器执行这些命令，将数据库状态更新至主服务器当前状态。

   - 命令传播

     同步操作仅仅保证了主从服务器数据库状态的暂时一致，当主服务器状态被修改后，主服务器会通过命令传播的方式，即将造成主从服务器不一致的写命令发送给从服务器执行，使主从服务器状态再次回到一致状态。

2. 旧版redis（redis2.8以前）复制功能的缺陷

   对于初次复制来说，旧版复制功能可以很好完成任务，对于断线后重新复制来说，旧版复制功能存在严重性能问题。因为对于断线重连复制的情况，主从服务器真正不一致的状态需要同步的是断线期间的写命令，而断线前的数据是一直保持一致的，不需要同步的。但是旧版复制依然会发送sync，生成整个数据库的完整RDB文件，而sync是一个很消耗资源的操作（bgsave需要消耗大量CPU、内存、磁盘IO，发送rdb文件会占用网络资源，从服务器载入RDB文件会阻塞命令请求），所以旧版复制在对于处理断线后重复制的处理非常低效。

3. 新版redis（redis2.8及以后）

   - 新版redis采用psync代替sync执行复制时的同步操作。psync具有完整重同步和部分重同步两种模式。完整重同步用于初次复制，和sync的步骤基本一样，都是通过让主服务器发送RDB文件以及缓冲区中的写命令实现同步。部分重同步用于处理断线后重复制情况，主服务器只将断线期间的写命令发送给从服务器，从服务器就可以将数据库状态更新至主服务器当前状态。

   - 部分重同步实现

     部分重同步由三部分构成：主、从服务器的复制偏移量，主服务器的复制积压缓冲区，服务器runid。

     （1）主服务器每次向从服务器传送N个字节，就将自己的复制偏移量+N。从服务器每次收到主服务器的N个字节，就将自己的复制偏移量+N。所以如果主从服务器处于一致状态，主从服务器的偏移量总是相同的。如果不相同说明主从服务器处于不一致状态。

     （2）主服务器会维护一个复制积压缓冲区（一个固定长度先进先出队列，默认大小1M），当主服务器进行命令传播时，不仅会将命令发送给所有从服务器，还会将写命令入队缓冲区，缓冲区会记录每个字节的偏移量。当从服务器通过psync将自己的偏移量offset发送给主服务器，如果offset之后的数据还存在于复制积压缓冲区，那么主服务器将会对从服务器执行部分重同步。如果offset之后的数据已经不存在于复制积压缓冲区，则执行完整重同步。可以使用repl-backlog-size选项修改复制积压缓冲区大小。

     （3）从服务器对主服务器初次复制时，主服务器会将自己的runid发送给从服务器，从服务器会保存起来。当从服务器断线重连后，从服务器通过向主服务器发送之前保存的主服务器runid，确定完整重同步还是部分重同步。如果发送的runid和主服务器runid一致，说明是原来的主服务器，执行部分重同步，否则执行完整重同步。

   - psync实现

     （1）如果从服务器从来没有复制过任何主服务器或者执行过slaveof no one，从服务器会先向主服务器发送psync ? -1主动请求完整重同步。如果已经复制过主服务器，从服务器向主服务器发送psync <上一次复制的主服务器runid> <从服务器当前offset>。主服务器根据参数判断执行完整重同步还是部分重同步。

     （2）如果主服务器返回+FULLRESYNC <主服务器runid> <主服务器当前偏移量>，表示将执行完整重同步。如果返回+CONTINUE，则执行部分重同步。如果返回-ERR，说明主服务器版本低于redis2.8，不支持psync操作。

4. 主从复制实现

   - 从服务器收到slaveof命令时，首先将参数对应的主服务器的ip、port信息保存在redisServer结构的masterhost和masterport属性中，并立刻向客户端返回OK回复（slaveof是一个异步命令），真正的复制操作在返回回复后执行。
   - 从服务器根据主服务器信息与主服务器建立套接字连接，这时可以将从服务器看成主服务器的一个客户端。接着从服务器会向主服务器发送ping，如果返回pong则说明可以继续执行复制工作。如果返回错误或者超时，则会重新创建套接字连接。
   - 如果从服务器设置了masterpass则需要发送auth进行身份验证。如果主从服务器都没有设置requirepass和masterpass，或者从服务器发送auth的masterpass和主服务器的requirepass相同，则继续复制工作。如果主从服务器设置了不同密码，或者主服务器设置了密码但从服务器没有设置密码，或者主服务器没有设置密码但从服务器设置了密码的情况都会返回错误，并且重新创建套接字连接重新执行复制。
   - 从服务器发送replconf listening-port <从服务器监听端口>，向主服务器发送从服务器监听端口。主服务器收到该端口后，将其保存在服务器状态中的客户端链表中该从服务器所对应的客户端状态结构中的slave_listening_port中，这个会在执行info replication时打印。
   - 从服务器发送psync给主服务器，将自己的数据库状态更新至主服务器当前状态。在同步操作完成后，主服务器也会成为从服务器的客户端，因为主服务器在执行完整重同步或者部分重同步时，都需要作为客户端将缓冲区或者复制积压缓冲区中保存的写命令发送给从服务器执行。
   - 完成同步后，主从服务器进入命令传播阶段，主服务器会一直将自己执行的写命令发送给从服务器执行，保证主从服务器一直保持一致状态。

5. 心跳检测

   命令传播过程中，从服务器默认以每秒1次向主服务器发送心跳replconf ack <从服务器当前的偏移量>。

   作用：

   - 可以检测主从服务器的网络连接状态。info replication列出的slave列表中每项中的lag表示心跳延迟时间（秒数）。如果超过1秒，主从服务器的网络连接出现问题。

   - 通过min-slaves-to-write和min-slaves-max-lag选项防止主服务器在不安全情况下执行写命令。

     min-slaves-to-write表示少于指定的slave数量，主服务器拒绝执行写命令。

     min-slaves-max-lag表示min-slaves-to-write指定的slave数都大于等于min-slaves-max-lag指定的时间时，主服务器拒绝执行写命令。

   - 网络问题导致主服务器发送的命令数据丢失，通过心跳，主服务器可以对比从服务器发送的偏移量，在复制积压缓冲区中找到并重发丢失命令数据。补发缺失命令数据和部分重同步的原理基本一致，区别在于，部分重同步是断线重连情况，而补发缺失命令数据是主从服务器没有断线，但是命令丢失。