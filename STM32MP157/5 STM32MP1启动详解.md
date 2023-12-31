# STM32MP1启动详解

## STM32MP1启动模式


#### 单片机启动方式

1. SMT32单片机，
2. 在集成IDE上编写代码，调试
3. 烧写进入单片机内
4. 上电即可运行

#### 能跑Linux的芯片，启动方式很复杂

1. linux系统相对于单片机flash**更大**，例如H7系列内部flash有2M，有的甚至有8M的flash。linux大概在几十甚至上百M。功能强大、完善，就有几个G
2. 因此linux芯片需要借助**外部flash存放镜像**，比如==SD卡、EMMC、NAND Flash、NOR Flash、SPI Flash==
3. 半导体厂商会提供相应的**烧写工具**
4. 同一款芯片会同时支持多种flash启动。因此在上电启动时，就需要判断从哪个设备加载镜像

#### MP1支持的启动设备
EMMC，SD卡，NAND，NOR，==USB，UART（串行启动）==

#### MP1如何判断从哪里启动镜像
1. ![MP1 ROM空间](MP1ROM%E7%A9%BA%E9%97%B4.png)

如上图，MP1启动区ROM（CA7）有128KB，供ST自己写代码，其重要功能就是判断加载哪个外部flash

2. ![Alt text](MP1%E5%BA%95%E6%9D%BFBOOT.png)

如上图左，在底板上有三个BOOT引脚，BOOT0、1、2，通过改变引脚的高低电平可以选择启动模式（上图右）

3. ==MP1的引脚有复用功能==
   >比如SDMMC2的D0引脚，可以使用PB14或者PE6。不过，MP1内部ROM代码==已经有了默认的引脚==，比如上述默认使用PB14，如果自己画板子用了PE6，那么就会有问题。

   >也可以通过修改OTP（One-Time Programmable memory）来设置启动引脚，**<font color=red>但是不建议这么做，容易使芯片报废</font>**

4. **<font color=red>MP157底板的拨码盘上1下0，从左到右是BOOT0、BOOT1、BOOT2</font>**


------

## STM32MP1启动流程详解

