---
title: Tinyalsa PCM API 实现深度剖析
date: 2023-10-07 20:23:29
categories: Linux 内核
tags:
- Linux 内核
---

高级 Linux 音频架构 (ALSA) 用于为 Linux 操作系统提供音频和 MIDI 功能。它可以高效地支持所有类型的音频接口，从消费者声卡到专业的多通道音频接口。它支持全模块化的音频驱动。它是 SMP 和线程安全的。它提供了用户空间库 (alsa-lib) 来简化应用程序编程并提供了更高级的功能。它支持老式的 OSS API。

## ALSA 内核接口

ALSA 定义了用户空间程序和内核音频子系统之间交互的接口。这些接口主要由几个部分组成：

1. 导出的音频设备文件。Linux 内核 ALSA 子系统以固定的规则在 `devtmpfs` 文件系统中特定的位置导出音频设备文件，这些音频设备文件在文件系统中的位置，具体来说为 `/dev/snd/`。如某 Linux 系统中的音频设备文件：
```
$ ls -sl /dev/snd/
总用量 0
0 crw-rw----+ 1 root audio 116,  5 10月 10 09:39 controlC0
0 crw-rw----+ 1 root audio 116,  3 10月 11 09:32 pcmC0D0c
0 crw-rw----+ 1 root audio 116,  2 10月 11 17:21 pcmC0D0p
0 crw-rw----+ 1 root audio 116,  4 10月 10 09:39 pcmC0D1c
0 crw-rw----+ 1 root audio 116,  1 10月 11 09:29 seq
0 crw-rw----+ 1 root audio 116, 33 10月 11 09:29 timer
```

所有音频设备文件其文件类型都是字符设备，其主设备号都是 116，次设备号因各设备文件创建的先后而异，但 `timer` 设备文件的从设备号为 33。ALSA 为特定声卡导出多个设备文件，`controlCX` 设备文件用于控制声卡，特别是对于 audio codec 等设备的内部小部件进行控制；`pcmCXDYp`/`pcmCXDYc` 设备文件用于数据交换，用于播放的设备文件以 `p` 结尾，`p` 表示 **playout**，用于录制的设备文件以 `c` 结尾，`c` 表示 **capture** (音频设备文件名中的 `X` 表示声卡编号，`Y` 表示音频设备编号，如上面看到的 0、1 等)。`controlCX` 和 `pcmCXDYp`/`pcmCXDYc` 是 ALSA 导出的最为重要的设备文件。

2. 各个音频设备文件支持的文件操作。不同的音频设备文件支持的文件操作集合不同，具体如下：
 - `controlCX` 设备文件：`open()`、`close()`、`ioctl()`、`read()`、`lseek()`/`llseek()`、`poll()` 和 `fcntl()`；
 - `pcmCXDYp` 设备文件：`open()`、`close()`、`write()`、`ioctl()`、`lseek()`/`llseek()`，`poll()`，`mmap()` 和 `fcntl()`；
 - `pcmCXDYc` 设备文件：`open()`、`close()`、`read()`、`ioctl()`、`lseek()`/`llseek()`，`poll()`，`mmap()` 和 `fcntl()`。

内核中，`controlCX` 设备文件的文件操作定义(位于`sound/core/control.c`)如下：
```
static const struct file_operations snd_ctl_f_ops =
{
	.owner =	THIS_MODULE,
	.read =		snd_ctl_read,
	.open =		snd_ctl_open,
	.release =	snd_ctl_release,
	.llseek =	no_llseek,
	.poll =		snd_ctl_poll,
	.unlocked_ioctl =	snd_ctl_ioctl,
	.compat_ioctl =	snd_ctl_ioctl_compat,
	.fasync =	snd_ctl_fasync,
};
```

内核中，`pcmCXDYp`/`pcmCXDYc` 音频设备文件的文件操作定义 (位于`sound/core/pcm_native.c`) 如下：
```
const struct file_operations snd_pcm_f_ops[2] = {
	{
		.owner =		THIS_MODULE,
		.write =		snd_pcm_write,
		.write_iter =		snd_pcm_writev,
		.open =			snd_pcm_playback_open,
		.release =		snd_pcm_release,
		.llseek =		no_llseek,
		.poll =			snd_pcm_poll,
		.unlocked_ioctl =	snd_pcm_ioctl,
		.compat_ioctl = 	snd_pcm_ioctl_compat,
		.mmap =			snd_pcm_mmap,
		.fasync =		snd_pcm_fasync,
		.get_unmapped_area =	snd_pcm_get_unmapped_area,
	},
	{
		.owner =		THIS_MODULE,
		.read =			snd_pcm_read,
		.read_iter =		snd_pcm_readv,
		.open =			snd_pcm_capture_open,
		.release =		snd_pcm_release,
		.llseek =		no_llseek,
		.poll =			snd_pcm_poll,
		.unlocked_ioctl =	snd_pcm_ioctl,
		.compat_ioctl = 	snd_pcm_ioctl_compat,
		.mmap =			snd_pcm_mmap,
		.fasync =		snd_pcm_fasync,
		.get_unmapped_area =	snd_pcm_get_unmapped_area,
	}
};
```

Linux 内核 ALSA 框架为音频设备文件提供的这些文件操作不是完全正交的，不同文件操作的功能有一定的重合，如 `pcmCXDYp`/`pcmCXDYc` 音频设备文件，它们的 `write()`/`read()` 操作，和 `ioctl()` 操作的一些命令在功能上是重合的。这些音频文件操作不是每个都有意义，如 `lseek()`/`llseek()` 操作被实现为空操作，`fcntl()` 对于音频设备文件没有意义。

Linux 内核 ALSA 框架的用于方便应用程序开发的用户空间封装库，如 alsa-lib 和 tinyalsa 等，可以再次定义各设备文件支持的操作集合。如 tinyalsa 在 `tinyalsa/src/pcm_io.h` 文件中定义了 pcm 文件操作集合：
```
struct pcm_ops {
    int (*open) (unsigned int card, unsigned int device,
                 unsigned int flags, void **data, struct snd_node *node);
    void (*close) (void *data);
    int (*ioctl) (void *data, unsigned int cmd, ...);
    void *(*mmap) (void *data, void *addr, size_t length, int prot, int flags,
                   off_t offset);
    int (*munmap) (void *data, void *addr, size_t length);
    int (*poll) (void *data, struct pollfd *pfd, nfds_t nfds, int timeout);
};
```

tinyalsa 在 `tinyalsa/src/mixer_io.h` 文件中定义了 control 文件操作集合：
```
struct mixer_ops {
    void (*close) (void *data);
    int (*get_poll_fd) (void *data, struct pollfd *pfd, int count);
    ssize_t (*read_event) (void *data, struct snd_ctl_event *ev, size_t size);
    int (*ioctl) (void *data, unsigned int cmd, ...);
};
```

用户空间的内核 ALSA 框架封装库，使用了 ALSA 框架导出的文件操作集合的子集。

3. 各个音频设备文件的文件操作支持的命令及参数和返回值中用到的枚举和各种数据结构等。pcm 音频设备文件的 `ioctl()` 操作支持众多命令，这些命令如下：

 * **SNDRV_PCM_IOCTL_PVERSION**
 * **SNDRV_PCM_IOCTL_INFO**
 * **SNDRV_PCM_IOCTL_TSTAMP**
 * **SNDRV_PCM_IOCTL_TTSTAMP**
 * **SNDRV_PCM_IOCTL_USER_PVERSION**
 * **SNDRV_PCM_IOCTL_HW_REFINE**
 * **SNDRV_PCM_IOCTL_HW_PARAMS**
 * **SNDRV_PCM_IOCTL_HW_FREE**
 * **SNDRV_PCM_IOCTL_SW_PARAMS**
 * **SNDRV_PCM_IOCTL_STATUS**
 * **SNDRV_PCM_IOCTL_DELAY**
 * **SNDRV_PCM_IOCTL_HWSYNC**
 * **SNDRV_PCM_IOCTL_SYNC_PTR**
 * **SNDRV_PCM_IOCTL_STATUS_EXT**
 * **SNDRV_PCM_IOCTL_CHANNEL_INFO**
 * **SNDRV_PCM_IOCTL_PREPARE**
 * **SNDRV_PCM_IOCTL_RESET**
 * **SNDRV_PCM_IOCTL_START**
 * **SNDRV_PCM_IOCTL_DROP**
 * **SNDRV_PCM_IOCTL_DRAIN**
 * **SNDRV_PCM_IOCTL_PAUSE**
 * **SNDRV_PCM_IOCTL_REWIND**
 * **SNDRV_PCM_IOCTL_RESUME**
 * **SNDRV_PCM_IOCTL_XRUN**
 * **SNDRV_PCM_IOCTL_FORWARD**
 * **SNDRV_PCM_IOCTL_WRITEI_FRAMES**
 * **SNDRV_PCM_IOCTL_READI_FRAMES**
 * **SNDRV_PCM_IOCTL_WRITEN_FRAMES**
 * **SNDRV_PCM_IOCTL_READN_FRAMES**
 * **SNDRV_PCM_IOCTL_LINK**
 * **SNDRV_PCM_IOCTL_UNLINK**

`pcmCXDYp` 音频设备文件的 `write()` 操作与它的 `ioctl()` 操作的 **SNDRV_PCM_IOCTL_WRITEI_FRAMES** 和 **SNDRV_PCM_IOCTL_WRITEN_FRAMES** 命令实现相同的功能。`pcmCXDYc` 音频设备文件的 `read()` 操作与它的 `ioctl()` 操作的 **SNDRV_PCM_IOCTL_READI_FRAMES** 和 **SNDRV_PCM_IOCTL_READN_FRAMES** 命令实现相同的功能。

pcm 音频设备文件的 `mmap()` 操作支持一些特殊的 `offset`，以获取一些特别的用于和内核通信的数据结构，这些特殊的 `offset` 值如下：
 * **SNDRV_PCM_MMAP_OFFSET_DATA** - 0x00000000
 * **SNDRV_PCM_MMAP_OFFSET_STATUS** - 0x80000000
 * **SNDRV_PCM_MMAP_OFFSET_CONTROL** - 0x81000000

control 音频设备文件的 `ioctl()` 操作也支持众多命令，这些命令如下：

 * **SNDRV_CTL_IOCTL_PVERSION**
 * **SNDRV_CTL_IOCTL_CARD_INFO**
 * **SNDRV_CTL_IOCTL_ELEM_LIST**
 * **SNDRV_CTL_IOCTL_ELEM_INFO**
 * **SNDRV_CTL_IOCTL_ELEM_READ**
 * **SNDRV_CTL_IOCTL_ELEM_WRITE**
 * **SNDRV_CTL_IOCTL_ELEM_LOCK**
 * **SNDRV_CTL_IOCTL_ELEM_UNLOCK**
 * **SNDRV_CTL_IOCTL_SUBSCRIBE_EVENTS**
 * **SNDRV_CTL_IOCTL_ELEM_ADD**
 * **SNDRV_CTL_IOCTL_ELEM_REPLACE**
 * **SNDRV_CTL_IOCTL_ELEM_REMOVE**
 * **SNDRV_CTL_IOCTL_TLV_READ**
 * **SNDRV_CTL_IOCTL_TLV_WRITE**
 * **SNDRV_CTL_IOCTL_TLV_COMMAND**
 * **SNDRV_CTL_IOCTL_HWDEP_NEXT_DEVICE**
 * **SNDRV_CTL_IOCTL_HWDEP_INFO**
 * **SNDRV_CTL_IOCTL_PCM_NEXT_DEVICE**
 * **SNDRV_CTL_IOCTL_PCM_INFO**
 * **SNDRV_CTL_IOCTL_PCM_PREFER_SUBDEVICE**
 * **SNDRV_CTL_IOCTL_RAWMIDI_NEXT_DEVICE**
 * **SNDRV_CTL_IOCTL_RAWMIDI_INFO**
 * **SNDRV_CTL_IOCTL_RAWMIDI_PREFER_SUBDEVICE**
 * **SNDRV_CTL_IOCTL_POWER**
 * **SNDRV_CTL_IOCTL_POWER_STATE**

Linux 内核 ALSA 框架的用户空间封装库，如 alsa-lib 和 tinyalsa 等，封装 Linux 内核音频设备文件的这些文件操作，实现内部建立的文件操作集合。这里不关心 control 设备文件操作的封装。tinyalsa 在 `tinyalsa/src/pcm_hw.c` 文件中，基于系统调用封装器实现各个 pcm 操作，具体如下：
```
struct pcm_hw_data {
    /** Card number of the pcm device */
    unsigned int card;
    /** Device number for the pcm device */
    unsigned int device;
    /** File descriptor to the pcm device file node */
    int fd;
    /** Pointer to the pcm node from snd card definiton */
    struct snd_node *node;
};

static void pcm_hw_close(void *data)
{
    struct pcm_hw_data *hw_data = data;

    if (hw_data->fd >= 0)
        close(hw_data->fd);

    free(hw_data);
}

static int pcm_hw_ioctl(void *data, unsigned int cmd, ...)
{
    struct pcm_hw_data *hw_data = data;
    va_list ap;
    void *arg;

    va_start(ap, cmd);
    arg = va_arg(ap, void *);
    va_end(ap);

    return ioctl(hw_data->fd, cmd, arg);
}

static int pcm_hw_poll(void *data __attribute__((unused)),
                        struct pollfd *pfd, nfds_t nfds, int timeout)
{
    return poll(pfd, nfds, timeout);
}

static void *pcm_hw_mmap(void *data, void *addr, size_t length, int prot,
                       int flags, off_t offset)
{
    struct pcm_hw_data *hw_data = data;

    return mmap(addr, length, prot, flags, hw_data->fd, offset);
}

static int pcm_hw_munmap(void *data __attribute__((unused)), void *addr, size_t length)
{
    return munmap(addr, length);
}

static int pcm_hw_open(unsigned int card, unsigned int device,
                unsigned int flags, void **data, struct snd_node *node)
{
    struct pcm_hw_data *hw_data;
    char fn[256];
    int fd;

    hw_data = calloc(1, sizeof(*hw_data));
    if (!hw_data) {
        return -ENOMEM;
    }

    snprintf(fn, sizeof(fn), "/dev/snd/pcmC%uD%u%c", card, device,
             flags & PCM_IN ? 'c' : 'p');
    // Open the device with non-blocking flag to avoid to be blocked in kernel when all of the
    //   substreams of this PCM device are opened by others.
    fd = open(fn, O_RDWR | O_NONBLOCK);

    if (fd < 0) {
        free(hw_data);
        return fd;
    }

    if ((flags & PCM_NONBLOCK) == 0) {
        // Set the file descriptor to blocking mode.
        if (fcntl(fd, F_SETFL, fcntl(fd, F_GETFL) & ~O_NONBLOCK) < 0) {
            fprintf(stderr, "failed to set to blocking mode on %s", fn);
            close(fd);
            free(hw_data);
            return -ENODEV;
        }
    }

    hw_data->card = card;
    hw_data->device = device;
    hw_data->fd = fd;
    hw_data->node = node;

    *data = hw_data;

    return fd;
}

const struct pcm_ops hw_ops = {
    .open = pcm_hw_open,
    .close = pcm_hw_close,
    .ioctl = pcm_hw_ioctl,
    .mmap = pcm_hw_mmap,
    .munmap = pcm_hw_munmap,
    .poll = pcm_hw_poll,
};
```

