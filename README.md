# mysql_os_metrics
A MySQL plugin for displaying Operating System metrics in INFORMATION_SCHEMA.  This would allow monitoring tools, such as PMM, to retrieve these values remotely via the MySQL interface.  This is just a Proof of Concept (POC) at this point in time and needs to be expanded greatly.

Values are pulled via standard C library calls so overhead is absolutely minimal.  I added a couple of libraries originally to show that even Windows and other variants of UNIX can be utilized, but commented them out to keep it all simple for now.

Many variables were added to show what was possible.  Some of these may not be of interest.  I just wanted to see what kind of stuff was possible and would tweak these over time.  

Also, none of the calculations were rounded.  This was done just to keep precision for graphing of values but could easily be canged later.

If there is interest, this could be expanded to add more metrics and unnecessary ones removed.  Just looking for feedback.

Also keep in mind that my C programming skills are rusty and I am sure the code could be cleaned up.

Make sure you have the source code for MySQL and have done a cmake on it.  This will be necessary to compile the plugin.

#### Compiling the Plugin
Below is the way that I compiled the plugin.  You will obviously need to make changes to match your environment.
You will also need to have the Percona Server source code on your server:

    wget https://www.percona.com/downloads/Percona-Server-5.7/Percona-Server-5.7.17-13/source/tarball/percona-server-5.7.17-13.tar.gz

I also had to add a few utilities:

    sudo yum install cmake
    sudo yum install boost
    sudo yum install ncurses-devel
    sudo yum install readline-devel
    cmake -DDOWNLOAD_BOOST=1 -DWITH_BOOST=.. 

Once the above was complete, I was able to compile with something like the following:

    SRCBASE="/home/ec2-user/percona-server-5.7.17-13"
    g++ -DMYSQL_DYNAMIC_PLUGIN -Wall -fPIC -shared \
    -I/usr/include/mysql -m64 \
    -I${SRCBASE}/sql \
    -I${SRCBASE}/include \
    -I${SRCBASE}/libbinlogevents/export \
    -I${SRCBASE}/libbinlogevents/include \
    -I/home/ec2-user/mysql_os_metrics-master \
    -o osmetricsplugin.so osmetricsplugin.cc

#### Plugin Installation
Copy the resulting osmetricsplugin.so file to the location of plugin_dir in MySQL.  You will need to know where your plugin directory is.  You can query that with the following SQL:

    mysql> SHOW GLOBAL VARIABLES LIKE "%plugin_dir%";
    +---------------+-------------------------+
    | Variable_name | Value                   |
    +---------------+-------------------------+
    | plugin_dir    | /jet/var/mysqld/plugin/ |
    +---------------+-------------------------+
    1 row in set (0.01 sec)

You will need to copy the compiled plugin to this directory:

    cp -f osmetricsplugin.so /jet/var/mysqld/plugin/

Finally, you can login to MySQL and activate the plugin:

    mysql> INSTALL PLUGIN OS_METRICS SONAME 'osmetricsplugin.so';

#### Verify Installation
    mysql> SELECT * FROM information_schema.PLUGINS WHERE PLUGIN_NAME LIKE "%OS%";;
    +-------------+----------------+---------------+--------------------+---------------------+--------------------+------------------------+-----------------+-------------------------------------+----------------+-------------+
    | PLUGIN_NAME | PLUGIN_VERSION | PLUGIN_STATUS | PLUGIN_TYPE        | PLUGIN_TYPE_VERSION | PLUGIN_LIBRARY     | PLUGIN_LIBRARY_VERSION | PLUGIN_AUTHOR   | PLUGIN_DESCRIPTION                  | PLUGIN_LICENSE | LOAD_OPTION |
    +-------------+----------------+---------------+--------------------+---------------------+-------------------+------------------------+-----------------+-------------------------------------+----------------+-------------+
    | OS_METRICS  | 1.0            | ACTIVE        | INFORMATION SCHEMA | 50724.0             | osmetricsplugin.so | 1.7                    | Michael Patrick | OS Metrics INFORMATION_SCHEMA table | GPL            | ON          |
    +-------------+----------------+---------------+--------------------+---------------------+--------------------+------------------------+-----------------+-------------------------------------+----------------+-------------+
    1 row in set (0.00 sec)