#### 内部ROM代码
ST官网link： <font size=5>[STM32MP15x ROM Code](https://wiki.stmicroelectronics.cn/stm32mpu/wiki/Category:ROM_code)</font>


#### ROM代码启动流程

![Alt text](<ROM code boot flow.png>)





#### ROM代码启动的硬件选择流程

<img src="ROM code boot device selection.png" alt="Alt text" style="zoom:200%;" />





#### STM32MP157 启动模式

*  启动模式选择
    * Serial boot    串口启动
    * Flash memory boot    闪存启动
    * Engi boot (developer mode)    开发者模式
    * SSP (secure secrets provisioning)
    * RMA (return material analysis)
   * Image loading binary size (without STM32 header)     无STM32头文件的镜像启动
* Secure boot      安全模式
    * FSBL authentication (ECDSA)
    * FSBL authentication key revocation
    * FSBL anti-rollback mechanism
    * FSBL decryption (AES)
    * Cryptographic accelerators (ECDSA, AES)
* Secondary core boot 二级核心启动
* Wake  up from low power modes
* Secure services export

**<font color=blue>#FSBL=First Stage Boot Loader</font>**
**<font color=blue>#SSBL=Second Stage Boot Loader</font>**




#### 安全启动流程

本课程中主要使用安全模式进行启动

1. ROM代码从选定的Flash设备中加载FSBL镜像文件，一般为TF-A镜像。但是可以换成我们自己写的程序，不过不能用随便的什么bin文件，而是需要一个头部信息，即STM32 header
2. FSBL加载后，要进行鉴权(authentication)
3. 鉴权成功，就会跳转到FSBL镜像入口地址，开始运行FSBL固件
4. 此时DDR还没有初始化，所以==FSBL只有在RAM中运行==，STM32MP1的SYSRAM共有256KB，地址从==0x2000 0000 ~ 0x2FFF FFFF==
5. ROM代码会把FSBL镜像拷贝到==0x2FFC 2400==地址，但是由于镜像前面还有一个长度为256字节（0x100）的STM32 header，所以FSBL镜像的真是其实地址为0x2FFC 2400 + 0x100 = ==0x2FFC 2500==
6. 从而可以计算出，从0x2FFC 2500 到 0x2FFF FFFF 一共有246.75KB，故==FSBL镜像大小不可以超过246.75KB==
7. FSBL authentication 成功后，ROM代码会把boot上下文的起始地址存储到R0寄存器中，然后跳转到FSBL镜像的入口地址
8. boot上下文包含了boot信息，例如选定的boot设备，还有一些与authentication 相关的信息。结构体boot_api_context_t定义了boot上下文结构

------

## STM32 头部文件 (Header File)

```
/*
 * Copyright (c) 2017-2020, STMicroelectronics - All Rights Reserved
 *
 * SPDX-License-Identifier: BSD-3-Clause
 */

#include <asm/byteorder.h>
#include <errno.h>
#include <fcntl.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>

/* Magic = 'S' 'T' 'M' 0x32 */
#define HEADER_MAGIC		__be32_to_cpu(0x53544D32)
#define VER_MAJOR		2
#define VER_MINOR		1
#define VER_VARIANT		0
#define HEADER_VERSION_V1	0x1
#define TF_BINARY_TYPE		0x10
#define UBOOT_BINARY_TYPE		0x00

/* Default option : bit0 => no signature */
#define HEADER_DEFAULT_OPTION	(__cpu_to_le32(0x00000001))

struct stm32_header {
	uint32_t magic_number;
	uint8_t image_signature[64];
	uint32_t image_checksum;
	uint8_t  header_version[4];
	uint32_t image_length;
	uint32_t image_entry_point;
	uint32_t reserved1;
	uint32_t load_address;
	uint32_t reserved2;
	uint32_t version_number;
	uint32_t option_flags;
	uint32_t ecdsa_algorithm;
	uint8_t ecdsa_public_key[64];
	uint8_t padding[83];
	uint8_t binary_type;
};

static void stm32image_default_header(struct stm32_header *ptr)
{
	if (!ptr) {
		return;
	}

	ptr->magic_number = HEADER_MAGIC;
	ptr->option_flags = HEADER_DEFAULT_OPTION;
	ptr->ecdsa_algorithm = __cpu_to_le32(1);
	ptr->version_number = __cpu_to_le32(0);
	ptr->binary_type = UBOOT_BINARY_TYPE;
}

static uint32_t stm32image_checksum(void *start, uint32_t len)
{
	uint32_t csum = 0;
	uint32_t hdr_len = sizeof(struct stm32_header);
	uint8_t *p;

	if (len < hdr_len) {
		return 0;
	}

	p = (unsigned char *)start + hdr_len;
	len -= hdr_len;

	while (len > 0) {
		csum += *p;
		p++;
		len--;
	}

	return csum;
}

static void stm32image_print_header(const void *ptr)
{
	struct stm32_header *stm32hdr = (struct stm32_header *)ptr;

	printf("Image Type   : ST Microelectronics STM32 V%d.%d\n",
	       stm32hdr->header_version[VER_MAJOR],
	       stm32hdr->header_version[VER_MINOR]);
	printf("Image Size   : %lu bytes\n",
	       (unsigned long)__le32_to_cpu(stm32hdr->image_length));
	printf("Image Load   : 0x%08x\n",
	       __le32_to_cpu(stm32hdr->load_address));
	printf("Entry Point  : 0x%08x\n",
	       __le32_to_cpu(stm32hdr->image_entry_point));
	printf("Checksum     : 0x%08x\n",
	       __le32_to_cpu(stm32hdr->image_checksum));
	printf("Option     : 0x%08x\n",
	       __le32_to_cpu(stm32hdr->option_flags));
	printf("Version	   : 0x%08x\n",
	       __le32_to_cpu(stm32hdr->version_number));
}

static void stm32image_set_header(void *ptr, struct stat *sbuf, int ifd,
				  uint32_t loadaddr, uint32_t ep, uint32_t ver,
				  uint32_t major, uint32_t minor)
{
	struct stm32_header *stm32hdr = (struct stm32_header *)ptr;

	stm32image_default_header(stm32hdr);

	stm32hdr->header_version[VER_MAJOR] = major;
	stm32hdr->header_version[VER_MINOR] = minor;
	stm32hdr->load_address = __cpu_to_le32(loadaddr);
	stm32hdr->image_entry_point = __cpu_to_le32(ep);
	stm32hdr->image_length = __cpu_to_le32((uint32_t)sbuf->st_size -
					     sizeof(struct stm32_header));
	stm32hdr->image_checksum =
		__cpu_to_le32(stm32image_checksum(ptr, sbuf->st_size));
	stm32hdr->version_number = __cpu_to_le32(ver);
}

static int stm32image_create_header_file(char *srcname, char *destname,
					 uint32_t loadaddr, uint32_t entry,
					 uint32_t version, uint32_t major,
					 uint32_t minor)
{
	int src_fd, dest_fd;
	struct stat sbuf;
	unsigned char *ptr;
	struct stm32_header stm32image_header;

	dest_fd = open(destname, O_RDWR | O_CREAT | O_TRUNC | O_APPEND, 0666);
	if (dest_fd == -1) {
		fprintf(stderr, "Can't open %s: %s\n", destname,
			strerror(errno));
		return -1;
	}

	src_fd = open(srcname, O_RDONLY);
	if (src_fd == -1) {
		fprintf(stderr, "Can't open %s: %s\n", srcname,
			strerror(errno));
		return -1;
	}

	if (fstat(src_fd, &sbuf) < 0) {
		return -1;
	}

	ptr = mmap(NULL, sbuf.st_size, PROT_READ, MAP_SHARED, src_fd, 0);
	if (ptr == MAP_FAILED) {
		fprintf(stderr, "Can't read %s\n", srcname);
		return -1;
	}

	memset(&stm32image_header, 0, sizeof(struct stm32_header));

	if (write(dest_fd, &stm32image_header, sizeof(struct stm32_header)) !=
	    sizeof(struct stm32_header)) {
		fprintf(stderr, "Write error %s: %s\n", destname,
			strerror(errno));
		return -1;
	}

	if (write(dest_fd, ptr, sbuf.st_size) != sbuf.st_size) {
		fprintf(stderr, "Write error on %s: %s\n", destname,
			strerror(errno));
		return -1;
	}

	munmap((void *)ptr, sbuf.st_size);
	close(src_fd);

	if (fstat(dest_fd, &sbuf) < 0) {
		return -1;
	}

	ptr = mmap(0, sbuf.st_size, PROT_READ | PROT_WRITE, MAP_SHARED,
		   dest_fd, 0);

	if (ptr == MAP_FAILED) {
		fprintf(stderr, "Can't write %s\n", destname);
		return -1;
	}

	stm32image_set_header(ptr, &sbuf, dest_fd, loadaddr, entry, version,
			      major, minor);

	stm32image_print_header(ptr);

	munmap((void *)ptr, sbuf.st_size);
	close(dest_fd);
	return 0;
}

int main(int argc, char *argv[])
{
	int opt, loadaddr = -1, entry = -1, err = 0, version = 0;
	int major = HEADER_VERSION_V1;
	int minor = 0;
	char *dest = NULL, *src = NULL;

	while ((opt = getopt(argc, argv, ":s:d:l:e:v:m:n:")) != -1) {
		switch (opt) {
		case 's':
			src = optarg;
			break;
		case 'd':
			dest = optarg;
			break;
		case 'l':
			loadaddr = strtol(optarg, NULL, 0);
			break;
		case 'e':
			entry = strtol(optarg, NULL, 0);
			break;
		case 'v':
			version = strtol(optarg, NULL, 0);
			break;
		case 'm':
			major = strtol(optarg, NULL, 0);
			break;
		case 'n':
			minor = strtol(optarg, NULL, 0);
			break;
		default:
			fprintf(stderr,
				"Usage : %s [-s srcfile] [-d destfile] [-l loadaddr] [-e entry_point] [-m major] [-n minor]\n",
					argv[0]);
			return -1;
		}
	}

	if (!src) {
		fprintf(stderr, "Missing -s option\n");
		return -1;
	}

	if (!dest) {
		fprintf(stderr, "Missing -d option\n");
		return -1;
	}

	if (loadaddr == -1) {
		fprintf(stderr, "Missing -l option\n");
		return -1;
	}

	if (entry == -1) {
		fprintf(stderr, "Missing -e option\n");
		return -1;
	}

	err = stm32image_create_header_file(src, dest, loadaddr,
					    entry, version, major, minor);

	return err;
}

```