tinyalsa 创建了一个结构体 `struct pcm_hw_data` 来保存具体的 pcm 设备文件相关的信息，包括打开的音频设备文件的文件描述符，音频设备文件所属的声卡编号及设备编号等。这里各个操作的具体实现没有什么特别值得关注的地方。

## Tinyalsa 的 API

ALSA 的用户空间封装库，基于 Linux 内核提供的接口，为应用程序开发提供更高级的接口。ALSA 官方的 alsa-lib 库提供了功能完备且强大的接口，但它的接口显得有点易用性不足。tinyalsa 提供了一个 alsa-lib 库的简化版，其功能接口不如 alsa-lib 完备，但好在方便易用。tinyalsa 已经用于 Android 系统多年，众多 audio HAL 服务基于这个库实现。这里关注 PCM 相关的 API。alsa-lib 的 API 多以 `snd_pcm_` 开头，如 `snd_pcm_open()`、`snd_pcm_close()`、`snd_pcm_info()`、`snd_pcm_start()`、`snd_pcm_writei()`、`snd_pcm_readi()`、`snd_pcm_hw_params_set_channels()` 和 `snd_pcm_wait()` 等。tinyalsa 的 API 则多以 `pcm_` 开头，如 `pcm_open()`、`pcm_close()`、`pcm_set_config()`、`pcm_writei()`、`pcm_readi()`、`pcm_mmap_write()`、`pcm_mmap_read()`、`pcm_mmap_begin()`、`pcm_mmap_commit()`、`pcm_start()`、`pcm_stop()` 和 `pcm_wait()` 等。alsa-lib 的 API 和 tinyalsa 的比较大的区别在与，alsa-lib 通过 `snd_pcm_hw_params_set_format()` 和 `snd_pcm_hw_params_set_channels()` 等众多接口为打开的音频流设置参数，而 tinyalsa 通过 `pcm_set_config()` 单个接口设置多个音频流参数。

tinyalsa 的 PCM 相关 API 有如下 (位于 `tinyalsa/include/tinyalsa/pcm.h`) 这些：
```
#if defined(__cplusplus)
extern "C" {
#endif

/** Audio sample format of a PCM.
 * The first letter specifiers whether the sample is signed or unsigned.
 * The letter 'S' means signed. The letter 'U' means unsigned.
 * The following number is the amount of bits that the sample occupies in memory.
 * Following the underscore, specifiers whether the sample is big endian or little endian.
 * The letters 'LE' mean little endian.
 * The letters 'BE' mean big endian.
 * This enumeration is used in the @ref pcm_config structure.
 * @ingroup libtinyalsa-pcm
 */
enum pcm_format {

/* Note: This section must stay in the same
 * order for binary compatibility with older
 * versions of TinyALSA. */

    PCM_FORMAT_INVALID = -1,
    /** Signed 16-bit, little endian */
    PCM_FORMAT_S16_LE = 0,
    /** Signed, 32-bit, little endian */
    PCM_FORMAT_S32_LE,
    /** Signed, 8-bit */
    PCM_FORMAT_S8,
    /** Signed, 24-bit (32-bit in memory), little endian */
    PCM_FORMAT_S24_LE,
    /** Signed, 24-bit, little endian */
    PCM_FORMAT_S24_3LE,

/* End of compatibility section. */

    /** Signed, 16-bit, big endian */
    PCM_FORMAT_S16_BE,
    /** Signed, 24-bit (32-bit in memory), big endian */
    PCM_FORMAT_S24_BE,
    /** Signed, 24-bit, big endian */
    PCM_FORMAT_S24_3BE,
    /** Signed, 32-bit, big endian */
    PCM_FORMAT_S32_BE,
    /** 32-bit float, little endian */
    PCM_FORMAT_FLOAT_LE,
    /** 32-bit float, big endian */
    PCM_FORMAT_FLOAT_BE,
    /** Max of the enumeration list, not an actual format. */
    PCM_FORMAT_MAX
};

/** A bit mask of 256 bits (32 bytes) that describes some hardware parameters of a PCM */
struct pcm_mask {
    /** bits of the bit mask */
    unsigned int bits[32 / sizeof(unsigned int)];
};

/** Encapsulates the hardware and software parameters of a PCM.
 * @ingroup libtinyalsa-pcm
 */
struct pcm_config {
    /** The number of channels in a frame */
    unsigned int channels;
    /** The number of frames per second */
    unsigned int rate;
    /** The number of frames in a period */
    unsigned int period_size;
    /** The number of periods in a PCM */
    unsigned int period_count;
    /** The sample format of a PCM */
    enum pcm_format format;
    /* Values to use for the ALSA start, stop and silence thresholds, and
     * silence size.  Setting any one of these values to 0 will cause the
     * default tinyalsa values to be used instead.
     * Tinyalsa defaults are as follows.
     *
     * start_threshold   : period_count * period_size
     * stop_threshold    : period_count * period_size
     * silence_threshold : 0
     * silence_size      : 0
     */
    /** The minimum number of frames required to start the PCM */
    unsigned long start_threshold;
    /** The minimum number of frames required to stop the PCM */
    unsigned long stop_threshold;
    /** The minimum number of frames to silence the PCM */
    unsigned long silence_threshold;
    /** The number of frames to overwrite the playback buffer when the playback underrun is greater
     * than the silence threshold */
    unsigned long silence_size;

    unsigned long avail_min;
};

/** Enumeration of a PCM's hardware parameters.
 * Each of these parameters is either a mask or an interval.
 * @ingroup libtinyalsa-pcm
 */
enum pcm_param
{
    /** A mask that represents the type of read or write method available (e.g. interleaved, mmap). */
    PCM_PARAM_ACCESS,
    /** A mask that represents the @ref pcm_format available (e.g. @ref PCM_FORMAT_S32_LE) */
    PCM_PARAM_FORMAT,
    /** A mask that represents the subformat available */
    PCM_PARAM_SUBFORMAT,
    /** An interval representing the range of sample bits available (e.g. 8 to 32) */
    PCM_PARAM_SAMPLE_BITS,
    /** An interval representing the range of frame bits available (e.g. 8 to 64) */
    PCM_PARAM_FRAME_BITS,
    /** An interval representing the range of channels available (e.g. 1 to 2) */
    PCM_PARAM_CHANNELS,
    /** An interval representing the range of rates available (e.g. 44100 to 192000) */
    PCM_PARAM_RATE,
    PCM_PARAM_PERIOD_TIME,
    /** The number of frames in a period */
    PCM_PARAM_PERIOD_SIZE,
    /** The number of bytes in a period */
    PCM_PARAM_PERIOD_BYTES,
    /** The number of periods for a PCM */
    PCM_PARAM_PERIODS,
    PCM_PARAM_BUFFER_TIME,
    PCM_PARAM_BUFFER_SIZE,
    PCM_PARAM_BUFFER_BYTES,
    PCM_PARAM_TICK_TIME,
}; /* enum pcm_param */

struct pcm_params;

struct pcm_params *pcm_params_get(unsigned int card, unsigned int device,
                                  unsigned int flags);

void pcm_params_free(struct pcm_params *pcm_params);

const struct pcm_mask *pcm_params_get_mask(const struct pcm_params *pcm_params, enum pcm_param param);

unsigned int pcm_params_get_min(const struct pcm_params *pcm_params, enum pcm_param param);

unsigned int pcm_params_get_max(const struct pcm_params *pcm_params, enum pcm_param param);

/* Converts the pcm parameters to a human readable string.
 * The string parameter is a caller allocated buffer of size bytes,
 * which is then filled up to size - 1 and null terminated,
 * if size is greater than zero.
 * The return value is the number of bytes copied to string
 * (not including null termination) if less than size; otherwise,
 * the number of bytes required for the buffer.
 */
int pcm_params_to_string(struct pcm_params *params, char *string, unsigned int size);

/* Returns 1 if the pcm_format is present (format bit set) in
 * the pcm_params structure; 0 otherwise, or upon unrecognized format.
 */
int pcm_params_format_test(struct pcm_params *params, enum pcm_format format);

struct pcm;

struct pcm *pcm_open(unsigned int card,
                     unsigned int device,
                     unsigned int flags,
                     const struct pcm_config *config);

struct pcm *pcm_open_by_name(const char *name,
                             unsigned int flags,
                             const struct pcm_config *config);

int pcm_close(struct pcm *pcm);

int pcm_is_ready(const struct pcm *pcm);

unsigned int pcm_get_channels(const struct pcm *pcm);

const struct pcm_config * pcm_get_config(const struct pcm *pcm);

unsigned int pcm_get_rate(const struct pcm *pcm);

enum pcm_format pcm_get_format(const struct pcm *pcm);

int pcm_get_file_descriptor(const struct pcm *pcm);

const char *pcm_get_error(const struct pcm *pcm);

int pcm_set_config(struct pcm *pcm, const struct pcm_config *config);

unsigned int pcm_format_to_bits(enum pcm_format format);

unsigned int pcm_get_buffer_size(const struct pcm *pcm);

unsigned int pcm_frames_to_bytes(const struct pcm *pcm, unsigned int frames);

unsigned int pcm_bytes_to_frames(const struct pcm *pcm, unsigned int bytes);

int pcm_get_htimestamp(struct pcm *pcm, unsigned int *avail, struct timespec *tstamp);

unsigned int pcm_get_subdevice(const struct pcm *pcm);

int pcm_writei(struct pcm *pcm, const void *data, unsigned int frame_count) TINYALSA_WARN_UNUSED_RESULT;

int pcm_readi(struct pcm *pcm, void *data, unsigned int frame_count) TINYALSA_WARN_UNUSED_RESULT;

int pcm_write(struct pcm *pcm, const void *data, unsigned int count) TINYALSA_DEPRECATED;

int pcm_read(struct pcm *pcm, void *data, unsigned int count) TINYALSA_DEPRECATED;

int pcm_mmap_write(struct pcm *pcm, const void *data, unsigned int count) TINYALSA_DEPRECATED;

int pcm_mmap_read(struct pcm *pcm, void *data, unsigned int count) TINYALSA_DEPRECATED;

int pcm_mmap_begin(struct pcm *pcm, void **areas, unsigned int *offset, unsigned int *frames);

int pcm_mmap_commit(struct pcm *pcm, unsigned int offset, unsigned int frames);

int pcm_mmap_avail(struct pcm *pcm);

int pcm_mmap_get_hw_ptr(struct pcm* pcm, unsigned int *hw_ptr, struct timespec *tstamp);

int pcm_get_poll_fd(struct pcm *pcm);

int pcm_link(struct pcm *pcm1, struct pcm *pcm2);

int pcm_unlink(struct pcm *pcm);

int pcm_prepare(struct pcm *pcm);

int pcm_start(struct pcm *pcm);

int pcm_stop(struct pcm *pcm);

int pcm_wait(struct pcm *pcm, int timeout);

long pcm_get_delay(struct pcm *pcm);

int pcm_ioctl(struct pcm *pcm, int code, ...) TINYALSA_DEPRECATED;

#if defined(__cplusplus)
}  /* extern "C" */
#endif
```

tinyalsa 的 PCM 相关 API 主要分为两部分。一是硬件设备参数查询及测试，这主要包括如下这些：
```
struct pcm_params;

struct pcm_params *pcm_params_get(unsigned int card, unsigned int device,
                                  unsigned int flags);

void pcm_params_free(struct pcm_params *pcm_params);

const struct pcm_mask *pcm_params_get_mask(const struct pcm_params *pcm_params, enum pcm_param param);

unsigned int pcm_params_get_min(const struct pcm_params *pcm_params, enum pcm_param param);

unsigned int pcm_params_get_max(const struct pcm_params *pcm_params, enum pcm_param param);

/* Converts the pcm parameters to a human readable string.
 * The string parameter is a caller allocated buffer of size bytes,
 * which is then filled up to size - 1 and null terminated,
 * if size is greater than zero.
 * The return value is the number of bytes copied to string
 * (not including null termination) if less than size; otherwise,
 * the number of bytes required for the buffer.
 */
int pcm_params_to_string(struct pcm_params *params, char *string, unsigned int size);

/* Returns 1 if the pcm_format is present (format bit set) in
 * the pcm_params structure; 0 otherwise, or upon unrecognized format.
 */
int pcm_params_format_test(struct pcm_params *params, enum pcm_format format);
```