#### Query the Plugin   
    mysql> SELECT * FROM INFORMATION_SCHEMA.OS_METRICS;
    +-----------------------+--------------------+----------------------------------------------------+
    | NAME                  | VALUE              | COMMENT                                            |
    +-----------------------+--------------------+----------------------------------------------------+
    | cpu_user              |              22872 | Normal processes executing in user mode            |
    | cpu_nice              |                 15 | Niced processes executing in user mode             |
    | cpu_sys               |               7192 | Processes executing in kernel mode                 |
    | cpu_idle              |            3726031 | Processes which are idle                           |
    | cpu_iowait            |               1413 | Processes waiting for I/O to complete              |
    | cpu_irq               |                  0 | Processes servicing interrupts                     |
    | cpu_softirq           |                 17 | Processes servicing Softirqs                       |
    | cpu_guest             |               1295 | Processes running a guest                          |
    | cpu_guest_nice        |                  0 | Processes running a niced guest                    |
    | cpu_idle_pct          |   99.1614460524705 | Average CPU Idleness                               |
    | cpu_util_pct          | 0.8385539475295047 | Average CPU utilization                            |
    | datadir_size          |         8318783488 | MySQL data directory size                          |
    | datadir_size_free     |         2314506240 | MySQL data directory size free space               |
    | datadir_size_used     |         6004277248 | MySQL data directory size used space               |
    | datadir_size_used_pct |  72.17734728474761 | MySQL data directory used space as a percentage    |
    | total_ram             |         2090319872 | Total usable main memory size                      |
    | free_ram              |         1573638144 | Available memory size                              |
    | used_ram              |          516681728 | Used memory size                                   |
    | free_ram_pct          |  75.28216552734375 | Available memory as a percentage                   |
    | used_ram_pct          | 24.717830657958984 | Free memory as a percentage                        |
    | shared_ram            |              61440 | Amount of shared memory                            |
    | buffer_ram            |           63504384 | Memory used by buffers                             |
    | total_high_ram        |                  0 | Total high memory size                             |
    | free_high_ram         |                  0 | Available high memory size                         |
    | total_low_ram         |         2090319872 | Total low memory size                              |
    | free_low_ram          |         1573638144 | Available low memory size                          |
    | load_1min             |        0.080078125 | 1 minute load average                              |
    | load_5min             |       0.0166015625 | 5 minute load average                              |
    | load_15min            |      0.00537109375 | 15 minute load average                             |
    | uptime                |              37612 | Uptime (in seconds)                                |
    | uptime_days           |                  0 | Uptime (in days)                                   |
    | uptime_hours          |                 10 | Uptime (in hours)                                  |
    | total_swap            |                  0 | Total swap space size                              |
    | free_swap             |                  0 | Swap space available                               |
    | used_swap             |                  0 | Swap space used                                    |
    | free_swap_pct         |                  0 | Swap space available as a percentage               |
    | used_swap_pct         |                  0 | Swap space used as a percentage                    |
    | procs                 |                119 | Number of current processes                        |
    | uptime_tv_sec         |                  0 | User CPU time used (in seconds)                    |
    | utime_tv_usec         |             949712 | User CPU time used (in microseconds)               |
    | stime_tv_sec          |                  0 | System CPU time (in seconds)                       |
    | stime_tv_usec         |             573801 | System CPU time (in microseconds)                  |
    | utime                 |           0.949712 | Total user time                                    |
    | stime                 |           0.573801 | Total system time                                  |
    | maxrss                |             148936 | Maximum resident set size                          |
    | maxrss_bytes          |          152510464 | Maximum resident set size (in bytes)               |
    | minflt                |              34695 | Page reclaims (soft page faults)                   |
    | majflt                |                  0 | Page faults                                        |
    | inblock               |               7680 | Number of block input operations                   |
    | oublock               |              33720 | Number of block output operations                  |
    | nvcsw                 |              21786 | Number of voluntary context switches               |
    | nivcsw                |                 72 | Number of involuntary context switches             |
    | lo_tx_packets         |               9976 | Number of network packets sent                     |
    | lo_rx_packets         |               9976 | Number of network packets received                 |
    | lo_tx_bytes           |             519036 | Number of bytes sent over the network interface    |
    | lo_rx_bytes           |             519036 | Number of bytes received over the network interfac |
    | lo_tx_dropped         |                  0 | Number of received network packets dropped         |
    | lo_rx_dropped         |                  0 | Number of sent network packets dropped             |
    | lo_tx_errors          |                  0 | Number of errors on received packets               |
    | lo_rx_errors          |                  0 | Number of errors on sent packets                   |
    | eth0_tx_packets       |              70865 | Number of network packets sent                     |
    | eth0_rx_packets       |             109186 | Number of network packets received                 |
    | eth0_tx_bytes         |           13948955 | Number of bytes sent over the network interface    |
    | eth0_rx_bytes         |            8672754 | Number of bytes received over the network interfac |
    | eth0_tx_dropped       |                  0 | Number of received network packets dropped         |
    | eth0_rx_dropped       |                  0 | Number of sent network packets dropped             |
    | eth0_tx_errors        |                  0 | Number of errors on received packets               |
    | eth0_rx_errors        |                  0 | Number of errors on sent packets                   |
    +-----------------------+--------------------+----------------------------------------------------+
    68 rows in set (0.00 sec)

#### Plugin Uninstallation
    mysql> UNINSTALL PLUGIN OS_METRICS;
    
#### Future Additions
Below are some ideas for possible additions.
* Swappiness Setting
* CPU Utilization
* Some data in iostat and vmstat (key metrics)
* I/O Scheduler Setting & other OS metrics (maybe useful for checking without having to login to terminal)
* And more...
