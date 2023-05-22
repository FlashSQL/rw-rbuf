# RW and R-Buf

## Publication
- Mijin An, Soojun Im, Dawoon Jung, Sang-Won Lee, "Your Read is Our Priority in Flash Storage", VLDB'22, Sydney, September 2022
- [Paper Link](https://dl.acm.org/doi/10.14778/3538598.3538612)

## RW interface for MySQL

**RW** is **an intuitive and explicit storage command to address read stalls**. Upon read stall, both read and write can be requested to the flash storage in a single call. And the read can be processed as soon as the dirty page is copied to the storage buffer cache using DMA, without waiting for the NAND write to finish. By replacing two I/O calls for write and read upon read stalls with one call, RW can simplify the structure of the buffer manager in DBMS and reduce the number of I/O interrupts and consequently the number of context switches.

### Structure

The major modifications on MySQL are made in directories below:

- [`storage/innobase/buf`](mysql/storage/innobase/buf): The database buffer implementation for InnoDB
- [`storage/innobase/fil`](mysql/storage/innobase/fil): File I/O operations for InnoDB
- [`storage/innobase/include`](mysql/storage/innobase/include): The collection of header files for InnoDB

RW consists of the following components:

- Basic modules for the buffer manager to utilize the RW command (e.g., buffer initialization, buffer allocation, buffer replacement, etc.)
  - [`buf0buf.cc`](mysql/storage/innobase/buf/buf0buf.cc)
- The module to send the RW NVM command set to storage
  - [`rw0rw.cc`](mysql/storage/innobase/rw/rw0rw.cc), [`rw0rw.h`](mysql/storage/innobase/include/rw0rw.h)
  - [`buf0flu.cc`](mysql/storage/innobase/buf/buf0flu.cc)
  - [`fil0fil.cc`](mysql/storage/innobase/fil/fil0fil.cc)
  - [`os0file.cc`](mysql/storage/innobase/os/os0file.cc)
- The module to proceed the transaction immediately using the read data buffer without a separate free buffer acquisition procedure
  - [`buf0rea.cc`](mysql/storage/innobase/buf/buf0rea.cc)
  - [`buf0lru.cc`](mysql/storage/innobase/buf/buf0lru.cc)

You can check the modified code by searching for the `#ifdef RW_CMD` keyword in the file. For example:

> `storage/innobase/fil/fil0fil.cc`

```cpp
#ifdef RW_CMD
...
fil_mutex_enter_and_prepare_for_rw_io(
/*===============================*/
  ulint space_id, /*!< in: space id to write */
  ulint space_id_to_read /*!< in: space id to read */
)
{
  fil_space_t*  space;
    ...
#endif /* RW_CMD */
```

## RW and R-Buf firmware for Cosmos+ OpenSSD

**R-Buf** is a read-dedicated in-storage buffer. **RW** and **R-Buf** are complementary, so we combine two techniques to eliminate the read stall problem in both host and storage buffer. To evaluate the performance of the combined technique, we implement them on Comsos+ OpenSSD firmware.

### Modifications

1. RW

- [`custom_io_cmd_completion.c`](cosmos-firmware/custom_io_cmd_completion.c)
- [`custom_io_cmd_completion.h`](cosmos-firmware/custom_io_cmd_completion.h)

2. R-Buf

- [`data_buffer_ccmd.c`](cosmos-firmware/data_buffer_ccmd.c)
- [`data_buffer_ccmd.h`](cosmos-firmware/data_buffer_ccmd.h)

## Build and install

### Pre-requisites

- libreadline

```bash
$ sudo apt-get install libreadline6 libreadline6-dev
```

- libaio

```bash
$ sudo apt-get install libaio1 libaio-dev
```

- etc.

```bash
$ sudo apt-get install build-essential cmake libncurses5 libncurses5-dev bison
```

### Build MySQL

1. Clone the source code: 

```bash
$ git clone https://github.com/meeeejin/rw-rbuf.git
$ cd rw-rbuf/mysql
```

2. Then build and install the source code:
(8: # of cores in your machine)

```bash
$ cmake -DCMAKE_INSTALL_PREFIX=/path/to/basedir
$ make -j8 install
```

For example:

```bash
$ cmake -DCMAKE_INSTALL_PREFIX=/home/mijin/rw-rbuf/mysql
$ make -j8 install
```

3. Initialize the data directory, including the *mysql* database:
- `--datadir` : the path to the MySQL data directory
- `--basedir` : the path to the MySQL installation directory

```bash
$ cd scripts
$ sudo chmod +x mysql_install_db
$ ./mysql_install_db --user=mysql --datadir=/path/to/datadir --basedir=/path/to/basedir
```

For example:

```bash
$ ./mysql_install_db --user=mysql --datadir=/home/mijin/test-data --basedir=/home/mijin/rw-rbuf/mysql
```

4. Open `.bashrc` and add MySQL installation path to your path:

```bash
$ vi ~/.bashrc

export PATH=/path/to/basedir/bin:$PATH
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/path/to/basedir/lib/

$ source ~/.bashrc
```

For example:

```bash
export PATH=/home/mijin/rw-rbuf/mysql/bin:$PATH
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/mijin/rw-rbuf/mysql/lib/
```

5. Create a configuration file (`my.cnf`) for the MySQL server or use the [uploaded configuration file](mysql/my.cnf). You should modify the path `/path/to/datadir` to your local path. For details on each variable, refer to [MySQL Server System Variable](https://dev.mysql.com/doc/refman/5.6/en/server-system-variable-reference.html)

6. You can start the MySQL server using the below command:

```bash
$ ./bin/mysqld_safe --defaults-file=/path/to/my.cnf
```

7. You can shut down the server using the below command:

```bash
$ ./bin/mysqladmin -uroot shutdown
```

### Build Cosmos+ OpenSSD Firmware

1. Refer to p70-137 of [slides](http://crztech.iptime.org:8080/Release/OpenSSD/Cosmos-plus-OpenSSD/Cosmos%2B%20OpenSSD%202017%20Tutorial.pdf) to build the basic Cosmos+ OpenSSD firmware.

2. Copy source files in `rw-rbuf/cosmos-firmware` to `src` folder in project explorer (See p107 in the slides above).

3. Build firmware.