tinyalsa 用一个没有实际定义的 `struct pcm_params` 向应用程序表示音频流的硬件设备参数。在 tinyalsa 的实现内部，用 `struct snd_pcm_hw_params` 表示音频流的硬件设备参数。`struct snd_pcm_hw_params` 是 Linux 内核定义的，用户空间程序和 Linux 内核通过这个结构交换音频流的硬件设备参数，这个结构体定义 (位于 `include/uapi/sound/asound.h`) 如下：
```
struct snd_interval {
	unsigned int min, max;
	unsigned int openmin:1,
		     openmax:1,
		     integer:1,
		     empty:1;
};

#define SNDRV_MASK_MAX	256

struct snd_mask {
	__u32 bits[(SNDRV_MASK_MAX+31)/32];
};

struct snd_pcm_hw_params {
	unsigned int flags;
	struct snd_mask masks[SNDRV_PCM_HW_PARAM_LAST_MASK -
			       SNDRV_PCM_HW_PARAM_FIRST_MASK + 1];
	struct snd_mask mres[5];	/* reserved masks */
	struct snd_interval intervals[SNDRV_PCM_HW_PARAM_LAST_INTERVAL -
				        SNDRV_PCM_HW_PARAM_FIRST_INTERVAL + 1];
	struct snd_interval ires[9];	/* reserved intervals */
	unsigned int rmask;		/* W: requested masks */
	unsigned int cmask;		/* R: changed masks */
	unsigned int info;		/* R: Info flags for returned setup */
	unsigned int msbits;		/* R: used most significant bits */
	unsigned int rate_num;		/* R: rate numerator */
	unsigned int rate_den;		/* R: rate denominator */
	snd_pcm_uframes_t fifo_size;	/* R: chip FIFO size in frames */
	unsigned char reserved[64];	/* reserved for future */
};
```

`struct snd_pcm_hw_params` 主要用 `masks` 和 `intervals` 两个数组保存不同类型的音频参数，`masks` 用于保存 mask 型参数，`intervals` 用于保存 interval 型参数。对于每一个参数，Linux 内核都会给它分配一个参数编号，且在这两个数组中的一个里，有一个对应的位置保存其相关参数值。Linux 内核支持的音频参数 (位于 `include/uapi/sound/asound.h`) 有如下这些：
```
typedef int snd_pcm_hw_param_t;
#define	SNDRV_PCM_HW_PARAM_ACCESS	0	/* Access type */
#define	SNDRV_PCM_HW_PARAM_FORMAT	1	/* Format */
#define	SNDRV_PCM_HW_PARAM_SUBFORMAT	2	/* Subformat */
#define	SNDRV_PCM_HW_PARAM_FIRST_MASK	SNDRV_PCM_HW_PARAM_ACCESS
#define	SNDRV_PCM_HW_PARAM_LAST_MASK	SNDRV_PCM_HW_PARAM_SUBFORMAT

#define	SNDRV_PCM_HW_PARAM_SAMPLE_BITS	8	/* Bits per sample */
#define	SNDRV_PCM_HW_PARAM_FRAME_BITS	9	/* Bits per frame */
#define	SNDRV_PCM_HW_PARAM_CHANNELS	10	/* Channels */
#define	SNDRV_PCM_HW_PARAM_RATE		11	/* Approx rate */
#define	SNDRV_PCM_HW_PARAM_PERIOD_TIME	12	/* Approx distance between
						 * interrupts in us
						 */
#define	SNDRV_PCM_HW_PARAM_PERIOD_SIZE	13	/* Approx frames between
						 * interrupts
						 */
#define	SNDRV_PCM_HW_PARAM_PERIOD_BYTES	14	/* Approx bytes between
						 * interrupts
						 */
#define	SNDRV_PCM_HW_PARAM_PERIODS	15	/* Approx interrupts per
						 * buffer
						 */
#define	SNDRV_PCM_HW_PARAM_BUFFER_TIME	16	/* Approx duration of buffer
						 * in us
						 */
#define	SNDRV_PCM_HW_PARAM_BUFFER_SIZE	17	/* Size of buffer in frames */
#define	SNDRV_PCM_HW_PARAM_BUFFER_BYTES	18	/* Size of buffer in bytes */
#define	SNDRV_PCM_HW_PARAM_TICK_TIME	19	/* Approx tick duration in us */
#define	SNDRV_PCM_HW_PARAM_FIRST_INTERVAL	SNDRV_PCM_HW_PARAM_SAMPLE_BITS
#define	SNDRV_PCM_HW_PARAM_LAST_INTERVAL	SNDRV_PCM_HW_PARAM_TICK_TIME
```

mask 型音频参数的值在 `masks` 数组中的位置为 (音频参数编号 - 第一个 mask 型音频参数编号)，interval 型参数在 `intervals` 数组中的位置为 (音频参数编号 - 第一个 interval 型音频参数编号)。在使用 tinyalsa API 的应用程序中，各个音频参数表示如下：
```
/** Enumeration of a PCM's hardware parameters.
 * Each of these parameters is either a mask or an interval.
 * @ingroup libtinyalsa-pcm
 */
enum pcm_param
{
    /** A mask that represents the type of read or write method available (e.g. interleaved, mmap). */
    PCM_PARAM_ACCESS,
    /** A mask that represents the @ref pcm_format available (e.g. @ref PCM_FORMAT_S32_LE) */
    PCM_PARAM_FORMAT,
    /** A mask that represents the subformat available */
    PCM_PARAM_SUBFORMAT,
    /** An interval representing the range of sample bits available (e.g. 8 to 32) */
    PCM_PARAM_SAMPLE_BITS,
    /** An interval representing the range of frame bits available (e.g. 8 to 64) */
    PCM_PARAM_FRAME_BITS,
    /** An interval representing the range of channels available (e.g. 1 to 2) */
    PCM_PARAM_CHANNELS,
    /** An interval representing the range of rates available (e.g. 44100 to 192000) */
    PCM_PARAM_RATE,
    PCM_PARAM_PERIOD_TIME,
    /** The number of frames in a period */
    PCM_PARAM_PERIOD_SIZE,
    /** The number of bytes in a period */
    PCM_PARAM_PERIOD_BYTES,
    /** The number of periods for a PCM */
    PCM_PARAM_PERIODS,
    PCM_PARAM_BUFFER_TIME,
    PCM_PARAM_BUFFER_SIZE,
    PCM_PARAM_BUFFER_BYTES,
    PCM_PARAM_TICK_TIME,
}; /* enum pcm_param */
```

硬件设备参数查询及测试的这些接口实现 (位于 `include/uapi/sound/asound.h`) 如下：
```
static void param_init(struct snd_pcm_hw_params *p)
{
    int n;

    memset(p, 0, sizeof(*p));
    for (n = SNDRV_PCM_HW_PARAM_FIRST_MASK;
         n <= SNDRV_PCM_HW_PARAM_LAST_MASK; n++) {
            struct snd_mask *m = param_to_mask(p, n);
            m->bits[0] = ~0;
            m->bits[1] = ~0;
    }
    for (n = SNDRV_PCM_HW_PARAM_FIRST_INTERVAL;
         n <= SNDRV_PCM_HW_PARAM_LAST_INTERVAL; n++) {
            struct snd_interval *i = param_to_interval(p, n);
            i->min = 0;
            i->max = ~0;
    }
    p->rmask = ~0U;
    p->cmask = 0;
    p->info = ~0U;
}
 . . . . . .
/** Gets the hardware parameters of a PCM, without created a PCM handle.
 * @param card The card of the PCM.
 *  The default card is zero.
 * @param device The device of the PCM.
 *  The default device is zero.
 * @param flags Specifies whether the PCM is an input or output.
 *  May be one of the following:
 *   - @ref PCM_IN
 *   - @ref PCM_OUT
 * @return On success, the hardware parameters of the PCM; on failure, NULL.
 * @ingroup libtinyalsa-pcm
 */
struct pcm_params *pcm_params_get(unsigned int card, unsigned int device,
                                  unsigned int flags)
{
    struct snd_pcm_hw_params *params;
    void *snd_node = NULL, *data;
    const struct pcm_ops *ops;
    int fd;

    ops = &hw_ops;
    fd = ops->open(card, device, flags, &data, snd_node);

#ifdef TINYALSA_USES_PLUGINS
    if (fd < 0) {
        int pcm_type;
        snd_node = snd_utils_open_pcm(card, device);
        pcm_type = snd_utils_get_node_type(snd_node);
        if (!snd_node || pcm_type != SND_NODE_TYPE_PLUGIN) {
            fprintf(stderr, "no device (hw/plugin) for card(%u), device(%u)",
                 card, device);
            goto err_open;
        }
        ops = &plug_ops;
        fd = ops->open(card, device, flags, &data, snd_node);
    }
#endif
    if (fd < 0) {
        fprintf(stderr, "cannot open card(%d) device (%d): %s\n",
                card, device, strerror(errno));
        goto err_open;
    }

    params = calloc(1, sizeof(struct snd_pcm_hw_params));
    if (!params)
        goto err_calloc;

    param_init(params);
    if (ops->ioctl(data, SNDRV_PCM_IOCTL_HW_REFINE, params)) {
        fprintf(stderr, "SNDRV_PCM_IOCTL_HW_REFINE error (%d)\n", errno);
        goto err_hw_refine;
    }

#ifdef TINYALSA_USES_PLUGINS
    if (snd_node)
        snd_utils_close_dev_node(snd_node);
#endif
    ops->close(data);

    return (struct pcm_params *)params;

err_hw_refine:
    free(params);
err_calloc:
#ifdef TINYALSA_USES_PLUGINS
    if (snd_node)
        snd_utils_close_dev_node(snd_node);
#endif
    ops->close(data);
err_open:
    return NULL;
}

/** Frees the hardware parameters returned by @ref pcm_params_get.
 * @param pcm_params Hardware parameters of a PCM.
 *  May be NULL.
 * @ingroup libtinyalsa-pcm
 */
void pcm_params_free(struct pcm_params *pcm_params)
{
    struct snd_pcm_hw_params *params = (struct snd_pcm_hw_params *)pcm_params;

    if (params)
        free(params);
}

static int pcm_param_to_alsa(enum pcm_param param)
{
    switch (param) {
    case PCM_PARAM_ACCESS:
        return SNDRV_PCM_HW_PARAM_ACCESS;
    case PCM_PARAM_FORMAT:
        return SNDRV_PCM_HW_PARAM_FORMAT;
    case PCM_PARAM_SUBFORMAT:
        return SNDRV_PCM_HW_PARAM_SUBFORMAT;
    case PCM_PARAM_SAMPLE_BITS:
        return SNDRV_PCM_HW_PARAM_SAMPLE_BITS;
        break;
    case PCM_PARAM_FRAME_BITS:
        return SNDRV_PCM_HW_PARAM_FRAME_BITS;
        break;
    case PCM_PARAM_CHANNELS:
        return SNDRV_PCM_HW_PARAM_CHANNELS;
        break;
    case PCM_PARAM_RATE:
        return SNDRV_PCM_HW_PARAM_RATE;
        break;
    case PCM_PARAM_PERIOD_TIME:
        return SNDRV_PCM_HW_PARAM_PERIOD_TIME;
        break;
    case PCM_PARAM_PERIOD_SIZE:
        return SNDRV_PCM_HW_PARAM_PERIOD_SIZE;
        break;
    case PCM_PARAM_PERIOD_BYTES:
        return SNDRV_PCM_HW_PARAM_PERIOD_BYTES;
        break;
    case PCM_PARAM_PERIODS:
        return SNDRV_PCM_HW_PARAM_PERIODS;
        break;
    case PCM_PARAM_BUFFER_TIME:
        return SNDRV_PCM_HW_PARAM_BUFFER_TIME;
        break;
    case PCM_PARAM_BUFFER_SIZE:
        return SNDRV_PCM_HW_PARAM_BUFFER_SIZE;
        break;
    case PCM_PARAM_BUFFER_BYTES:
        return SNDRV_PCM_HW_PARAM_BUFFER_BYTES;
        break;
    case PCM_PARAM_TICK_TIME:
        return SNDRV_PCM_HW_PARAM_TICK_TIME;
        break;

    default:
        return -1;
    }
}

/** Gets a mask from a PCM's hardware parameters.
 * @param pcm_params A PCM's hardware parameters.
 * @param param The parameter to get.
 * @return If @p pcm_params is NULL or @p param is not a mask, NULL is returned.
 *  Otherwise, the mask associated with @p param is returned.
 * @ingroup libtinyalsa-pcm
 */
const struct pcm_mask *pcm_params_get_mask(const struct pcm_params *pcm_params,
                                     enum pcm_param param)
{
    int p;
    struct snd_pcm_hw_params *params = (struct snd_pcm_hw_params *)pcm_params;
    if (params == NULL) {
        return NULL;
    }

    p = pcm_param_to_alsa(param);
    if (p < 0 || !param_is_mask(p)) {
        return NULL;
    }

    return (const struct pcm_mask *)param_to_mask(params, p);
}

/** Get the minimum of a specified PCM parameter.
 * @param pcm_params A PCM parameters structure.
 * @param param The specified parameter to get the minimum of.
 * @returns On success, the parameter minimum.
 *  On failure, zero.
 */
unsigned int pcm_params_get_min(const struct pcm_params *pcm_params,
                                enum pcm_param param)
{
    struct snd_pcm_hw_params *params = (struct snd_pcm_hw_params *)pcm_params;
    int p;

    if (!params)
        return 0;

    p = pcm_param_to_alsa(param);
    if (p < 0)
        return 0;

    return param_get_min(params, p);
}

/** Get the maximum of a specified PCM parameter.
 * @param pcm_params A PCM parameters structure.
 * @param param The specified parameter to get the maximum of.
 * @returns On success, the parameter maximum.
 *  On failure, zero.
 */
unsigned int pcm_params_get_max(const struct pcm_params *pcm_params,
                                enum pcm_param param)
{
    const struct snd_pcm_hw_params *params = (const struct snd_pcm_hw_params *)pcm_params;
    int p;

    if (!params)
        return 0;

    p = pcm_param_to_alsa(param);
    if (p < 0)
        return 0;

    return param_get_max(params, p);
}

static int pcm_mask_test(const struct pcm_mask *m, unsigned int index)
{
    const unsigned int bitshift = 5; /* for 32 bit integer */
    const unsigned int bitmask = (1 << bitshift) - 1;
    unsigned int element;

    element = index >> bitshift;
    if (element >= ARRAY_SIZE(m->bits))
        return 0; /* for safety, but should never occur */
    return (m->bits[element] >> (index & bitmask)) & 1;
}

static int pcm_mask_to_string(const struct pcm_mask *m, char *string, unsigned int size,
                              char *mask_name,
                              const char * const *bit_array_name, size_t bit_array_size)
{
    unsigned int i;
    unsigned int offset = 0;

    if (m == NULL)
        return 0;
    if (bit_array_size < 32) {
        STRLOG(string, offset, size, "%12s:\t%#08x\n", mask_name, m->bits[0]);
    } else { /* spans two or more bitfields, print with an array index */
        for (i = 0; i < (bit_array_size + 31) >> 5; ++i) {
            STRLOG(string, offset, size, "%9s[%d]:\t%#08x\n",
                   mask_name, i, m->bits[i]);
        }
    }
    for (i = 0; i < bit_array_size; ++i) {
        if (pcm_mask_test(m, i)) {
            STRLOG(string, offset, size, "%12s \t%s\n", "", bit_array_name[i]);
        }
    }
    return offset;
}

int pcm_params_to_string(struct pcm_params *params, char *string, unsigned int size)
{
    const struct pcm_mask *m;
    unsigned int min, max;
    unsigned int clipoffset, offset;

    m = pcm_params_get_mask(params, PCM_PARAM_ACCESS);
    offset = pcm_mask_to_string(m, string, size,
                                 "Access", access_lookup, ARRAY_SIZE(access_lookup));
    m = pcm_params_get_mask(params, PCM_PARAM_FORMAT);
    clipoffset = offset > size ? size : offset;
    offset += pcm_mask_to_string(m, string + clipoffset, size - clipoffset,
                                 "Format", format_lookup, ARRAY_SIZE(format_lookup));
    m = pcm_params_get_mask(params, PCM_PARAM_SUBFORMAT);
    clipoffset = offset > size ? size : offset;
    offset += pcm_mask_to_string(m, string + clipoffset, size - clipoffset,
                                 "Subformat", subformat_lookup, ARRAY_SIZE(subformat_lookup));
    min = pcm_params_get_min(params, PCM_PARAM_RATE);
    max = pcm_params_get_max(params, PCM_PARAM_RATE);
    STRLOG(string, offset, size, "        Rate:\tmin=%uHz\tmax=%uHz\n", min, max);
    min = pcm_params_get_min(params, PCM_PARAM_CHANNELS);
    max = pcm_params_get_max(params, PCM_PARAM_CHANNELS);
    STRLOG(string, offset, size, "    Channels:\tmin=%u\t\tmax=%u\n", min, max);
    min = pcm_params_get_min(params, PCM_PARAM_SAMPLE_BITS);
    max = pcm_params_get_max(params, PCM_PARAM_SAMPLE_BITS);
    STRLOG(string, offset, size, " Sample bits:\tmin=%u\t\tmax=%u\n", min, max);
    min = pcm_params_get_min(params, PCM_PARAM_PERIOD_SIZE);
    max = pcm_params_get_max(params, PCM_PARAM_PERIOD_SIZE);
    STRLOG(string, offset, size, " Period size:\tmin=%u\t\tmax=%u\n", min, max);
    min = pcm_params_get_min(params, PCM_PARAM_PERIODS);
    max = pcm_params_get_max(params, PCM_PARAM_PERIODS);
    STRLOG(string, offset, size, "Period count:\tmin=%u\t\tmax=%u\n", min, max);
    return offset;
}

int pcm_params_format_test(struct pcm_params *params, enum pcm_format format)
{
    unsigned int alsa_format = pcm_format_to_alsa(format);

    if (alsa_format == SNDRV_PCM_FORMAT_S16_LE && format != PCM_FORMAT_S16_LE)
        return 0; /* caution: format not recognized is equivalent to S16_LE */
    return pcm_mask_test(pcm_params_get_mask(params, PCM_PARAM_FORMAT), alsa_format);
}
```

`pcm_params_get()` 接口用于获取特定声卡的特定设备的特定音频流方向的硬件设备参数，其中的音频流方向通过参数 `flags` 指定。这个函数打开音频设备文件，通过 `ioctl()` 的 **SNDRV_PCM_IOCTL_HW_REFINE** 命令获得硬件设备参数，关闭音频设备文件，并返回获得的硬件设备参数。`pcm_params_free()` 接口用于释放获得的硬件设备参数。

`pcm_params_get_mask()`、`pcm_params_get_min()` 和 `pcm_params_get_max()` 等接口分别用于从获得的硬件设备参数中提取 mask 型参数和 interval 型参数的参数值。`pcm_params_format_test()` 接口用于测试获得的硬件设备参数中是否包含特定 `pcm_format`。这些函数的实现中，将传入的 tinyalsa 参数表示转为 Linux 内核 ALSA 的参数表示，并从 `struct snd_pcm_hw_params` 中获取请求的参数值。

`pcm_params_to_string()` 接口用于生成硬件设备参数的字符串表示。

tinyalsa 的 PCM 相关 API 的第二部分是音频流相关 API。

## Tinyalsa 的 PCM API 的音频流相关部分

tinyalsa 的 PCM 相关 API 的音频流相关部分又可以分为音频流生命周期管理 API 和音频流状态和参数查询 API 两部分，其中的音频流生命周期管理 API 有如下这些：
```
struct pcm;

struct pcm *pcm_open(unsigned int card,
                     unsigned int device,
                     unsigned int flags,
                     const struct pcm_config *config);

struct pcm *pcm_open_by_name(const char *name,
                             unsigned int flags,
                             const struct pcm_config *config);

int pcm_close(struct pcm *pcm);
 . . . . . .
int pcm_set_config(struct pcm *pcm, const struct pcm_config *config);
 . . . . . .
int pcm_writei(struct pcm *pcm, const void *data, unsigned int frame_count) TINYALSA_WARN_UNUSED_RESULT;

int pcm_readi(struct pcm *pcm, void *data, unsigned int frame_count) TINYALSA_WARN_UNUSED_RESULT;

int pcm_write(struct pcm *pcm, const void *data, unsigned int count) TINYALSA_DEPRECATED;

int pcm_read(struct pcm *pcm, void *data, unsigned int count) TINYALSA_DEPRECATED;

int pcm_mmap_write(struct pcm *pcm, const void *data, unsigned int count) TINYALSA_DEPRECATED;

int pcm_mmap_read(struct pcm *pcm, void *data, unsigned int count) TINYALSA_DEPRECATED;

int pcm_mmap_begin(struct pcm *pcm, void **areas, unsigned int *offset, unsigned int *frames);

int pcm_mmap_commit(struct pcm *pcm, unsigned int offset, unsigned int frames);
 . . . . . .
int pcm_link(struct pcm *pcm1, struct pcm *pcm2);

int pcm_unlink(struct pcm *pcm);

int pcm_prepare(struct pcm *pcm);

int pcm_start(struct pcm *pcm);

int pcm_stop(struct pcm *pcm);

int pcm_wait(struct pcm *pcm, int timeout);
 . . . . . .
int pcm_ioctl(struct pcm *pcm, int code, ...) TINYALSA_DEPRECATED;
```

tinyalsa 用一个其定义应用程序不可见的结构体 `struct pcm` 向应用程序表示音频流，`struct pcm` 结构体的指针可以理解为音频流的句柄。`struct pcm` 结构体定义 (位于 `tinyalsa/src/pcm.c`) 如下：
```
/** A PCM handle.
 * @ingroup libtinyalsa-pcm
 */
struct pcm {
    /** The PCM's file descriptor */
    int fd;
    /** Flags that were passed to @ref pcm_open */
    unsigned int flags;
    /** The number of (under/over)runs that have occured */
    int xruns;
    /** Size of the buffer */
    unsigned int buffer_size;
    /** The boundary for ring buffer pointers */
    unsigned long boundary;
    /** Description of the last error that occured */
    char error[PCM_ERROR_MAX];
    /** Configuration that was passed to @ref pcm_open */
    struct pcm_config config;
    struct snd_pcm_mmap_status *mmap_status;
    struct snd_pcm_mmap_control *mmap_control;
    struct snd_pcm_sync_ptr *sync_ptr;
    void *mmap_buffer;
    unsigned int noirq_frames_per_msec;
    /** The delay of the PCM, in terms of frames */
    long pcm_delay;
    /** The subdevice corresponding to the PCM */
    unsigned int subdevice;
    /** Pointer to the pcm ops, either hw or plugin */
    const struct pcm_ops *ops;
    /** Private data for pcm_hw or pcm_plugin */
    void *data;
    /** Pointer to the pcm node from snd card definition */
    struct snd_node *snd_node;
};
```

tinyalsa 的音频流生命周期管理 API 不是正交的，这些 API 的非正交主要发生在数据传递相关的 API 上，`pcm_write()`、`pcm_read()`、`pcm_mmap_write()` 和 `pcm_mmap_read()` 这几个接口已经被标记为废弃的，它们在功能上，基本上与 `pcm_writei()` 和 `pcm_readi()` 相同，都用于发送或接收数据，在实现上，它们通过 `pcm_writei()` 和 `pcm_readi()` 完成其主要职责，相关代码 (位于 `tinyalsa/src/pcm.c`) 具体如下：
```
int pcm_mmap_write(struct pcm *pcm, const void *data, unsigned int count)
{
    if ((~pcm->flags) & (PCM_OUT | PCM_MMAP))
        return -EINVAL;

    unsigned int frames = pcm_bytes_to_frames(pcm, count);
    int res = pcm_writei(pcm, (void *) data, frames);

    if (res < 0) {
        return res;
    }

    return (unsigned int) res == frames ? 0 : -EIO;
}

int pcm_mmap_read(struct pcm *pcm, void *data, unsigned int count)
{
    if ((~pcm->flags) & (PCM_IN | PCM_MMAP))
        return -EINVAL;

    unsigned int frames = pcm_bytes_to_frames(pcm, count);
    int res = pcm_readi(pcm, data, frames);

    if (res < 0) {
        return res;
    }

    return (unsigned int) res == frames ? 0 : -EIO;
}
 . . . . . .
/** Writes audio samples to PCM.
 * If the PCM has not been started, it is started in this function.
 * This function is only valid for PCMs opened with the @ref PCM_OUT flag.
 * This function is not valid for PCMs opened with the @ref PCM_MMAP flag.
 * @param pcm A PCM handle.
 * @param data The audio sample array
 * @param count The number of bytes occupied by the sample array.
 * @return On success, this function returns zero; otherwise, a negative number.
 * @deprecated
 * @ingroup libtinyalsa-pcm
 */
int pcm_write(struct pcm *pcm, const void *data, unsigned int count)
{
    unsigned int requested_frames = pcm_bytes_to_frames(pcm, count);
    int ret = pcm_writei(pcm, data, requested_frames);

    if (ret < 0)
        return ret;

    return ((unsigned int )ret == requested_frames) ? 0 : -EIO;
}

/** Reads audio samples from PCM.
 * If the PCM has not been started, it is started in this function.
 * This function is only valid for PCMs opened with the @ref PCM_IN flag.
 * This function is not valid for PCMs opened with the @ref PCM_MMAP flag.
 * @param pcm A PCM handle.
 * @param data The audio sample array
 * @param count The number of bytes occupied by the sample array.
 * @return On success, this function returns zero; otherwise, a negative number.
 * @deprecated
 * @ingroup libtinyalsa-pcm
 */
int pcm_read(struct pcm *pcm, void *data, unsigned int count)
{
    unsigned int requested_frames = pcm_bytes_to_frames(pcm, count);
    int ret = pcm_readi(pcm, data, requested_frames);

    if (ret < 0)
        return ret;

    return ((unsigned int )ret == requested_frames) ? 0 : -EIO;
}
```

`pcm_write()`、`pcm_read()`、`pcm_mmap_write()` 和 `pcm_mmap_read()` 与 `pcm_writei()` 和 `pcm_readi()` 的主要区别在于，前者用字节数表示要传递的数据量，后者则用帧数表示。在接口定义上，`pcm_write()` 和 `pcm_read()` 用于以非 mmap 的方式和 Linux 内核交换数据，`pcm_mmap_write()` 和 `pcm_mmap_read()` 用于以 mmap 的方式和 Linux 内核交换数据，因而 `pcm_mmap_write()` 和 `pcm_mmap_read()` 函数的实现中检查了 `flags` 是否包含 `PCM_MMAP`。但实现上，除了要传递的数据量的描述方式外，`pcm_write()` 和 `pcm_read()` 与 `pcm_writei()` 和 `pcm_readi()` 完全相同。

tinyalsa 用一组接口 `pcm_writei()` 和 `pcm_readi()` 支持了 mmap 和非 mmap 两种方式与 Linux 内核的数据交换，与 Linux 内核的实际的数据交换方式通过 `flags` 来确定。

另外，`pcm_writei()` 和 `pcm_readi()` 以 mmap 方式与 Linux 内核交换数据时，需借助 `pcm_mmap_begin()`、`pcm_mmap_commit()` 和 `pcm_mmap_avail()` 等接口来完成。

音频流状态和参数查询 API 有如下这些：
```
int pcm_is_ready(const struct pcm *pcm);

unsigned int pcm_get_channels(const struct pcm *pcm);

const struct pcm_config * pcm_get_config(const struct pcm *pcm);

unsigned int pcm_get_rate(const struct pcm *pcm);

enum pcm_format pcm_get_format(const struct pcm *pcm);

int pcm_get_file_descriptor(const struct pcm *pcm);

const char *pcm_get_error(const struct pcm *pcm);
 . . . . . .
unsigned int pcm_format_to_bits(enum pcm_format format);

unsigned int pcm_get_buffer_size(const struct pcm *pcm);

unsigned int pcm_frames_to_bytes(const struct pcm *pcm, unsigned int frames);

unsigned int pcm_bytes_to_frames(const struct pcm *pcm, unsigned int bytes);

int pcm_get_htimestamp(struct pcm *pcm, unsigned int *avail, struct timespec *tstamp);

unsigned int pcm_get_subdevice(const struct pcm *pcm);
 . . . . . .
int pcm_mmap_avail(struct pcm *pcm);

int pcm_mmap_get_hw_ptr(struct pcm* pcm, unsigned int *hw_ptr, struct timespec *tstamp);

int pcm_get_poll_fd(struct pcm *pcm);
 . . . . . .
long pcm_get_delay(struct pcm *pcm);
```

open 是音频流的生命周期开始的地方。tinyalsa 提供了 `pcm_open()` 和 `pcm_open_by_name()` 两个接口来打开一个音频流，它们以不同的方式指定音频设备，前者通过声卡编号和设备编号来指定，后者通过一个字符串设备名来指定。`pcm_open_by_name()` 函数定义 (位于 `tinyalsa/src/pcm.c`) 如下：
```
struct pcm *pcm_open_by_name(const char *name,
                             unsigned int flags,
                             const struct pcm_config *config)
{
    unsigned int card, device;
    if (name[0] != 'h' || name[1] != 'w' || name[2] != ':') {
        oops(&bad_pcm, 0, "name format is not matched");
        return &bad_pcm;
    } else if (sscanf(&name[3], "%u,%u", &card, &device) != 2) {
        oops(&bad_pcm, 0, "name format is not matched");
        return &bad_pcm;
    }
    return pcm_open(card, device, flags, config);
}
```

字符串设备名的格式为 'hw:card,device'，`pcm_open_by_name()` 函数从设备名参数中解析提取声卡编号和设备编号，并调用 `pcm_open()` 函数打开音频流。以如下方式打开音频流：
```
 . . . . pcm = pcm_open_by_name("hw:0,0",  flags, config);
```

等价于如下方式打开音频流：
```
 . . . . pcm = pcm_open(0,  0, flags, config);
```

`pcm_open()` 和 `pcm_open_by_name()` 两个接口的 `flags` 参数指定了 pcm 的特性和功能，它是如下这些标记的按位或：
```
/** A flag that specifies that the PCM is an output.
 * May not be bitwise AND'd with @ref PCM_IN.
 * Used in @ref pcm_open.
 * @ingroup libtinyalsa-pcm
 */
#define PCM_OUT 0x00000000

/** Specifies that the PCM is an input.
 * May not be bitwise AND'd with @ref PCM_OUT.
 * Used in @ref pcm_open.
 * @ingroup libtinyalsa-pcm
 */
#define PCM_IN 0x10000000

/** Specifies that the PCM will use mmap read and write methods.
 * Used in @ref pcm_open.
 * @ingroup libtinyalsa-pcm
 */
#define PCM_MMAP 0x00000001

/** Specifies no interrupt requests.
 * May only be bitwise AND'd with @ref PCM_MMAP.
 * Used in @ref pcm_open.
 * @ingroup libtinyalsa-pcm
 */
#define PCM_NOIRQ 0x00000002

/** When set, calls to @ref pcm_write
 * for a playback stream will not attempt
 * to restart the stream in the case of an
 * underflow, but will return -EPIPE instead.
 * After the first -EPIPE error, the stream
 * is considered to be stopped, and a second
 * call to pcm_write will attempt to restart
 * the stream.
 * Used in @ref pcm_open.
 * @ingroup libtinyalsa-pcm
 */
#define PCM_NORESTART 0x00000004

/** Specifies monotonic timestamps.
 * Used in @ref pcm_open.
 * @ingroup libtinyalsa-pcm
 */
#define PCM_MONOTONIC 0x00000008

/** If used with @ref pcm_open and @ref pcm_params_get,
 * it will not cause the function to block if
 * the PCM is not available. It will also cause
 * the functions @ref pcm_readi and @ref pcm_writei
 * to exit if they would cause the caller to wait.
 * @ingroup libtinyalsa-pcm
 * */
#define PCM_NONBLOCK 0x00000010
```

`pcm_open()` 和 `pcm_open_by_name()` 两个接口的 `config` 参数指定了打开 PCM 所使用的硬件和软件参数。

`pcm_open()` 函数定义 (位于 `tinyalsa/src/pcm.c`) 如下：
```
/** Sets the PCM configuration.
 * @param pcm A PCM handle.
 * @param config The configuration to use for the
 *  PCM. This parameter may be NULL, in which case
 *  the default configuration is used.
 * @returns Zero on success, a negative errno value
 *  on failure.
 * @ingroup libtinyalsa-pcm
 * */
int pcm_set_config(struct pcm *pcm, const struct pcm_config *config)
{
    if (pcm == NULL)
        return -EFAULT;
    else if (config == NULL) {
        config = &pcm->config;
        pcm->config.channels = 2;
        pcm->config.rate = 48000;
        pcm->config.period_size = 1024;
        pcm->config.period_count = 4;
        pcm->config.format = PCM_FORMAT_S16_LE;
        pcm->config.start_threshold = config->period_count * config->period_size;
        pcm->config.stop_threshold = config->period_count * config->period_size;
        pcm->config.silence_threshold = 0;
        pcm->config.silence_size = 0;
    } else
        pcm->config = *config;

    struct snd_pcm_hw_params params;
    param_init(&params);
    param_set_mask(&params, SNDRV_PCM_HW_PARAM_FORMAT,
                   pcm_format_to_alsa(config->format));
    param_set_min(&params, SNDRV_PCM_HW_PARAM_PERIOD_SIZE, config->period_size);
    param_set_int(&params, SNDRV_PCM_HW_PARAM_CHANNELS,
                  config->channels);
    param_set_int(&params, SNDRV_PCM_HW_PARAM_PERIODS, config->period_count);
    param_set_int(&params, SNDRV_PCM_HW_PARAM_RATE, config->rate);

    if (pcm->flags & PCM_NOIRQ) {

        if (!(pcm->flags & PCM_MMAP)) {
            oops(pcm, EINVAL, "noirq only currently supported with mmap().");
            return -EINVAL;
        }

        params.flags |= SNDRV_PCM_HW_PARAMS_NO_PERIOD_WAKEUP;
        pcm->noirq_frames_per_msec = config->rate / 1000;
    }

    if (pcm->flags & PCM_MMAP)
        param_set_mask(&params, SNDRV_PCM_HW_PARAM_ACCESS,
                   SNDRV_PCM_ACCESS_MMAP_INTERLEAVED);
    else
        param_set_mask(&params, SNDRV_PCM_HW_PARAM_ACCESS,
                   SNDRV_PCM_ACCESS_RW_INTERLEAVED);

    if (pcm->ops->ioctl(pcm->data, SNDRV_PCM_IOCTL_HW_PARAMS, &params)) {
        int errno_copy = errno;
        oops(pcm, errno, "cannot set hw params");
        return -errno_copy;
    }

    /* get our refined hw_params */
    pcm->config.period_size = param_get_int(&params, SNDRV_PCM_HW_PARAM_PERIOD_SIZE);
    pcm->config.period_count = param_get_int(&params, SNDRV_PCM_HW_PARAM_PERIODS);
    pcm->buffer_size = config->period_count * config->period_size;

    if (pcm->flags & PCM_MMAP) {
        pcm->mmap_buffer = pcm->ops->mmap(pcm->data, NULL, pcm_frames_to_bytes(pcm, pcm->buffer_size),
                                PROT_READ | PROT_WRITE, MAP_SHARED, 0);
        if (pcm->mmap_buffer == MAP_FAILED) {
            int errno_copy = errno;
            oops(pcm, errno, "failed to mmap buffer %d bytes\n",
                 pcm_frames_to_bytes(pcm, pcm->buffer_size));
            return -errno_copy;
        }
    }

    struct snd_pcm_sw_params sparams;
    memset(&sparams, 0, sizeof(sparams));
    sparams.tstamp_mode = SNDRV_PCM_TSTAMP_ENABLE;
    sparams.period_step = 1;
    sparams.avail_min = config->period_size;

    if (!config->start_threshold) {
        if (pcm->flags & PCM_IN)
            pcm->config.start_threshold = sparams.start_threshold = 1;
        else
            pcm->config.start_threshold = sparams.start_threshold =
                config->period_count * config->period_size / 2;
    } else
        sparams.start_threshold = config->start_threshold;

    /* pick a high stop threshold - todo: does this need further tuning */
    if (!config->stop_threshold) {
        if (pcm->flags & PCM_IN)
            pcm->config.stop_threshold = sparams.stop_threshold =
                config->period_count * config->period_size * 10;
        else
            pcm->config.stop_threshold = sparams.stop_threshold =
                config->period_count * config->period_size;
    }
    else
        sparams.stop_threshold = config->stop_threshold;

    sparams.xfer_align = config->period_size / 2; /* needed for old kernels */
    sparams.silence_size = config->silence_size;
    sparams.silence_threshold = config->silence_threshold;

    if (pcm->ops->ioctl(pcm->data, SNDRV_PCM_IOCTL_SW_PARAMS, &sparams)) {
        int errno_copy = errno;
        oops(pcm, errno, "cannot set sw params");
        return -errno_copy;
    }

    pcm->boundary = sparams.boundary;
    return 0;
}
 . . . . . .
static int pcm_hw_mmap_status(struct pcm *pcm)
{
    if (pcm->sync_ptr)
        return 0;

    int page_size = sysconf(_SC_PAGE_SIZE);
    pcm->mmap_status = pcm->ops->mmap(pcm->data, NULL, page_size, PROT_READ, MAP_SHARED,
                            SNDRV_PCM_MMAP_OFFSET_STATUS);
    if (pcm->mmap_status == MAP_FAILED)
        pcm->mmap_status = NULL;
    if (!pcm->mmap_status)
        goto mmap_error;

    pcm->mmap_control = pcm->ops->mmap(pcm->data, NULL, page_size, PROT_READ | PROT_WRITE,
                             MAP_SHARED, SNDRV_PCM_MMAP_OFFSET_CONTROL);
    if (pcm->mmap_control == MAP_FAILED)
        pcm->mmap_control = NULL;
    if (!pcm->mmap_control) {
        pcm->ops->munmap(pcm->data, pcm->mmap_status, page_size);
        pcm->mmap_status = NULL;
        goto mmap_error;
    }

    return 0;

mmap_error:

    pcm->sync_ptr = calloc(1, sizeof(*pcm->sync_ptr));
    if (!pcm->sync_ptr)
        return -ENOMEM;
    pcm->mmap_status = &pcm->sync_ptr->s.status;
    pcm->mmap_control = &pcm->sync_ptr->c.control;

    return 0;
}
 . . . . . .
struct pcm *pcm_open(unsigned int card, unsigned int device,
                     unsigned int flags, const struct pcm_config *config)
{
    struct pcm *pcm;
    struct snd_pcm_info info;
    int rc;

    pcm = calloc(1, sizeof(struct pcm));
    if (!pcm) {
        oops(&bad_pcm, ENOMEM, "can't allocate PCM object");
        return &bad_pcm;
    }

    /* Default to hw_ops, attemp plugin open only if hw (/dev/snd/pcm*) open fails */
    pcm->ops = &hw_ops;
    pcm->fd = pcm->ops->open(card, device, flags, &pcm->data, NULL);

#ifdef TINYALSA_USES_PLUGINS
    if (pcm->fd < 0) {
        int pcm_type;
        pcm->snd_node = snd_utils_open_pcm(card, device);
        pcm_type = snd_utils_get_node_type(pcm->snd_node);
        if (!pcm->snd_node || pcm_type != SND_NODE_TYPE_PLUGIN) {
            oops(&bad_pcm, ENODEV, "no device (hw/plugin) for card(%u), device(%u)",
                 card, device);
            goto fail_close_dev_node;
        }
        pcm->ops = &plug_ops;
        pcm->fd = pcm->ops->open(card, device, flags, &pcm->data, pcm->snd_node);
    }
#endif
    if (pcm->fd < 0) {
        oops(&bad_pcm, errno, "cannot open device (%u) for card (%u)",
             device, card);
        goto fail_close_dev_node;
    }

    pcm->flags = flags;

    if (pcm->ops->ioctl(pcm->data, SNDRV_PCM_IOCTL_INFO, &info)) {
        oops(&bad_pcm, errno, "cannot get info");
        goto fail_close;
    }
    pcm->subdevice = info.subdevice;

    if (pcm_set_config(pcm, config) != 0) {
        memcpy(bad_pcm.error, pcm->error, sizeof(pcm->error));
        goto fail_close;
    }

    rc = pcm_hw_mmap_status(pcm);
    if (rc < 0) {
        oops(&bad_pcm, errno, "mmap status failed");
        goto fail;
    }

#ifdef SNDRV_PCM_IOCTL_TTSTAMP
    if (pcm->flags & PCM_MONOTONIC) {
        int arg = SNDRV_PCM_TSTAMP_TYPE_MONOTONIC;
        rc = pcm->ops->ioctl(pcm->data, SNDRV_PCM_IOCTL_TTSTAMP, &arg);
        if (rc < 0) {
            oops(&bad_pcm, errno, "cannot set timestamp type");
            goto fail;
        }
    }
#endif

    pcm->xruns = 0;
    return pcm;

fail:
    pcm_hw_munmap_status(pcm);
    if (flags & PCM_MMAP)
        pcm->ops->munmap(pcm->data, pcm->mmap_buffer, pcm_frames_to_bytes(pcm, pcm->buffer_size));
fail_close:
    pcm->ops->close(pcm->data);
fail_close_dev_node:
#ifdef TINYALSA_USES_PLUGINS
    if (pcm->snd_node)
        snd_utils_close_dev_node(pcm->snd_node);
#endif
    free(pcm);
    return &bad_pcm;
}
```

打开 PCM 的主要过程如下：

1. 打开 PCM 设备文件，PCM 设备文件路径为 `/dev/snd/pcmC%uD%u%c`，如前面在 `pcm_hw_open()` 函数中看到的那样。

2. 通过 `ioctl()` 的 **SNDRV_PCM_IOCTL_INFO** 命令获得 PCM 信息，这里主要是想获得 PCM 设备的次设备号。内核中，这个 `ioctl()` 命令的实现 (位于 `sound/core/pcm_native.c`) 如下：
```
int snd_pcm_info(struct snd_pcm_substream *substream, struct snd_pcm_info *info)
{
	struct snd_pcm *pcm = substream->pcm;
	struct snd_pcm_str *pstr = substream->pstr;

	memset(info, 0, sizeof(*info));
	info->card = pcm->card->number;
	info->device = pcm->device;
	info->stream = substream->stream;
	info->subdevice = substream->number;
	strlcpy(info->id, pcm->id, sizeof(info->id));
	strlcpy(info->name, pcm->name, sizeof(info->name));
	info->dev_class = pcm->dev_class;
	info->dev_subclass = pcm->dev_subclass;
	info->subdevices_count = pstr->substream_count;
	info->subdevices_avail = pstr->substream_count - pstr->substream_opened;
	strlcpy(info->subname, substream->name, sizeof(info->subname));

	return 0;
}

int snd_pcm_info_user(struct snd_pcm_substream *substream,
		      struct snd_pcm_info __user * _info)
{
	struct snd_pcm_info *info;
	int err;

	info = kmalloc(sizeof(*info), GFP_KERNEL);
	if (! info)
		return -ENOMEM;
	err = snd_pcm_info(substream, info);
	if (err >= 0) {
		if (copy_to_user(_info, info, sizeof(*info)))
			err = -EFAULT;
	}
	kfree(info);
	return err;
}
 . . . . . .
static int snd_pcm_common_ioctl(struct file *file,
				 struct snd_pcm_substream *substream,
				 unsigned int cmd, void __user *arg)
{
	struct snd_pcm_file *pcm_file = file->private_data;
	int res;

	if (PCM_RUNTIME_CHECK(substream))
		return -ENXIO;

	res = snd_power_wait(substream->pcm->card, SNDRV_CTL_POWER_D0);
	if (res < 0)
		return res;

	switch (cmd) {
	case SNDRV_PCM_IOCTL_PVERSION:
		return put_user(SNDRV_PCM_VERSION, (int __user *)arg) ? -EFAULT : 0;
	case SNDRV_PCM_IOCTL_INFO:
		return snd_pcm_info_user(substream, arg);
```

从内核往用户空间拷贝数据总是很麻烦，需要先在内核空间分配一个相同类型的数据结构，之后填充该数据结构，随后通过 `copy_to_user()` 将填充之后的数据结构拷贝到用户空间内存，最后释放分配的内核空间数据结构。

3. 调用 `pcm_set_config()` 接口设置音频参数。通过 `ioctl()` 的 **SNDRV_PCM_IOCTL_HW_PARAMS** 命令设置硬件参数，如音频采样格式，通道数，采样率等；如果 `flags` 包含 **PCM_MMAP** 标记，则将用于存放音频数据的 DMA 缓冲区 mmap 内存映射到用户空间；过 `ioctl()` 的 **SNDRV_PCM_IOCTL_SW_PARAMS** 命令设置软件参数，如 `start_threshold`，`stop_threshold`，`silence_size` 等。大多数情况下，`pcm_set_config()` 接口不单独使用，而是在 open 中使用。

4. 调用 `pcm_hw_mmap_status()` 函数为音频流获得 `struct snd_pcm_mmap_status` 和 `struct snd_pcm_mmap_control`。这两个结构由 Linux 内核定义，具体定义 (位于 `include/uapi/sound/asound.h`) 如下：
```
struct __snd_pcm_mmap_status {
	snd_pcm_state_t state;		/* RO: state - SNDRV_PCM_STATE_XXXX */
	int pad1;			/* Needed for 64 bit alignment */
	snd_pcm_uframes_t hw_ptr;	/* RO: hw ptr (0...boundary-1) */
	struct __snd_timespec tstamp;	/* Timestamp */
	snd_pcm_state_t suspended_state; /* RO: suspended stream state */
	struct __snd_timespec audio_tstamp; /* from sample counter or wall clock */
};

struct __snd_pcm_mmap_control {
	snd_pcm_uframes_t appl_ptr;	/* RW: appl ptr (0...boundary-1) */
	snd_pcm_uframes_t avail_min;	/* RW: min available frames for wakeup */
};

#define SNDRV_PCM_SYNC_PTR_HWSYNC	(1<<0)	/* execute hwsync */
#define SNDRV_PCM_SYNC_PTR_APPL		(1<<1)	/* get appl_ptr from driver (r/w op) */
#define SNDRV_PCM_SYNC_PTR_AVAIL_MIN	(1<<2)	/* get avail_min from driver */

struct snd_pcm_sync_ptr {
	unsigned int flags;
	union {
		struct snd_pcm_mmap_status status;
		unsigned char reserved[64];
	} s;
	union {
		struct snd_pcm_mmap_control control;
		unsigned char reserved[64];
	} c;
};
```

通过 ALSA 播放音频数据的过程大体为，应用程序将数据拷贝到 DMA 缓冲区，并更新指向 DMA 缓冲区的应用程序指针，硬件驱动程序将 DMA 缓冲区的数据发送给硬件，并更新硬件指针，应用程序指针和硬件指针分别为 DMA 缓冲区这个循环缓冲区的写指针和读指针。无论是否以 mmap 的方式来完成应用程序与 Linux 内核的数据交换，这个过程都是一样的。

应用程序以 Read/Write 的方式与 Linux 内核交换数据时，先将音频数据放在一块用户空间内存缓冲区中，之后通过 `ioctl()` 命令或 `write()` 操作将音频数据送给内核，内核将用户空间内存缓冲区中的数据拷贝到 DMA 缓冲区，并更新指向 DMA 缓冲区的应用程序指针。以 mmap 的方式与 Linux 内核交换数据时，则是将 DMA 缓冲区内存映射到用户空间，应用程序向 DMA 缓冲区中写入音频数据，应用程序更新应用程序指针，应用程序通过 `ioctl()` 命令将应用程序指针等信息同步给内核。使用 mmap 时，可以少一次音频数据在用户空间内存和 DMA 缓冲区之间的内存拷贝。

`struct snd_pcm_mmap_status` 和 `struct snd_pcm_mmap_control` 用于维护应用程序指针和硬件指针等音频流相关状态信息。

`pcm_hw_mmap_status()` 函数首先尝试用 `mmap()` 操作获得 `struct snd_pcm_mmap_status` 和 `struct snd_pcm_mmap_control`，为 `mmap()` 操作传入的 `offset` 分别为 **SNDRV_PCM_MMAP_OFFSET_STATUS** 和 **SNDRV_PCM_MMAP_OFFSET_CONTROL**。这种方式直接将内核空间的数据结构映射到用户空间，但这不是每种硬件架构都支持的，如 Linux 内核中 pcm 设备文件 mmap 操作的实现 (位于 `sound/core/pcm_native.c`) 如下：
```
/*
 * Only on coherent architectures, we can mmap the status and the control records
 * for effcient data transfer.  On others, we have to use HWSYNC ioctl...
 */
#if defined(CONFIG_X86) || defined(CONFIG_PPC) || defined(CONFIG_ALPHA)
/*
 * mmap status record
 */
static vm_fault_t snd_pcm_mmap_status_fault(struct vm_fault *vmf)
{
	struct snd_pcm_substream *substream = vmf->vma->vm_private_data;
	struct snd_pcm_runtime *runtime;
	
	if (substream == NULL)
		return VM_FAULT_SIGBUS;
	runtime = substream->runtime;
	vmf->page = virt_to_page(runtime->status);
	get_page(vmf->page);
	return 0;
}

static const struct vm_operations_struct snd_pcm_vm_ops_status =
{
	.fault =	snd_pcm_mmap_status_fault,
};

static int snd_pcm_mmap_status(struct snd_pcm_substream *substream, struct file *file,
			       struct vm_area_struct *area)
{
	long size;
	if (!(area->vm_flags & VM_READ))
		return -EINVAL;
	size = area->vm_end - area->vm_start;
	if (size != PAGE_ALIGN(sizeof(struct snd_pcm_mmap_status)))
		return -EINVAL;
	area->vm_ops = &snd_pcm_vm_ops_status;
	area->vm_private_data = substream;
	area->vm_flags |= VM_DONTEXPAND | VM_DONTDUMP;
	return 0;
}

/*
 * mmap control record
 */
static vm_fault_t snd_pcm_mmap_control_fault(struct vm_fault *vmf)
{
	struct snd_pcm_substream *substream = vmf->vma->vm_private_data;
	struct snd_pcm_runtime *runtime;
	
	if (substream == NULL)
		return VM_FAULT_SIGBUS;
	runtime = substream->runtime;
	vmf->page = virt_to_page(runtime->control);
	get_page(vmf->page);
	return 0;
}

static const struct vm_operations_struct snd_pcm_vm_ops_control =
{
	.fault =	snd_pcm_mmap_control_fault,
};

static int snd_pcm_mmap_control(struct snd_pcm_substream *substream, struct file *file,
				struct vm_area_struct *area)
{
	long size;
	if (!(area->vm_flags & VM_READ))
		return -EINVAL;
	size = area->vm_end - area->vm_start;
	if (size != PAGE_ALIGN(sizeof(struct snd_pcm_mmap_control)))
		return -EINVAL;
	area->vm_ops = &snd_pcm_vm_ops_control;
	area->vm_private_data = substream;
	area->vm_flags |= VM_DONTEXPAND | VM_DONTDUMP;
	return 0;
}

static bool pcm_status_mmap_allowed(struct snd_pcm_file *pcm_file)
{
	/* See pcm_control_mmap_allowed() below.
	 * Since older alsa-lib requires both status and control mmaps to be
	 * coupled, we have to disable the status mmap for old alsa-lib, too.
	 */
	if (pcm_file->user_pversion < SNDRV_PROTOCOL_VERSION(2, 0, 14) &&
	    (pcm_file->substream->runtime->hw.info & SNDRV_PCM_INFO_SYNC_APPLPTR))
		return false;
	return true;
}

static bool pcm_control_mmap_allowed(struct snd_pcm_file *pcm_file)
{
	if (pcm_file->no_compat_mmap)
		return false;
	/* Disallow the control mmap when SYNC_APPLPTR flag is set;
	 * it enforces the user-space to fall back to snd_pcm_sync_ptr(),
	 * thus it effectively assures the manual update of appl_ptr.
	 */
	if (pcm_file->substream->runtime->hw.info & SNDRV_PCM_INFO_SYNC_APPLPTR)
		return false;
	return true;
}

#else /* ! coherent mmap */
/*
 * don't support mmap for status and control records.
 */
#define pcm_status_mmap_allowed(pcm_file)	false
#define pcm_control_mmap_allowed(pcm_file)	false

static int snd_pcm_mmap_status(struct snd_pcm_substream *substream, struct file *file,
			       struct vm_area_struct *area)
{
	return -ENXIO;
}
static int snd_pcm_mmap_control(struct snd_pcm_substream *substream, struct file *file,
				struct vm_area_struct *area)
{
	return -ENXIO;
}
#endif /* coherent mmap */
 . . . . . .
static int snd_pcm_mmap(struct file *file, struct vm_area_struct *area)
{
	struct snd_pcm_file * pcm_file;
	struct snd_pcm_substream *substream;	
	unsigned long offset;
	
	pcm_file = file->private_data;
	substream = pcm_file->substream;
	if (PCM_RUNTIME_CHECK(substream))
		return -ENXIO;

	offset = area->vm_pgoff << PAGE_SHIFT;
	switch (offset) {
	case SNDRV_PCM_MMAP_OFFSET_STATUS_OLD:
		if (pcm_file->no_compat_mmap || !IS_ENABLED(CONFIG_64BIT))
			return -ENXIO;
		fallthrough;
	case SNDRV_PCM_MMAP_OFFSET_STATUS_NEW:
		if (!pcm_status_mmap_allowed(pcm_file))
			return -ENXIO;
		return snd_pcm_mmap_status(substream, file, area);
	case SNDRV_PCM_MMAP_OFFSET_CONTROL_OLD:
		if (pcm_file->no_compat_mmap || !IS_ENABLED(CONFIG_64BIT))
			return -ENXIO;
		fallthrough;
	case SNDRV_PCM_MMAP_OFFSET_CONTROL_NEW:
		if (!pcm_control_mmap_allowed(pcm_file))
			return -ENXIO;
		return snd_pcm_mmap_control(substream, file, area);
	default:
		return snd_pcm_mmap_data(substream, file, area);
	}
	return 0;
}
```

只有在 X86、PPC 和 Alpha 这种 coherent 架构上可以将内核的 `struct snd_pcm_mmap_status` 和 `struct snd_pcm_mmap_control` 映射到用户空间。

`pcm_hw_mmap_status()` 函数尝试用 `mmap()` 操作获得 `struct snd_pcm_mmap_status` 和 `struct snd_pcm_mmap_control` 失败时，则在应用程序的堆内存上直接分配它们。

和内核同步由 `struct snd_pcm_mmap_status` 和 `struct snd_pcm_mmap_control` 表示的音频流相关状态信息，是 Read/Write 和 mmap 两种数据传递方式都需要的，因而 `pcm_hw_mmap_status()` 函数的调用是无条件的。

打开音频流之后，即可获取音频流状态并查询参数了，相关的 API 实现 (位于 `tinyalsa/src/pcm.c`) 如下：
```
/** Gets the buffer size of the PCM.
 * @param pcm A PCM handle.
 * @return The buffer size of the PCM.
 * @ingroup libtinyalsa-pcm
 */
unsigned int pcm_get_buffer_size(const struct pcm *pcm)
{
    return pcm->buffer_size;
}

/** Gets the channel count of the PCM.
 * @param pcm A PCM handle.
 * @return The channel count of the PCM.
 * @ingroup libtinyalsa-pcm
 */
unsigned int pcm_get_channels(const struct pcm *pcm)
{
    return pcm->config.channels;
}

/** Gets the PCM configuration.
 * @param pcm A PCM handle.
 * @return The PCM configuration.
 *  This function only returns NULL if
 *  @p pcm is NULL.
 * @ingroup libtinyalsa-pcm
 * */
const struct pcm_config * pcm_get_config(const struct pcm *pcm)
{
    if (pcm == NULL)
        return NULL;
    return &pcm->config;
}

/** Gets the rate of the PCM.
 * The rate is given in frames per second.
 * @param pcm A PCM handle.
 * @return The rate of the PCM.
 * @ingroup libtinyalsa-pcm
 */
unsigned int pcm_get_rate(const struct pcm *pcm)
{
    return pcm->config.rate;
}

/** Gets the format of the PCM.
 * @param pcm A PCM handle.
 * @return The format of the PCM.
 * @ingroup libtinyalsa-pcm
 */
enum pcm_format pcm_get_format(const struct pcm *pcm)
{
    return pcm->config.format;
}

/** Gets the file descriptor of the PCM.
 * Useful for extending functionality of the PCM when needed.
 * @param pcm A PCM handle.
 * @return The file descriptor of the PCM.
 * @ingroup libtinyalsa-pcm
 */
int pcm_get_file_descriptor(const struct pcm *pcm)
{
    return pcm->fd;
}

/** Gets the error message for the last error that occured.
 * If no error occured and this function is called, the results are undefined.
 * @param pcm A PCM handle.
 * @return The error message of the last error that occured.
 * @ingroup libtinyalsa-pcm
 */
const char* pcm_get_error(const struct pcm *pcm)
{
    return pcm->error;
}
 . . . . . .
/** Gets the subdevice on which the pcm has been opened.
 * @param pcm A PCM handle.
 * @return The subdevice on which the pcm has been opened */
unsigned int pcm_get_subdevice(const struct pcm *pcm)
{
    return pcm->subdevice;
}

/** Determines the number of bits occupied by a @ref pcm_format.
 * @param format A PCM format.
 * @return The number of bits associated with @p format
 * @ingroup libtinyalsa-pcm
 */
unsigned int pcm_format_to_bits(enum pcm_format format)
{
    switch (format) {
    case PCM_FORMAT_S32_LE:
    case PCM_FORMAT_S32_BE:
    case PCM_FORMAT_S24_LE:
    case PCM_FORMAT_S24_BE:
    case PCM_FORMAT_FLOAT_LE:
    case PCM_FORMAT_FLOAT_BE:
        return 32;
    case PCM_FORMAT_S24_3LE:
    case PCM_FORMAT_S24_3BE:
        return 24;
    default:
    case PCM_FORMAT_S16_LE:
    case PCM_FORMAT_S16_BE:
        return 16;
    case PCM_FORMAT_S8:
        return 8;
    };
}

/** Determines how many frames of a PCM can fit into a number of bytes.
 * @param pcm A PCM handle.
 * @param bytes The number of bytes.
 * @return The number of frames that may fit into @p bytes
 * @ingroup libtinyalsa-pcm
 */
unsigned int pcm_bytes_to_frames(const struct pcm *pcm, unsigned int bytes)
{
    return bytes / (pcm->config.channels *
        (pcm_format_to_bits(pcm->config.format) >> 3));
}

/** Determines how many bytes are occupied by a number of frames of a PCM.
 * @param pcm A PCM handle.
 * @param frames The number of frames of a PCM.
 * @return The bytes occupied by @p frames.
 * @ingroup libtinyalsa-pcm
 */
unsigned int pcm_frames_to_bytes(const struct pcm *pcm, unsigned int frames)
{
    return frames * pcm->config.channels *
        (pcm_format_to_bits(pcm->config.format) >> 3);
}
. . . . . .
/** Checks if a PCM file has been opened without error.
 * @param pcm A PCM handle.
 *  May be NULL.
 * @return If a PCM's file descriptor is not valid or the pointer is NULL, it returns zero.
 *  Otherwise, the function returns one.
 * @ingroup libtinyalsa-pcm
 */
int pcm_is_ready(const struct pcm *pcm)
{
    if (pcm != NULL) {
        return pcm->fd >= 0;
    }
    return 0;
}
. . . . . .
int pcm_get_poll_fd(struct pcm *pcm)
{
    return pcm->fd;
}
. . . . . .
/** Gets the delay of the PCM, in terms of frames.
 * @param pcm A PCM handle.
 * @returns On success, the delay of the PCM.
 *  On failure, a negative number.
 * @ingroup libtinyalsa-pcm
 */
long pcm_get_delay(struct pcm *pcm)
{
    if (ioctl(pcm->fd, SNDRV_PCM_IOCTL_DELAY, &pcm->pcm_delay) < 0)
        return -1;

    return pcm->pcm_delay;
}
```

这些接口的实现中，除 `pcm_get_delay()` 使用了 `ioctl()` 的 **SNDRV_PCM_IOCTL_DELAY** 命令外，其它的都没有与 Linux 内核的交互。

## Tinyalsa 数据传递接口的实现

tinyalsa 主要通过 `pcm_writei()` 和 `pcm_readi()` 接口实现音频数据的传输，这两个函数实现 (位于 `tinyalsa/src/pcm.c`) 如下：
```
static int pcm_sync_ptr(struct pcm *pcm, int flags)
{
    if (pcm->sync_ptr == NULL) {
        /* status and control are mmapped */
        if (flags & SNDRV_PCM_SYNC_PTR_HWSYNC) {
            if (pcm->ops->ioctl(pcm->data, SNDRV_PCM_IOCTL_HWSYNC) == -1) {
                return oops(pcm, errno, "failed to sync hardware pointer");
            }
        }
    } else {
        pcm->sync_ptr->flags = flags;
        if (pcm->ops->ioctl(pcm->data, SNDRV_PCM_IOCTL_SYNC_PTR,
                            pcm->sync_ptr) < 0) {
            return oops(pcm, errno, "failed to sync mmap ptr");
        }
    }

    return 0;
}

int pcm_state(struct pcm *pcm)
{
    // Update the state only. Do not sync HW sync.
    int err = pcm_sync_ptr(pcm, SNDRV_PCM_SYNC_PTR_APPL | SNDRV_PCM_SYNC_PTR_AVAIL_MIN);
    if (err < 0)
        return err;

    return pcm->mmap_status->state;
}
 . . . . . .
/** Prepares a PCM, if it has not been prepared already.
 * @param pcm A PCM handle.
 * @return On success, zero; on failure, a negative number.
 * @ingroup libtinyalsa-pcm
 */
int pcm_prepare(struct pcm *pcm)
{
    if (pcm->ops->ioctl(pcm->data, SNDRV_PCM_IOCTL_PREPARE) < 0)
        return oops(pcm, errno, "cannot prepare channel");

    /* get appl_ptr and avail_min from kernel */
    pcm_sync_ptr(pcm, SNDRV_PCM_SYNC_PTR_APPL|SNDRV_PCM_SYNC_PTR_AVAIL_MIN);

    return 0;
}
 . . . . . .
static int pcm_rw_transfer(struct pcm *pcm, void *data, unsigned int frames)
{
    int is_playback;

    struct snd_xferi transfer;
    int res;

    is_playback = !(pcm->flags & PCM_IN);

    transfer.buf = data;
    transfer.frames = frames;
    transfer.result = 0;

    res = pcm->ops->ioctl(pcm->data, is_playback
                          ? SNDRV_PCM_IOCTL_WRITEI_FRAMES
                          : SNDRV_PCM_IOCTL_READI_FRAMES, &transfer);

    return res == 0 ? (int) transfer.result : -1;
}

static int pcm_generic_transfer(struct pcm *pcm, void *data,
                                unsigned int frames)
{
    int res;

#if UINT_MAX > TINYALSA_FRAMES_MAX
    if (frames > TINYALSA_FRAMES_MAX)
        return -EINVAL;
#endif
    if (frames > INT_MAX)
        return -EINVAL;

    if (pcm_state(pcm) == PCM_STATE_SETUP && pcm_prepare(pcm) != 0) {
        return -1;
    }

again:

    if (pcm->flags & PCM_MMAP)
        res = pcm_mmap_transfer(pcm, data, frames);
    else
        res = pcm_rw_transfer(pcm, data, frames);

    if (res < 0) {
        switch (errno) {
        case EPIPE:
            pcm->xruns++;
            /* fallthrough */
        case ESTRPIPE:
            /*
             * Try to restart if we are allowed to do so.
             * Otherwise, return error.
             */
            if (pcm->flags & PCM_NORESTART || pcm_prepare(pcm))
                return -1;
            goto again;
        case EAGAIN:
            if (pcm->flags & PCM_NONBLOCK)
                return -1;
            /* fallthrough */
        default:
            return oops(pcm, errno, "cannot read/write stream data");
        }
    }

    return res;
}

/** Writes audio samples to PCM.
 * If the PCM has not been started, it is started in this function.
 * This function is only valid for PCMs opened with the @ref PCM_OUT flag.
 * @param pcm A PCM handle.
 * @param data The audio sample array
 * @param frame_count The number of frames occupied by the sample array.
 *  This value should not be greater than @ref TINYALSA_FRAMES_MAX
 *  or INT_MAX.
 * @return On success, this function returns the number of frames written; otherwise, a negative number.
 * @ingroup libtinyalsa-pcm
 */
int pcm_writei(struct pcm *pcm, const void *data, unsigned int frame_count)
{
    if (pcm->flags & PCM_IN)
        return -EINVAL;

    return pcm_generic_transfer(pcm, (void*) data, frame_count);
}

/** Reads audio samples from PCM.
 * If the PCM has not been started, it is started in this function.
 * This function is only valid for PCMs opened with the @ref PCM_IN flag.
 * @param pcm A PCM handle.
 * @param data The audio sample array
 * @param frame_count The number of frames occupied by the sample array.
 *  This value should not be greater than @ref TINYALSA_FRAMES_MAX
 *  or INT_MAX.
 * @return On success, this function returns the number of frames written; otherwise, a negative number.
 * @ingroup libtinyalsa-pcm
 */
int pcm_readi(struct pcm *pcm, void *data, unsigned int frame_count)
{
    if (!(pcm->flags & PCM_IN))
        return -EINVAL;

    return pcm_generic_transfer(pcm, data, frame_count);
}
```

`pcm_writei()` 和 `pcm_readi()` 函数都通过相同的 `pcm_generic_transfer()` 函数来实现。在 `pcm_generic_transfer()` 函数中，会借助于 `pcm_state()` 检查 PCM 的状态，如果状态为 **PCM_STATE_SETUP**，则会为音频流执行 Prepare。 之后，`pcm_generic_transfer()` 函数根据是否要以 mmap 的方式传输音频数据，分别调用 `pcm_mmap_transfer()` 和 `pcm_rw_transfer()` 函数来完成数据传输。

`pcm_state()` 先调用 `pcm_sync_ptr()` 与内核同步音频流状态信息，这最终通过 `ioctl()` 的 **SNDRV_PCM_IOCTL_HWSYNC** 或 **SNDRV_PCM_IOCTL_SYNC_PTR** 命令实现，前者用于支持从内核空间向用户空间内存映射 mmap `struct snd_pcm_mmap_status` 和 `struct snd_pcm_mmap_control` 的硬件架构，后者用于不支持的。`pcm_state()` 与内核同步音频流状态信息之后返回 `pcm->mmap_status->state`。Prepare 主要通过 `ioctl()` 的 **SNDRV_PCM_IOCTL_PREPARE** 命令完成，之后 `pcm_prepare()` 还会再次与内核同步音频流状态信息。在 Linux 内核中，`ioctl()` 的 **SNDRV_PCM_IOCTL_HWSYNC** 和 **SNDRV_PCM_IOCTL_SYNC_PTR** 命令实现 (位于 `sound/core/pcm_native.c`) 如下：
```
/* check and update PCM state; return 0 or a negative error
 * call this inside PCM lock
 */
static int do_pcm_hwsync(struct snd_pcm_substream *substream)
{
	switch (substream->runtime->status->state) {
	case SNDRV_PCM_STATE_DRAINING:
		if (substream->stream == SNDRV_PCM_STREAM_CAPTURE)
			return -EBADFD;
		fallthrough;
	case SNDRV_PCM_STATE_RUNNING:
		return snd_pcm_update_hw_ptr(substream);
	case SNDRV_PCM_STATE_PREPARED:
	case SNDRV_PCM_STATE_PAUSED:
		return 0;
	case SNDRV_PCM_STATE_SUSPENDED:
		return -ESTRPIPE;
	case SNDRV_PCM_STATE_XRUN:
		return -EPIPE;
	default:
		return -EBADFD;
	}
}
 . . . . . .
 static int snd_pcm_hwsync(struct snd_pcm_substream *substream)
{
	int err;

	snd_pcm_stream_lock_irq(substream);
	err = do_pcm_hwsync(substream);
	snd_pcm_stream_unlock_irq(substream);
	return err;
}
 . . . . . .
 static int snd_pcm_sync_ptr(struct snd_pcm_substream *substream,
			    struct snd_pcm_sync_ptr __user *_sync_ptr)
{
	struct snd_pcm_runtime *runtime = substream->runtime;
	struct snd_pcm_sync_ptr sync_ptr;
	volatile struct snd_pcm_mmap_status *status;
	volatile struct snd_pcm_mmap_control *control;
	int err;

	memset(&sync_ptr, 0, sizeof(sync_ptr));
	if (get_user(sync_ptr.flags, (unsigned __user *)&(_sync_ptr->flags)))
		return -EFAULT;
	if (copy_from_user(&sync_ptr.c.control, &(_sync_ptr->c.control), sizeof(struct snd_pcm_mmap_control)))
		return -EFAULT;	
	status = runtime->status;
	control = runtime->control;
	if (sync_ptr.flags & SNDRV_PCM_SYNC_PTR_HWSYNC) {
		err = snd_pcm_hwsync(substream);
		if (err < 0)
			return err;
	}
	snd_pcm_stream_lock_irq(substream);
	if (!(sync_ptr.flags & SNDRV_PCM_SYNC_PTR_APPL)) {
		err = pcm_lib_apply_appl_ptr(substream,
					     sync_ptr.c.control.appl_ptr);
		if (err < 0) {
			snd_pcm_stream_unlock_irq(substream);
			return err;
		}
	} else {
		sync_ptr.c.control.appl_ptr = control->appl_ptr;
	}
	if (!(sync_ptr.flags & SNDRV_PCM_SYNC_PTR_AVAIL_MIN))
		control->avail_min = sync_ptr.c.control.avail_min;
	else
		sync_ptr.c.control.avail_min = control->avail_min;
	sync_ptr.s.status.state = status->state;
	sync_ptr.s.status.hw_ptr = status->hw_ptr;
	sync_ptr.s.status.tstamp = status->tstamp;
	sync_ptr.s.status.suspended_state = status->suspended_state;
	sync_ptr.s.status.audio_tstamp = status->audio_tstamp;
	snd_pcm_stream_unlock_irq(substream);
	if (copy_to_user(_sync_ptr, &sync_ptr, sizeof(sync_ptr)))
		return -EFAULT;
	return 0;
}
 . . . . . .
static int snd_pcm_common_ioctl(struct file *file,
				 struct snd_pcm_substream *substream,
				 unsigned int cmd, void __user *arg)
{
	struct snd_pcm_file *pcm_file = file->private_data;
	int res;

	if (PCM_RUNTIME_CHECK(substream))
		return -ENXIO;

	res = snd_power_wait(substream->pcm->card, SNDRV_CTL_POWER_D0);
	if (res < 0)
		return res;

	switch (cmd) {
	case SNDRV_PCM_IOCTL_PVERSION:
		return put_user(SNDRV_PCM_VERSION, (int __user *)arg) ? -EFAULT : 0;
 . . . . . .
	case SNDRV_PCM_IOCTL_XRUN:
		return snd_pcm_xrun(substream);
	case SNDRV_PCM_IOCTL_HWSYNC:
		return snd_pcm_hwsync(substream);
	case SNDRV_PCM_IOCTL_DELAY:
	{
		snd_pcm_sframes_t delay;
		snd_pcm_sframes_t __user *res = arg;
		int err;

		err = snd_pcm_delay(substream, &delay);
		if (err)
			return err;
		if (put_user(delay, res))
			return -EFAULT;
		return 0;
	}
	case __SNDRV_PCM_IOCTL_SYNC_PTR32:
		return snd_pcm_ioctl_sync_ptr_compat(substream, arg);
	case __SNDRV_PCM_IOCTL_SYNC_PTR64:
		return snd_pcm_sync_ptr(substream, arg);
```

`pcm_generic_transfer()` 函数通过 `pcm_rw_transfer()` 函数以 Read/Write 的方式向内核传递音频数据时，简单地通过 `ioctl()` 的 **SNDRV_PCM_IOCTL_WRITEI_FRAMES** 或 **SNDRV_PCM_IOCTL_READI_FRAMES** 命令实现。在内核中，音频数据的详细流动过程可以参考 [Linux 内核音频数据传递主要流程 (上)](https://www.cnblogs.com/wolfcs/p/17650804.html) 和 [Linux 内核音频数据传递主要流程 (下)](https://www.cnblogs.com/wolfcs/p/17651440.html)。

以 mmap 的方式传输音频数据时，与以 Read/Write 的方式传输音频数据的详细过程不会有特别大的区别，但原本在内核空间执行的许多操作，会改为在用户空间执行。`pcm_mmap_transfer()` 函数的定义 (位于 `tinyalsa/src/pcm.c`) 如下：
```
/** Starts a PCM.
 * @param pcm A PCM handle.
 * @return On success, zero; on failure, a negative number.
 * @ingroup libtinyalsa-pcm
 */
int pcm_start(struct pcm *pcm)
{
    if (pcm_state(pcm) == PCM_STATE_SETUP && pcm_prepare(pcm) != 0) {
        return -1;
    }

    /* set appl_ptr and avail_min in kernel */
    if (pcm_sync_ptr(pcm, 0) < 0)
        return -1;

    if (pcm->mmap_status->state != PCM_STATE_RUNNING) {
        if (pcm->ops->ioctl(pcm->data, SNDRV_PCM_IOCTL_START) < 0)
            return oops(pcm, errno, "cannot start channel");
    }

    return 0;
}

/** Stops a PCM.
 * @param pcm A PCM handle.
 * @return On success, zero; on failure, a negative number.
 * @ingroup libtinyalsa-pcm
 */
int pcm_stop(struct pcm *pcm)
{
    if (pcm->ops->ioctl(pcm->data, SNDRV_PCM_IOCTL_DROP) < 0)
        return oops(pcm, errno, "cannot stop channel");

    return 0;
}

static inline long pcm_mmap_playback_avail(struct pcm *pcm)
{
    long avail = pcm->mmap_status->hw_ptr + (unsigned long) pcm->buffer_size -
            pcm->mmap_control->appl_ptr;

    if (avail < 0) {
        avail += pcm->boundary;
    } else if ((unsigned long) avail >= pcm->boundary) {
        avail -= pcm->boundary;
    }

    return avail;
}

static inline long pcm_mmap_capture_avail(struct pcm *pcm)
{
    long avail = pcm->mmap_status->hw_ptr - pcm->mmap_control->appl_ptr;
    if (avail < 0) {
        avail += pcm->boundary;
    }

    return avail;
}

int pcm_mmap_avail(struct pcm *pcm)
{
    pcm_sync_ptr(pcm, SNDRV_PCM_SYNC_PTR_HWSYNC);
    if (pcm->flags & PCM_IN) {
        return (int) pcm_mmap_capture_avail(pcm);
    } else {
        return (int) pcm_mmap_playback_avail(pcm);
    }
}

static void pcm_mmap_appl_forward(struct pcm *pcm, int frames)
{
    unsigned long appl_ptr = pcm->mmap_control->appl_ptr;
    appl_ptr += frames;

    /* check for boundary wrap */
    if (appl_ptr >= pcm->boundary) {
        appl_ptr -= pcm->boundary;
    }
    pcm->mmap_control->appl_ptr = appl_ptr;
}

int pcm_mmap_begin(struct pcm *pcm, void **areas, unsigned int *offset,
                   unsigned int *frames)
{
    unsigned int continuous, copy_frames, avail;

    /* return the mmap buffer */
    *areas = pcm->mmap_buffer;

    /* and the application offset in frames */
    *offset = pcm->mmap_control->appl_ptr % pcm->buffer_size;

    avail = pcm_mmap_avail(pcm);
    if (avail > pcm->buffer_size)
        avail = pcm->buffer_size;
    continuous = pcm->buffer_size - *offset;

    /* we can only copy frames if the are available and continuos */
    copy_frames = *frames;
    if (copy_frames > avail)
        copy_frames = avail;
    if (copy_frames > continuous)
        copy_frames = continuous;
    *frames = copy_frames;

    return 0;
}

static int pcm_areas_copy(struct pcm *pcm, unsigned int pcm_offset,
                          char *buf, unsigned int src_offset,
                          unsigned int frames)
{
    int size_bytes = pcm_frames_to_bytes(pcm, frames);
    int pcm_offset_bytes = pcm_frames_to_bytes(pcm, pcm_offset);
    int src_offset_bytes = pcm_frames_to_bytes(pcm, src_offset);

    /* interleaved only atm */
    if (pcm->flags & PCM_IN)
        memcpy(buf + src_offset_bytes,
               (char*)pcm->mmap_buffer + pcm_offset_bytes,
               size_bytes);
    else
        memcpy((char*)pcm->mmap_buffer + pcm_offset_bytes,
               buf + src_offset_bytes,
               size_bytes);
    return 0;
}

int pcm_mmap_commit(struct pcm *pcm, unsigned int offset, unsigned int frames)
{
    int ret;

    /* not used */
    (void) offset;

    /* update the application pointer in userspace and kernel */
    pcm_mmap_appl_forward(pcm, frames);
    ret = pcm_sync_ptr(pcm, 0);
    if (ret != 0){
        printf("%d\n", ret);
        return ret;
    }

    return frames;
}

static int pcm_mmap_transfer_areas(struct pcm *pcm, char *buf,
                                unsigned int offset, unsigned int size)
{
    void *pcm_areas;
    int commit;
    unsigned int pcm_offset, frames, count = 0;

    while (pcm_mmap_avail(pcm) && size) {
        frames = size;
        pcm_mmap_begin(pcm, &pcm_areas, &pcm_offset, &frames);
        pcm_areas_copy(pcm, pcm_offset, buf, offset, frames);
        commit = pcm_mmap_commit(pcm, pcm_offset, frames);
        if (commit < 0) {
            oops(pcm, commit, "failed to commit %d frames\n", frames);
            return commit;
        }

        offset += commit;
        count += commit;
        size -= commit;
    }
    return count;
}
 . . . . . .
/** Waits for frames to be available for read or write operations.
 * @param pcm A PCM handle.
 * @param timeout The maximum amount of time to wait for, in terms of milliseconds.
 * @returns If frames became available, one is returned.
 *  If a timeout occured, zero is returned.
 *  If an error occured, a negative number is returned.
 * @ingroup libtinyalsa-pcm
 */
int pcm_wait(struct pcm *pcm, int timeout)
{
    struct pollfd pfd;
    int err;

    pfd.fd = pcm->fd;
    pfd.events = POLLIN | POLLOUT | POLLERR | POLLNVAL;

    do {
        /* let's wait for avail or timeout */
        err = pcm->ops->poll(pcm->data, &pfd, 1, timeout);
        if (err < 0)
            return -errno;

        /* timeout ? */
        if (err == 0)
            return 0;

        /* have we been interrupted ? */
        if (errno == -EINTR)
            continue;

        /* check for any errors */
        if (pfd.revents & (POLLERR | POLLNVAL)) {
            switch (pcm_state(pcm)) {
            case PCM_STATE_XRUN:
                return -EPIPE;
            case PCM_STATE_SUSPENDED:
                return -ESTRPIPE;
            case PCM_STATE_DISCONNECTED:
                return -ENODEV;
            default:
                return -EIO;
            }
        }
    /* poll again if fd not ready for IO */
    } while (!(pfd.revents & (POLLIN | POLLOUT)));

    return 1;
}

/*
 * Transfer data to/from mmapped buffer. This imitates the
 * behavior of read/write system calls.
 *
 * However, this doesn't seems to offer any advantage over
 * the read/write syscalls. Should it be removed?
 */
static int pcm_mmap_transfer(struct pcm *pcm, void *buffer, unsigned int frames)
{
    int is_playback;

    int state;
    unsigned int avail;
    unsigned int user_offset = 0;

    int err;
    int transferred_frames;

    is_playback = !(pcm->flags & PCM_IN);

    if (frames == 0)
        return 0;

    /* update hardware pointer and get state */
    err = pcm_sync_ptr(pcm, SNDRV_PCM_SYNC_PTR_HWSYNC |
                            SNDRV_PCM_SYNC_PTR_APPL |
                            SNDRV_PCM_SYNC_PTR_AVAIL_MIN);
    if (err == -1)
        return -1;
    state = pcm->mmap_status->state;

    /*
     * If frames < start_threshold, wait indefinitely.
     * Another thread may start capture
     */
    if (!is_playback && state == PCM_STATE_PREPARED &&
            frames >= pcm->config.start_threshold) {
        if (pcm_start(pcm) < 0) {
            return -1;
        }
    }

    while (frames) {
        avail = pcm_mmap_avail(pcm);

        if (!avail) {
            if (pcm->flags & PCM_NONBLOCK) {
                errno = EAGAIN;
                break;
            }

            /* wait for interrupt */
            err = pcm_wait(pcm, -1);
            if (err < 0) {
                errno = -err;
                break;
            }
        }

        transferred_frames = pcm_mmap_transfer_areas(pcm, buffer, user_offset, frames);
        if (transferred_frames < 0) {
            break;
        }

        user_offset += transferred_frames;
        frames -= transferred_frames;

        /* start playback if written >= start_threshold */
        if (is_playback && state == PCM_STATE_PREPARED &&
                pcm->buffer_size - avail >= pcm->config.start_threshold) {
            if (pcm_start(pcm) < 0) {
                break;
            }
        }
    }

    return user_offset ? (int) user_offset : -1;
}
```

`pcm_mmap_transfer()` 函数的执行过程如下：
1. 调用 `pcm_sync_ptr()` 与内核同步音频流状态信息。
2. 如果音频流用于录制/采集音频数据，音频流的状态为 `PCM_STATE_PREPARED`，且请求的音频帧数超过阈值，则调用 `pcm_start(pcm)` 函数起动音频流。在 `pcm_start(pcm)` 函数中，会再次同步并检查音频流状态信息，如果状态为 **PCM_STATE_SETUP**，则会为音频流执行 Prepare，**这里与 `pcm_generic_transfer()` 的操作有重复？**之后，`pcm_start(pcm)` 函数通过 `ioctl()` 的 **SNDRV_PCM_IOCTL_START** 命令起动音频流，这最终将调用音频硬件驱动的 `trigger` 操作。
3. 通过一个循环传递请求的音频帧：
  (1). 等待 DMA 缓冲区中有足够的可用空间。DMA 缓冲区的可用空间通过 `hw_ptr` 和 `appl_ptr` 等状态计算，计算方法与 Linux 内核中的类似方法没有区别。关于这两个状态的含义，可以参考 [Linux 内核音频数据传递主要流程 (上)](https://www.cnblogs.com/wolfcs/p/17650804.html) 和 [Linux 内核音频数据传递主要流程 (下)](https://www.cnblogs.com/wolfcs/p/17651440.html)。
  (2). 调用 `pcm_mmap_transfer_areas()` 函数传递一块音频数据。
  (3). 音频流用于播放音频数据，音频流的状态为 `PCM_STATE_PREPARED`，且已经传递的音频帧数超过阈值，则调用 `pcm_start(pcm)` 函数起动音频流。

在 `pcm_mmap_transfer_areas()` 函数中，计算要写入或者读出 DMA 缓冲区中音频数据的具体位值，执行音频数据的内存拷贝，更新 `appl_ptr` 指针，并与内核同步音频流状态信息。

`pcm_mmap_transfer()` 函数的执行过程与内核中 `__snd_pcm_lib_xfer()` 函数的执行过程十分相似。

不难看出，Tinyalsa 音频流生命周期管理 API 中的许多的使用场景，不是在应用程序开发中，而是在支持实现以 mmap 方式向内核传递音频数据，这些 API 包括如下这些：
```
int pcm_mmap_begin(struct pcm *pcm, void **areas, unsigned int *offset, unsigned int *frames);

int pcm_mmap_commit(struct pcm *pcm, unsigned int offset, unsigned int frames);

int pcm_mmap_avail(struct pcm *pcm);
 . . . . . .
int pcm_prepare(struct pcm *pcm);

int pcm_start(struct pcm *pcm);

int pcm_stop(struct pcm *pcm);

int pcm_wait(struct pcm *pcm, int timeout);
```

Tinyalsa PCM API 实现大体如此。

Done.
